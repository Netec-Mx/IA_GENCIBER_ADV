# Desplegar LiteLLM + OPA y restringir acceso a modelos según rol

## Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 30 minutos                                 |
| **Complejidad**  | Alta                                       |
| **Nivel Bloom**  | Crear                                      |
| **Módulo**       | Capítulo 4 — AI Gateway y Control de Acceso |
| **Versiones**    | LiteLLM 1.40+, OPA 0.65+, Docker Compose 2.20+ |

---

## Descripción General

En este laboratorio desplegarás un stack completo de AI Gateway con control de acceso basado en roles (RBAC). Configurarás **LiteLLM** como proxy multi-proveedor con virtual keys y presupuestos diferenciados, e integrarás **Open Policy Agent (OPA)** como motor de políticas que valida cada solicitud antes de que sea enrutada al proveedor LLM. Al finalizar, habrás implementado el patrón *Policy Enforcement Point* (PEP) con sidecar, ejecutado una batería de 12 pruebas que cubre combinaciones de rol × modelo × horario, y revisado los audit logs generados por OPA.

---

## Objetivos de Aprendizaje

- [ ] Desplegar LiteLLM como AI Gateway con virtual keys, rate limiting y presupuestos por equipo usando Docker Compose.
- [ ] Escribir políticas Rego en OPA que implementen RBAC con restricciones temporales para modelos de producción.
- [ ] Integrar LiteLLM con OPA como Policy Enforcement Point mediante el patrón sidecar para validar cada request.
- [ ] Ejecutar y analizar una batería de tests que verifique el enforcement de políticas para diferentes roles y modelos.
- [ ] Revisar y auditar los logs de decisión de OPA para verificar trazabilidad de accesos permitidos y denegados.

---

## Prerrequisitos

### Conocimiento previo
- Comprensión básica de JWT y sus claims (`sub`, `role`, `iat`, `exp`).
- Familiaridad con RBAC y ABAC (cubiertos en el módulo teórico del Capítulo 4).
- Haber completado el Lab 03 (contexto del gateway y modelos de clasificación).
- Conocimiento básico de Docker Compose y redes de contenedores.

### Acceso y credenciales
- Al menos **una** de las siguientes opciones:
  - OpenAI API Key activa (`sk-...`).
  - AWS Bedrock configurado con `aws configure` y acceso a `us-east-1`.
  - *(Alternativa offline)*: el mock de API incluido en este lab (FastAPI, sin coste).
- Docker Engine 24.0+ y Docker Compose 2.20+ instalados y operativos.
- Python 3.11+ disponible en el host para ejecutar los scripts de test.

> **⚠️ Aviso de costes**: Si usas un proveedor real, este lab genera aproximadamente 20–40 llamadas cortas. El coste estimado es inferior a 0,10 USD con `gpt-3.5-turbo` / `gpt-4o-mini`. Configura un billing alert antes de comenzar.

> **🔒 Seguridad**: Ningún archivo de este lab debe contener credenciales reales. Todas las claves se leen desde variables de entorno o un archivo `.env` excluido por `.gitignore`.

---

## Entorno del Laboratorio

### Requisitos de hardware

| Recurso       | Mínimo            | Recomendado        |
|---------------|-------------------|--------------------|
| CPU           | 4 núcleos         | 8 núcleos          |
| RAM           | 8 GB disponibles  | 16 GB              |
| Almacenamiento| 5 GB libres       | 10 GB              |
| Red           | 10 Mbps           | 50 Mbps            |

### Software y versiones fijadas

| Componente     | Imagen / Versión                              | Puerto |
|----------------|-----------------------------------------------|--------|
| LiteLLM Proxy  | `ghcr.io/berriai/litellm:main-v1.40.10`       | 4000   |
| OPA            | `openpolicyagent/opa:0.65.0`                  | 8181   |
| Mock Identity  | `python:3.11-slim` (FastAPI, construido local)| 8080   |
| Mock LLM API   | `python:3.11-slim` (FastAPI, construido local)| 9090   |

### Estructura de directorios del lab

```
lab-04-00-01/
├── docker-compose.yml
├── .env.example
├── .gitignore
├── litellm/
│   └── config.yaml
├── opa/
│   ├── policies/
│   │   └── llm_access.rego
│   └── data/
│       └── model_tiers.json
├── mock-identity/
│   ├── Dockerfile
│   ├── main.py
│   └── requirements.txt
├── mock-llm/
│   ├── Dockerfile
│   ├── main.py
│   └── requirements.txt
└── tests/
    ├── requirements.txt
    └── test_policies.py
```

### Comandos de configuración inicial

```bash
# 1. Crear la estructura del laboratorio
mkdir -p lab-04-00-01/{litellm,opa/policies,opa/data,mock-identity,mock-llm,tests}
cd lab-04-00-01

# 2. Crear .gitignore para proteger credenciales
cat > .gitignore << 'EOF'
.env
*.env
*.pem
*.key
*.crt
__pycache__/
*.pyc
.pytest_cache/
EOF

# 3. Crear .env.example (plantilla sin valores reales)
cat > .env.example << 'EOF'
# Copiar a .env y rellenar con valores reales
OPENAI_API_KEY=sk-REPLACE_ME
LITELLM_MASTER_KEY=sk-lab-master-REPLACE_ME
JWT_SECRET=super-secret-jwt-key-REPLACE_ME
USE_MOCK_LLM=true
EOF

# 4. Copiar plantilla y configurar (editar con tus valores)
cp .env.example .env
echo "✅ Edita .env con tus credenciales reales antes de continuar"
```

---

## Instrucciones Paso a Paso

---

### Paso 1: Configurar el Mock de Proveedor LLM (alternativa offline)

**Objetivo**: Crear un servidor FastAPI que simule las respuestas de OpenAI para poder completar el lab sin incurrir en costes de API. Si tienes una API key real, puedes omitir la construcción del mock pero igualmente debes revisar este paso para entender la arquitectura.

#### Instrucciones

**1.1** Crear el Dockerfile del mock LLM:

```dockerfile
# mock-llm/Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
EXPOSE 9090
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "9090"]
```

**1.2** Crear las dependencias del mock LLM:

```text
# mock-llm/requirements.txt
fastapi==0.111.0
uvicorn==0.30.1
```

**1.3** Crear el servidor FastAPI que simula OpenAI:

```python
# mock-llm/main.py
"""
Mock LLM API - Simula respuestas de OpenAI para pruebas sin coste.
DISCLAIMER: Este servidor es únicamente para fines educativos y de prueba.
Las respuestas son ficticias y no provienen de ningún modelo real.
"""
import time
import uuid
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse

app = FastAPI(title="Mock LLM API", version="1.0.0")

MOCK_RESPONSES = {
    "gpt-3.5-turbo": "Respuesta simulada del modelo tier-1 (gpt-3.5-turbo). [MOCK]",
    "gpt-4o":        "Respuesta simulada del modelo tier-2 (gpt-4o). [MOCK]",
    "default":       "Respuesta simulada genérica. [MOCK]",
}

@app.post("/v1/chat/completions")
async def chat_completions(request: Request):
    body = await request.json()
    model = body.get("model", "default")
    content = MOCK_RESPONSES.get(model, MOCK_RESPONSES["default"])
    return JSONResponse({
        "id": f"chatcmpl-mock-{uuid.uuid4().hex[:8]}",
        "object": "chat.completion",
        "created": int(time.time()),
        "model": model,
        "choices": [{
            "index": 0,
            "message": {"role": "assistant", "content": content},
            "finish_reason": "stop"
        }],
        "usage": {"prompt_tokens": 10, "completion_tokens": 20, "total_tokens": 30}
    })

@app.get("/health")
async def health():
    return {"status": "ok", "service": "mock-llm"}
```

**Salida esperada**: Los archivos del mock LLM están creados correctamente.

**Verificación**:
```bash
ls -la mock-llm/
# Debe mostrar: Dockerfile, main.py, requirements.txt
```

---

### Paso 2: Configurar el Servicio de Mock de Identidad (JWT Generator)

**Objetivo**: Crear un microservicio que genere JWTs con claims de rol para simular un Identity Provider (IdP) real. Este servicio emitirá tokens para los roles `analyst`, `developer` y `admin`.

#### Instrucciones

**2.1** Crear el Dockerfile del mock de identidad:

```dockerfile
# mock-identity/Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**2.2** Crear las dependencias:

```text
# mock-identity/requirements.txt
fastapi==0.111.0
uvicorn==0.30.1
python-jose[cryptography]==3.3.0
```

**2.3** Crear el generador de JWTs:

```python
# mock-identity/main.py
"""
Mock Identity Provider - Genera JWTs con claims de rol para pruebas de RBAC.
DISCLAIMER: Solo para uso en laboratorio. No usar en producción.
"""
import os
import time
from fastapi import FastAPI, HTTPException
from jose import jwt
from pydantic import BaseModel

app = FastAPI(title="Mock Identity Provider", version="1.0.0")

JWT_SECRET = os.environ.get("JWT_SECRET", "super-secret-jwt-key-for-lab-only")
JWT_ALGORITHM = "HS256"
JWT_EXPIRY_SECONDS = 3600  # 1 hora

VALID_ROLES = {"analyst", "developer", "admin"}

ROLE_PERMISSIONS = {
    "analyst":   {"tier": "tier-1", "rpm_limit": 10,  "monthly_budget_usd": 50},
    "developer": {"tier": "tier-2", "rpm_limit": 50,  "monthly_budget_usd": 200},
    "admin":     {"tier": "tier-all","rpm_limit": -1,  "monthly_budget_usd": -1},
}

class TokenRequest(BaseModel):
    user_id: str
    role: str
    team: str = "default"

@app.post("/token")
async def generate_token(req: TokenRequest):
    if req.role not in VALID_ROLES:
        raise HTTPException(status_code=400, detail=f"Rol inválido. Válidos: {VALID_ROLES}")

    perms = ROLE_PERMISSIONS[req.role]
    now = int(time.time())
    payload = {
        "sub":          req.user_id,
        "role":         req.role,
        "team":         req.team,
        "tier":         perms["tier"],
        "rpm_limit":    perms["rpm_limit"],
        "monthly_budget": perms["monthly_budget_usd"],
        "iat":          now,
        "exp":          now + JWT_EXPIRY_SECONDS,
        "iss":          "mock-identity-provider-lab",
    }
    token = jwt.encode(payload, JWT_SECRET, algorithm=JWT_ALGORITHM)
    return {
        "access_token": token,
        "token_type": "bearer",
        "role": req.role,
        "expires_in": JWT_EXPIRY_SECONDS
    }

@app.get("/health")
async def health():
    return {"status": "ok", "service": "mock-identity"}
```

**Salida esperada**: Archivos del mock de identidad creados.

**Verificación**:
```bash
ls -la mock-identity/
# Debe mostrar: Dockerfile, main.py, requirements.txt
```

---

### Paso 3: Escribir las Políticas Rego en OPA

**Objetivo**: Implementar las reglas RBAC con restricciones temporales en el lenguaje Rego de OPA. Estas políticas serán el núcleo del control de acceso.

#### Instrucciones

**3.1** Crear el catálogo de modelos por tier:

```json
// opa/data/model_tiers.json
{
  "tiers": {
    "tier-1": {
      "models": ["gpt-3.5-turbo", "chat-standard"],
      "description": "Modelos económicos para tareas rutinarias",
      "production": false
    },
    "tier-2": {
      "models": ["gpt-4o", "gpt-4o-mini", "chat-advanced"],
      "description": "Modelos avanzados para tareas complejas",
      "production": true
    },
    "tier-all": {
      "models": ["gpt-3.5-turbo", "chat-standard", "gpt-4o", "gpt-4o-mini",
                 "chat-advanced", "claude-3-sonnet", "chat-premium"],
      "description": "Acceso completo — solo administradores",
      "production": true
    }
  },
  "business_hours": {
    "start_hour_utc": 8,
    "end_hour_utc": 20,
    "timezone_note": "UTC — ajustar según zona horaria del equipo"
  }
}
```

**3.2** Crear la política Rego principal:

```rego
# opa/policies/llm_access.rego
package llm.access

import future.keywords.if
import future.keywords.in

# ─────────────────────────────────────────────
# REGLA PRINCIPAL: allow
# Permite la solicitud si todas las sub-reglas pasan.
# ─────────────────────────────────────────────
default allow := false

allow if {
    role_exists
    model_allowed_for_role
    not production_model_outside_hours
}

# ─────────────────────────────────────────────
# SUB-REGLA 1: El rol del JWT es válido
# ─────────────────────────────────────────────
valid_roles := {"analyst", "developer", "admin"}

role_exists if {
    input.jwt_claims.role in valid_roles
}

# ─────────────────────────────────────────────
# SUB-REGLA 2: El modelo solicitado está permitido para el rol
# ─────────────────────────────────────────────

# Analyst: solo tier-1
model_allowed_for_role if {
    input.jwt_claims.role == "analyst"
    input.model in data.tiers["tier-1"].models
}

# Developer: tier-1 y tier-2
model_allowed_for_role if {
    input.jwt_claims.role == "developer"
    input.model in data.tiers["tier-1"].models
}

model_allowed_for_role if {
    input.jwt_claims.role == "developer"
    input.model in data.tiers["tier-2"].models
}

# Admin: todos los modelos (tier-all incluye todos)
model_allowed_for_role if {
    input.jwt_claims.role == "admin"
    input.model in data.tiers["tier-all"].models
}

# ─────────────────────────────────────────────
# SUB-REGLA 3: Restricción temporal para modelos de producción
# Los modelos de producción solo se pueden usar en horario laboral (UTC)
# ─────────────────────────────────────────────
is_production_model if {
    input.model in data.tiers["tier-2"].models
}

is_production_model if {
    input.model in data.tiers["tier-all"].models
    not input.model in data.tiers["tier-1"].models
    not input.model in data.tiers["tier-2"].models
}

within_business_hours if {
    hour := time.clock([input.request_time_ns, "UTC"])[0]
    hour >= data.business_hours.start_hour_utc
    hour < data.business_hours.end_hour_utc
}

production_model_outside_hours if {
    is_production_model
    not within_business_hours
}

# ─────────────────────────────────────────────
# METADATA: Razón de denegación (para audit logs)
# ─────────────────────────────────────────────
deny_reason := reason if {
    not role_exists
    reason := sprintf("Rol '%v' no reconocido o no presente en el JWT", [input.jwt_claims.role])
}

deny_reason := reason if {
    role_exists
    not model_allowed_for_role
    reason := sprintf("Rol '%v' no tiene permiso para usar el modelo '%v'",
                      [input.jwt_claims.role, input.model])
}

deny_reason := reason if {
    role_exists
    model_allowed_for_role
    production_model_outside_hours
    reason := sprintf("Modelo de producción '%v' solo disponible en horario laboral (08:00–20:00 UTC)",
                      [input.model])
}

deny_reason := "Solicitud denegada por política de acceso LLM" if {
    not allow
    not role_exists
    not model_allowed_for_role
}
```

**Salida esperada**: Archivos de política OPA creados sin errores de sintaxis.

**Verificación** (requiere OPA CLI instalado localmente, o se verifica en el Paso 6):
```bash
# Verificación de sintaxis (opcional si tienes OPA CLI local)
# opa check opa/policies/llm_access.rego
echo "Archivos de política creados:"
ls -la opa/policies/ opa/data/
```

---

### Paso 4: Configurar LiteLLM con Virtual Keys y Rate Limiting

**Objetivo**: Crear el archivo de configuración de LiteLLM con los tres roles, sus virtual keys, rate limits diferenciados y presupuestos mensuales. Configurar también el hook de autorización que consulta OPA.

#### Instrucciones

**4.1** Crear la configuración de LiteLLM:

```yaml
# litellm/config.yaml
# ─────────────────────────────────────────────────────────────────
# Configuración de LiteLLM AI Gateway — Lab 04-00-01
# NOTA: Las credenciales reales se inyectan via variables de entorno
# ─────────────────────────────────────────────────────────────────

model_list:
  # ── Tier 1: Modelos económicos ──────────────────────────────────
  - model_name: chat-standard
    litellm_params:
      model: gpt-3.5-turbo
      api_key: os.environ/OPENAI_API_KEY
    model_info:
      tier: tier-1
      id: chat-standard

  - model_name: gpt-3.5-turbo
    litellm_params:
      model: gpt-3.5-turbo
      api_key: os.environ/OPENAI_API_KEY
    model_info:
      tier: tier-1
      id: gpt-3.5-turbo

  # ── Tier 2: Modelos avanzados ────────────────────────────────────
  - model_name: chat-advanced
    litellm_params:
      model: gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY
    model_info:
      tier: tier-2
      id: chat-advanced

  - model_name: gpt-4o
    litellm_params:
      model: gpt-4o-mini      # alias a gpt-4o-mini para reducir costes en lab
      api_key: os.environ/OPENAI_API_KEY
    model_info:
      tier: tier-2
      id: gpt-4o

litellm_settings:
  request_timeout: 30
  max_tokens: 256
  temperature: 0.1
  drop_params: true           # ignora params no soportados por el backend
  # Hook de autorización: consulta OPA antes de enrutar
  # LiteLLM llamará a este endpoint con el request completo
  callbacks: []

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: null          # Sin base de datos para simplificar el lab
  store_model_in_db: false

# ── Virtual Keys por rol ─────────────────────────────────────────
# Estas keys se crearán programáticamente en el Paso 5 via Admin API.
# Se documentan aquí como referencia.
#
# analyst-key   → rate_limit: 10 RPM, budget: $50/mes,  modelos: tier-1
# developer-key → rate_limit: 50 RPM, budget: $200/mes, modelos: tier-1 + tier-2
# admin-key     → rate_limit: sin límite, budget: sin límite, modelos: todos
```

**4.2** Crear el script de inicialización de virtual keys:

```python
# litellm/create_virtual_keys.py
"""
Script para crear las virtual keys de LiteLLM via Admin API.
Ejecutar después de que el contenedor de LiteLLM esté en marcha.
"""
import os
import sys
import time
import requests

LITELLM_URL = os.environ.get("LITELLM_URL", "http://localhost:4000")
MASTER_KEY   = os.environ.get("LITELLM_MASTER_KEY", "")

if not MASTER_KEY:
    print("ERROR: LITELLM_MASTER_KEY no está definida en el entorno.")
    sys.exit(1)

HEADERS = {
    "Authorization": f"Bearer {MASTER_KEY}",
    "Content-Type": "application/json"
}

VIRTUAL_KEYS = [
    {
        "key_alias":     "analyst-key",
        "models":        ["chat-standard", "gpt-3.5-turbo"],
        "tpm_limit":     5000,
        "rpm_limit":     10,
        "max_budget":    50.0,
        "budget_duration": "1mo",
        "metadata":      {"role": "analyst", "team": "data-analytics", "tier": "tier-1"}
    },
    {
        "key_alias":     "developer-key",
        "models":        ["chat-standard", "gpt-3.5-turbo", "chat-advanced", "gpt-4o"],
        "tpm_limit":     20000,
        "rpm_limit":     50,
        "max_budget":    200.0,
        "budget_duration": "1mo",
        "metadata":      {"role": "developer", "team": "engineering", "tier": "tier-2"}
    },
    {
        "key_alias":     "admin-key",
        "models":        [],        # lista vacía = acceso a todos los modelos
        "tpm_limit":     None,
        "rpm_limit":     None,
        "max_budget":    None,
        "budget_duration": "1mo",
        "metadata":      {"role": "admin", "team": "platform", "tier": "tier-all"}
    },
]

def wait_for_litellm(max_retries: int = 30, delay: float = 2.0):
    """Espera hasta que LiteLLM responda en /health."""
    print(f"Esperando que LiteLLM esté disponible en {LITELLM_URL}...")
    for i in range(max_retries):
        try:
            r = requests.get(f"{LITELLM_URL}/health", timeout=5)
            if r.status_code == 200:
                print("✅ LiteLLM disponible.")
                return True
        except requests.exceptions.ConnectionError:
            pass
        print(f"  Intento {i+1}/{max_retries}... reintentando en {delay}s")
        time.sleep(delay)
    print("❌ LiteLLM no respondió a tiempo.")
    return False

def create_key(key_config: dict) -> dict:
    """Crea una virtual key via la Admin API de LiteLLM."""
    payload = {k: v for k, v in key_config.items() if v is not None}
    r = requests.post(f"{LITELLM_URL}/key/generate", json=payload, headers=HEADERS, timeout=10)
    r.raise_for_status()
    return r.json()

if __name__ == "__main__":
    if not wait_for_litellm():
        sys.exit(1)

    created_keys = {}
    for key_cfg in VIRTUAL_KEYS:
        alias = key_cfg["key_alias"]
        try:
            result = create_key(key_cfg)
            api_key = result.get("key", "N/A")
            created_keys[alias] = api_key
            print(f"✅ {alias}: {api_key[:20]}...")
        except Exception as e:
            print(f"❌ Error creando {alias}: {e}")

    print("\n── Resumen de Virtual Keys ──────────────────────────")
    for alias, key in created_keys.items():
        print(f"  {alias:15s} → {key}")
    print("\nGuarda estas keys en tu .env para los tests.")
```

**Salida esperada**: Archivos de configuración de LiteLLM creados.

**Verificación**:
```bash
ls -la litellm/
# Debe mostrar: config.yaml, create_virtual_keys.py
```

---

### Paso 5: Crear el Docker Compose Stack

**Objetivo**: Ensamblar todos los servicios en un único `docker-compose.yml` con networking correcto, health checks y orden de arranque.

#### Instrucciones

**5.1** Crear el archivo Docker Compose:

```yaml
# docker-compose.yml
version: "3.9"

networks:
  lab-net:
    driver: bridge

services:

  # ── Mock LLM Provider (alternativa offline) ──────────────────────
  mock-llm:
    build:
      context: ./mock-llm
      dockerfile: Dockerfile
    container_name: lab-mock-llm
    networks: [lab-net]
    ports:
      - "9090:9090"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9090/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ── Mock Identity Provider ────────────────────────────────────────
  mock-identity:
    build:
      context: ./mock-identity
      dockerfile: Dockerfile
    container_name: lab-mock-identity
    networks: [lab-net]
    ports:
      - "8080:8080"
    environment:
      - JWT_SECRET=${JWT_SECRET:-super-secret-jwt-key-for-lab-only}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ── Open Policy Agent ─────────────────────────────────────────────
  opa:
    image: openpolicyagent/opa:0.65.0
    container_name: lab-opa
    networks: [lab-net]
    ports:
      - "8181:8181"
    volumes:
      - ./opa/policies:/policies:ro
      - ./opa/data:/data:ro
    command:
      - "run"
      - "--server"
      - "--log-level=info"
      - "--log-format=json"
      - "--set=decision_logs.console=true"
      - "/policies"
      - "/data/model_tiers.json"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8181/health"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  # ── LiteLLM AI Gateway ────────────────────────────────────────────
  litellm:
    image: ghcr.io/berriai/litellm:main-v1.40.10
    container_name: lab-litellm
    networks: [lab-net]
    ports:
      - "4000:4000"
    volumes:
      - ./litellm/config.yaml:/app/config.yaml:ro
    environment:
      # Credenciales del proveedor LLM
      - OPENAI_API_KEY=${OPENAI_API_KEY:-dummy-key-for-mock}
      - LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY:-sk-lab-master-key}
      # Si USE_MOCK_LLM=true, LiteLLM apuntará al mock en lugar de OpenAI
      - OPENAI_API_BASE=${OPENAI_API_BASE:-}
    command:
      - "--config"
      - "/app/config.yaml"
      - "--host"
      - "0.0.0.0"
      - "--port"
      - "4000"
      - "--detailed_debug"
    depends_on:
      opa:
        condition: service_healthy
      mock-llm:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 30s
    restart: unless-stopped
```

**5.2** Si vas a usar el mock LLM, añadir `OPENAI_API_BASE` al `.env`:

```bash
# Añadir al archivo .env (si usas mock LLM)
echo "OPENAI_API_BASE=http://mock-llm:9090/v1" >> .env
echo "USE_MOCK_LLM=true" >> .env
```

**5.3** Levantar el stack:

```bash
# Construir imágenes locales y levantar todos los servicios
docker compose up --build -d

# Verificar que todos los contenedores están en estado healthy
docker compose ps
```

**Salida esperada**:
```
NAME                  IMAGE                                    STATUS
lab-mock-llm          lab-04-00-01-mock-llm                   Up (healthy)
lab-mock-identity     lab-04-00-01-mock-identity              Up (healthy)
lab-opa               openpolicyagent/opa:0.65.0              Up (healthy)
lab-litellm           ghcr.io/berriai/litellm:main-v1.40.10  Up (healthy)
```

**Verificación**:
```bash
# Health checks individuales
curl -s http://localhost:8080/health | python3 -m json.tool
curl -s http://localhost:8181/health | python3 -m json.tool
curl -s http://localhost:4000/health | python3 -m json.tool

# Verificar que OPA cargó las políticas
curl -s http://localhost:8181/v1/policies | python3 -m json.tool | grep "llm.access"
```

---

### Paso 6: Crear las Virtual Keys y Probar OPA Directamente

**Objetivo**: Inicializar las virtual keys en LiteLLM y verificar manualmente que OPA evalúa correctamente las políticas Rego antes de ejecutar la batería de tests completa.

#### Instrucciones

**6.1** Instalar dependencias Python para el lab:

```bash
# tests/requirements.txt
cat > tests/requirements.txt << 'EOF'
requests==2.32.3
python-jose[cryptography]==3.3.0
pytest==8.2.2
pytest-html==4.1.1
tabulate==0.9.0
EOF

pip install -r tests/requirements.txt
```

**6.2** Crear las virtual keys ejecutando el script de inicialización:

```bash
# Cargar variables de entorno y ejecutar el script
export $(grep -v '^#' .env | xargs)
python3 litellm/create_virtual_keys.py
```

**6.3** Probar OPA directamente con casos manuales:

```bash
# ── TEST MANUAL 1: Analyst + tier-1 → DEBE PERMITIR ──────────────
curl -s -X POST http://localhost:8181/v1/data/llm/access \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "jwt_claims": {"role": "analyst", "sub": "user-001", "team": "data-analytics"},
      "model": "gpt-3.5-turbo",
      "request_time_ns": '$(date -u +%s)'000000000
    }
  }' | python3 -m json.tool
# Resultado esperado: {"result": {"allow": true, ...}}

# ── TEST MANUAL 2: Analyst + tier-2 → DEBE DENEGAR ───────────────
curl -s -X POST http://localhost:8181/v1/data/llm/access \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "jwt_claims": {"role": "analyst", "sub": "user-001", "team": "data-analytics"},
      "model": "gpt-4o",
      "request_time_ns": '$(date -u +%s)'000000000
    }
  }' | python3 -m json.tool
# Resultado esperado: {"result": {"allow": false, "deny_reason": "Rol 'analyst' no tiene permiso..."}}

# ── TEST MANUAL 3: Developer + tier-2 → DEBE PERMITIR ────────────
curl -s -X POST http://localhost:8181/v1/data/llm/access \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "jwt_claims": {"role": "developer", "sub": "user-002", "team": "engineering"},
      "model": "gpt-4o",
      "request_time_ns": '$(date -u +%s)'000000000
    }
  }' | python3 -m json.tool
```

**Salida esperada** para el Test 1:
```json
{
  "result": {
    "allow": true,
    "deny_reason": "",
    "model_allowed_for_role": true,
    "role_exists": true
  }
}
```

**Verificación**:
```bash
# Listar decisiones en los logs de OPA
docker logs lab-opa 2>&1 | grep "decision_id" | tail -5
```

---

### Paso 7: Ejecutar la Batería Completa de 12 Tests

**Objetivo**: Crear y ejecutar el script de pruebas automatizado que cubre las 12 combinaciones de rol × modelo × horario, verificando que OPA permite y deniega correctamente cada caso.

#### Instrucciones

**7.1** Crear el script de tests:

```python
# tests/test_policies.py
"""
Batería de 12 tests para verificar el enforcement de políticas RBAC en OPA.
Cubre combinaciones: roles × modelos × restricción temporal.

DISCLAIMER: Los usuarios y datos usados son completamente ficticios.
"""
import os
import time
import requests
from jose import jwt
from tabulate import tabulate
import pytest

OPA_URL       = os.environ.get("OPA_URL", "http://localhost:8181")
IDENTITY_URL  = os.environ.get("IDENTITY_URL", "http://localhost:8080")
JWT_SECRET    = os.environ.get("JWT_SECRET", "super-secret-jwt-key-for-lab-only")
OPA_POLICY    = f"{OPA_URL}/v1/data/llm/access"

# ── Definición de los 12 casos de prueba ─────────────────────────────────────
TEST_CASES = [
    # ID  | Rol        | Modelo           | Hora UTC | Esperado | Descripción
    (1,  "analyst",   "gpt-3.5-turbo",  10, True,  "Analyst + tier-1 en horario → PERMITIR"),
    (2,  "analyst",   "chat-standard",  14, True,  "Analyst + alias tier-1 en horario → PERMITIR"),
    (3,  "analyst",   "gpt-4o",         10, False, "Analyst + tier-2 → DENEGAR (sin permiso de tier)"),
    (4,  "analyst",   "chat-advanced",  10, False, "Analyst + alias tier-2 → DENEGAR"),
    (5,  "developer", "gpt-3.5-turbo",  11, True,  "Developer + tier-1 en horario → PERMITIR"),
    (6,  "developer", "gpt-4o",         15, True,  "Developer + tier-2 en horario → PERMITIR"),
    (7,  "developer", "chat-advanced",  9,  True,  "Developer + alias tier-2 en horario → PERMITIR"),
    (8,  "developer", "gpt-4o",         3,  False, "Developer + tier-2 FUERA de horario → DENEGAR"),
    (9,  "admin",     "gpt-3.5-turbo",  12, True,  "Admin + tier-1 → PERMITIR"),
    (10, "admin",     "gpt-4o",         16, True,  "Admin + tier-2 en horario → PERMITIR"),
    (11, "admin",     "chat-premium",   22, False, "Admin + modelo prod FUERA de horario → DENEGAR"),
    (12, "unknown",   "gpt-3.5-turbo",  10, False, "Rol desconocido → DENEGAR (rol inválido)"),
]

def build_opa_input(role: str, model: str, hour_utc: int) -> dict:
    """Construye el payload de input para OPA con timestamp forzado a la hora indicada."""
    now = int(time.time())
    # Calcular el timestamp ajustado a la hora UTC deseada del día actual
    current_hour = int(time.strftime("%H", time.gmtime(now)))
    offset_seconds = (hour_utc - current_hour) * 3600
    forced_time_ns = (now + offset_seconds) * 1_000_000_000

    return {
        "input": {
            "jwt_claims": {
                "role": role,
                "sub": f"test-user-{role}",
                "team": f"team-{role}",
                "iat": now,
                "exp": now + 3600
            },
            "model": model,
            "request_time_ns": forced_time_ns
        }
    }

def evaluate_policy(role: str, model: str, hour_utc: int) -> dict:
    """Llama a OPA y retorna el resultado de la evaluación."""
    payload = build_opa_input(role, model, hour_utc)
    resp = requests.post(OPA_POLICY, json=payload, timeout=10)
    resp.raise_for_status()
    return resp.json().get("result", {})

def run_all_tests() -> list:
    """Ejecuta todos los casos de prueba y retorna resultados."""
    results = []
    passed = 0
    failed = 0

    for test_id, role, model, hour, expected, description in TEST_CASES:
        try:
            result = evaluate_policy(role, model, hour)
            actual_allow = result.get("allow", False)
            deny_reason  = result.get("deny_reason", "")
            status = "✅ PASS" if actual_allow == expected else "❌ FAIL"
            if actual_allow == expected:
                passed += 1
            else:
                failed += 1
        except Exception as e:
            actual_allow = None
            deny_reason  = str(e)
            status = "💥 ERROR"
            failed += 1

        results.append([
            test_id, role, model, f"{hour:02d}:00 UTC",
            "ALLOW" if expected else "DENY",
            "ALLOW" if actual_allow else "DENY",
            status,
            deny_reason[:60] if deny_reason else ""
        ])

    return results, passed, failed

# ── Pytest parametrized tests ─────────────────────────────────────────────────
@pytest.mark.parametrize(
    "test_id,role,model,hour,expected,description",
    TEST_CASES
)
def test_opa_policy(test_id, role, model, hour, expected, description):
    """Test parametrizado para pytest — ejecuta cada caso individualmente."""
    result = evaluate_policy(role, model, hour)
    actual = result.get("allow", False)
    deny_reason = result.get("deny_reason", "")

    assert actual == expected, (
        f"\nTest #{test_id}: {description}\n"
        f"  Rol: {role}, Modelo: {model}, Hora: {hour:02d}:00 UTC\n"
        f"  Esperado: {'ALLOW' if expected else 'DENY'}\n"
        f"  Obtenido: {'ALLOW' if actual else 'DENY'}\n"
        f"  Razón OPA: {deny_reason}"
    )

# ── Ejecución directa con tabla de resultados ─────────────────────────────────
if __name__ == "__main__":
    print("\n" + "="*80)
    print("  BATERÍA DE TESTS — LiteLLM + OPA RBAC Policy Enforcement")
    print("="*80 + "\n")

    results, passed, failed = run_all_tests()

    headers = ["#", "Rol", "Modelo", "Hora", "Esperado", "Obtenido", "Estado", "Razón OPA"]
    print(tabulate(results, headers=headers, tablefmt="grid"))

    print(f"\n{'='*80}")
    print(f"  Resultados: {passed} PASS | {failed} FAIL | {len(TEST_CASES)} TOTAL")
    print(f"{'='*80}\n")

    if failed > 0:
        print("⚠️  Algunos tests fallaron. Revisa las políticas Rego y los datos en opa/data/.")
        exit(1)
    else:
        print("🎉 Todos los tests pasaron correctamente.")
```

**7.2** Ejecutar la batería de tests:

```bash
# Opción A: Ejecución directa con tabla de resultados
cd tests
export $(grep -v '^#' ../.env | xargs)
python3 test_policies.py
```

```bash
# Opción B: Ejecución con pytest y reporte HTML
cd tests
export $(grep -v '^#' ../.env | xargs)
pytest test_policies.py -v --html=report.html --self-contained-html
echo "Reporte generado en tests/report.html"
```

**Salida esperada**:
```
================================================================================
  BATERÍA DE TESTS — LiteLLM + OPA RBAC Policy Enforcement
================================================================================

+----+------------+----------------+----------+-----------+-----------+--------+-------------------------------+
| #  | Rol        | Modelo         | Hora     | Esperado  | Obtenido  | Estado | Razón OPA                     |
+----+------------+----------------+----------+-----------+-----------+--------+-------------------------------+
|  1 | analyst    | gpt-3.5-turbo  | 10:00 UTC| ALLOW     | ALLOW     | ✅ PASS|                               |
|  2 | analyst    | chat-standard  | 14:00 UTC| ALLOW     | ALLOW     | ✅ PASS|                               |
|  3 | analyst    | gpt-4o         | 10:00 UTC| DENY      | DENY      | ✅ PASS| Rol 'analyst' no tiene perm...|
|  4 | analyst    | chat-advanced  | 10:00 UTC| DENY      | DENY      | ✅ PASS| Rol 'analyst' no tiene perm...|
|  5 | developer  | gpt-3.5-turbo  | 11:00 UTC| ALLOW     | ALLOW     | ✅ PASS|                               |
|  6 | developer  | gpt-4o         | 15:00 UTC| ALLOW     | ALLOW     | ✅ PASS|                               |
|  7 | developer  | chat-advanced  | 09:00 UTC| ALLOW     | ALLOW     | ✅ PASS|                               |
|  8 | developer  | gpt-4o         | 03:00 UTC| DENY      | DENY      | ✅ PASS| Modelo de producción 'gpt-4o'.|
|  9 | admin      | gpt-3.5-turbo  | 12:00 UTC| ALLOW     | ALLOW     | ✅ PASS|                               |
| 10 | admin      | gpt-4o         | 16:00 UTC| ALLOW     | ALLOW     | ✅ PASS|                               |
| 11 | admin      | chat-premium   | 22:00 UTC| DENY      | DENY      | ✅ PASS| Modelo de producción 'chat-p..|
| 12 | unknown    | gpt-3.5-turbo  | 10:00 UTC| DENY      | DENY      | ✅ PASS| Rol 'unknown' no reconocido...|
+----+------------+----------------+----------+-----------+-----------+--------+-------------------------------+

================================================================================
  Resultados: 12 PASS | 0 FAIL | 12 TOTAL
================================================================================
🎉 Todos los tests pasaron correctamente.
```

**Verificación**:
```bash
# Confirmar que pytest también reporta 12 passed
pytest tests/test_policies.py -v --tb=short 2>&1 | tail -5
```

---

## Validación y Pruebas

### Validación de la Integración Completa

Después de que todos los tests pasen, verifica la integración end-to-end entre LiteLLM y OPA:

```bash
# ── 1. Verificar que LiteLLM expone los modelos configurados ─────────────────
curl -s http://localhost:4000/v1/models \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" | python3 -m json.tool | grep '"id"'

# ── 2. Verificar virtual keys creadas ────────────────────────────────────────
curl -s http://localhost:4000/key/list \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" | python3 -m json.tool

# ── 3. Probar una llamada real via LiteLLM con la analyst-key ────────────────
# Sustituir ANALYST_KEY con el valor obtenido en el Paso 6.2
curl -s http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer ${ANALYST_KEY:-sk-analyst-key}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "chat-standard",
    "messages": [{"role": "user", "content": "Hola, ¿qué modelos puedo usar?"}],
    "max_tokens": 50
  }' | python3 -m json.tool

# ── 4. Verificar que OPA tiene decisiones registradas ────────────────────────
docker logs lab-opa 2>&1 | grep '"msg":"Decision Log"' | wc -l
echo "decisiones registradas en OPA"
```

### Revisión de Audit Logs de OPA

```bash
# Ver las últimas 20 decisiones de OPA en formato legible
docker logs lab-opa 2>&1 \
  | grep '"msg":"Decision Log"' \
  | tail -20 \
  | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        log = json.loads(line)
        result = log.get('result', {})
        inp = log.get('input', {})
        print(f\"[{log.get('timestamp','?')[:19]}] \
role={inp.get('jwt_claims',{}).get('role','?'):10s} \
model={inp.get('model','?'):20s} \
allow={result.get('allow','?')} \
reason={result.get('deny_reason','')[:50]}\")
    except:
        pass
"
```

**Salida esperada** (extracto):
```
[2024-01-15T10:23:45] role=analyst     model=gpt-3.5-turbo      allow=True  reason=
[2024-01-15T10:23:46] role=analyst     model=gpt-4o             allow=False reason=Rol 'analyst' no tiene permiso...
[2024-01-15T10:23:47] role=developer   model=gpt-4o             allow=True  reason=
[2024-01-15T10:23:48] role=unknown     model=gpt-3.5-turbo      allow=False reason=Rol 'unknown' no reconocido...
```

### Checklist de Validación Final

```bash
# Script de validación rápida
echo "── Checklist de Validación Lab 04-00-01 ──────────────────────"
echo -n "[1] Stack Docker en estado healthy: "
docker compose ps --format "{{.Status}}" | grep -c "healthy" | xargs -I{} echo "{}/4 servicios"

echo -n "[2] OPA responde en /health: "
curl -s http://localhost:8181/health | grep -q "ok" && echo "✅" || echo "❌"

echo -n "[3] LiteLLM responde en /health: "
curl -s http://localhost:4000/health | grep -q "healthy\|ok" && echo "✅" || echo "❌"

echo -n "[4] Política Rego cargada en OPA: "
curl -s http://localhost:8181/v1/policies | grep -q "llm.access" && echo "✅" || echo "❌"

echo -n "[5] Tests de políticas: "
cd tests && python3 test_policies.py 2>&1 | grep -q "12 PASS" && echo "✅ 12/12" || echo "❌ Revisar fallos"

echo "──────────────────────────────────────────────────────────────"
```

---

## Resolución de Problemas

### Problema 1: OPA devuelve `undefined` en lugar de `allow: false`

**Síntoma**: Las llamadas a `POST /v1/data/llm/access` retornan `{"result": {}}` (objeto vacío) en lugar de `{"result": {"allow": false}}`.

**Causa**: OPA no encontró la política `llm.access` o los datos de `model_tiers.json` no se cargaron correctamente. Esto ocurre cuando el volumen de políticas no está montado correctamente, o cuando el archivo `.rego` tiene un error de sintaxis que impide la carga.

**Solución**:
```bash
# 1. Verificar que OPA cargó las políticas
curl -s http://localhost:8181/v1/policies | python3 -m json.tool | grep "llm"

# 2. Si está vacío, revisar los logs de OPA para errores de sintaxis
docker logs lab-opa 2>&1 | grep -i "error\|failed" | head -20

# 3. Verificar que los volúmenes están montados correctamente
docker inspect lab-opa | python3 -m json.tool | grep -A5 "Mounts"

# 4. Recargar las políticas manualmente via API
curl -s -X PUT http://localhost:8181/v1/policies/llm_access \
  -H "Content-Type: text/plain" \
  --data-binary @opa/policies/llm_access.rego

# 5. Cargar los datos manualmente
curl -s -X PUT http://localhost:8181/v1/data/tiers \
  -H "Content-Type: application/json" \
  -d @opa/data/model_tiers.json

# 6. Verificar con una consulta de prueba
curl -s -X POST http://localhost:8181/v1/data/llm/access \
  -H "Content-Type: application/json" \
  -d '{"input": {"jwt_claims": {"role": "analyst"}, "model": "gpt-3.5-turbo", "request_time_ns": '$(date -u +%s)'000000000}}'
```

---

### Problema 2: LiteLLM falla al arrancar con error de configuración o credenciales

**Síntoma**: El contenedor `lab-litellm` aparece en estado `Restarting` o `Exit 1`. Los logs muestran errores como `ValueError: No models found in config` o `AuthenticationError: Incorrect API key`.

**Causa**: Hay dos causas frecuentes: (a) el archivo `config.yaml` no está siendo montado correctamente, o (b) las variables de entorno `OPENAI_API_KEY` y `LITELLM_MASTER_KEY` no están definidas en `.env` o tienen valores vacíos. LiteLLM falla en el arranque si no puede resolver las credenciales de al menos un modelo.

**Solución**:
```bash
# 1. Revisar logs detallados de LiteLLM
docker logs lab-litellm 2>&1 | tail -30

# 2. Verificar que .env tiene las variables necesarias
grep -E "OPENAI_API_KEY|LITELLM_MASTER_KEY" .env

# 3. Si usas el mock LLM, asegurarte de que OPENAI_API_BASE está configurado
grep "OPENAI_API_BASE" .env
# Debe mostrar: OPENAI_API_BASE=http://mock-llm:9090/v1

# 4. Verificar que el config.yaml está siendo montado
docker inspect lab-litellm | python3 -m json.tool | grep -A3 "config.yaml"

# 5. Probar la configuración antes de levantar el contenedor
docker run --rm \
  -v $(pwd)/litellm/config.yaml:/app/config.yaml:ro \
  -e OPENAI_API_KEY=dummy-test \
  -e LITELLM_MASTER_KEY=sk-test \
  ghcr.io/berriai/litellm:main-v1.40.10 \
  --config /app/config.yaml --test

# 6. Reconstruir y reiniciar el stack
docker compose down
docker compose up --build -d litellm
docker compose logs -f litellm
```

---

## Limpieza

```bash
# ── 1. Detener y eliminar todos los contenedores del lab ─────────────────────
docker compose down --volumes --remove-orphans

# ── 2. Eliminar imágenes construidas localmente ───────────────────────────────
docker rmi lab-04-00-01-mock-llm lab-04-00-01-mock-identity 2>/dev/null || true

# ── 3. Verificar que no quedan contenedores del lab ───────────────────────────
docker ps -a | grep "lab-" | awk '{print $1}' | xargs docker rm -f 2>/dev/null || true

# ── 4. Limpiar el archivo .env (NUNCA subir al repositorio) ───────────────────
# Opcional: eliminar el .env local si el lab ha terminado
# rm -f .env
echo "⚠️  Recuerda: el archivo .env con credenciales reales NO debe subirse a git."
echo "   Verifica que .gitignore incluye .env antes de cualquier commit."

# ── 5. Verificar limpieza completa ────────────────────────────────────────────
docker compose ps
echo "Stack eliminado correctamente."
```

---

## Resumen

En este laboratorio has construido y validado un pipeline de control de acceso para un AI Gateway de producción:

| Componente         | Lo que implementaste                                                                 |
|--------------------|--------------------------------------------------------------------------------------|
| **LiteLLM**        | Gateway multi-proveedor con 4 modelos en 2 tiers, 3 virtual keys y rate limits       |
| **OPA + Rego**     | Políticas RBAC con restricción temporal para modelos de producción                    |
| **Mock Identity**  | Generador de JWTs con claims de rol para simular un IdP real                         |
| **Batería de tests**| 12 casos que cubren todas las combinaciones rol × modelo × horario                  |
| **Audit Logs**     | Trazabilidad completa de decisiones de acceso en OPA con razón de denegación         |

### Conceptos Clave Reforzados

- **Principio de mínimo privilegio**: cada rol solo accede a los modelos que necesita (`analyst` → tier-1, `developer` → tier-1+2, `admin` → todo).
- **Separación de responsabilidades**: LiteLLM gestiona el routing y los presupuestos; OPA gestiona las decisiones de autorización de forma desacoplada.
- **Políticas como código**: las reglas Rego son versionables, testeables y auditables, a diferencia de la lógica de autorización embebida en la aplicación.
- **Restricciones contextuales (ABAC)**: la política temporal demuestra cómo OPA puede combinar atributos del sujeto (rol), recurso (modelo) y entorno (hora UTC) en una sola decisión.

### Próximos Pasos

- **Lab 04-00-02**: Integrar LiteLLM con Langfuse para observabilidad completa de prompts y decisiones de acceso.
- **Lab 05**: Implementar guardrails multicapa con LLM Guard y Presidio sobre el mismo gateway.
- **Exploración adicional**: Revisar la [documentación de OPA Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) para políticas más complejas (ABAC con atributos de datos, políticas jerárquicas).

### Referencias

- [Documentación oficial de LiteLLM — Virtual Keys](https://docs.litellm.ai/docs/proxy/virtual_keys)
- [OPA — Rego Language Reference](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [OWASP LLM Top 10 — LLM08: Excessive Agency](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [MITRE ATLAS — ML Model Access](https://atlas.mitre.org/)
- [LiteLLM GitHub — BerriAI](https://github.com/BerriAI/litellm)

---
*Banco Ficticio S.A. y todos los datos de usuario usados en este lab son completamente ficticios. Cualquier semejanza con entidades reales es coincidencia.*
