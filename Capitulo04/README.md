# Práctica 4: Desplegar LiteLLM + OPA y restringir acceso a modelos según rol

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

```powershell id="m83qyz"
# Crear estructura del laboratorio

$folders = @(
    "lab-04-00-01",
    "lab-04-00-01\litellm",
    "lab-04-00-01\opa",
    "lab-04-00-01\opa\policies",
    "lab-04-00-01\opa\data",
    "lab-04-00-01\mock-identity",
    "lab-04-00-01\mock-llm",
    "lab-04-00-01\tests"
)

foreach ($folder in $folders) {

    New-Item `
        -ItemType Directory `
        -Force `
        -Path $folder | Out-Null
}

# Entrar a la carpeta principal

Set-Location "lab-04-00-01"

# Crear .gitignore

@'
.env
*.env
*.pem
*.key
*.crt
__pycache__/
*.pyc
.pytest_cache/
'@ | Out-File .gitignore -Encoding utf8

# Crear .env.example

@'
# Copiar a .env y completar valores reales

OPENAI_API_KEY=sk-REPLACE_ME
LITELLM_MASTER_KEY=sk-lab-master-REPLACE_ME
JWT_SECRET=super-secret-jwt-key-REPLACE_ME
USE_MOCK_LLM=true
'@ | Out-File .env.example -Encoding utf8

# Copiar plantilla

Copy-Item .env.example .env -Force

# Mensaje final

Write-Host "Edita .env con credenciales reales antes de continuar"
```


## Instrucciones Paso a Paso

---

### Paso 1: Configurar el Mock de Proveedor LLM (alternativa offline)

**Objetivo**: Crear un servidor FastAPI que simule las respuestas de OpenAI para poder completar el lab sin incurrir en costes de API. Si tienes una API key real, puedes omitir la construcción del mock pero igualmente debes revisar este paso para entender la arquitectura.

#### Instrucciones

**1.1** Crear el Dockerfile del mock LLM:

```powershell id="0kkg4q"
# Crear carpeta si no existe

New-Item `
    -ItemType Directory `
    -Path .\mock-llm `
    -Force

# Crear Dockerfile

@'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 9090

CMD uvicorn main:app --host 0.0.0.0 --port 9090
'@ | Out-File .\mock-llm\Dockerfile -Encoding utf8
```

**1.2** Crear las dependencias del mock LLM:

```text
# Crear requirements.txt del mock-llm en UTF-8
@'
fastapi==0.109.2
uvicorn==0.27.1
'@ | Set-Content .\mock-llm\requirements.txt -Encoding UTF8
```

**1.3** Crear el servidor FastAPI que simula OpenAI:

```powershell id="fyjlwm"
# Crear main.py del mock-llm

@'
# -*- coding: utf-8 -*-
"""
Mock LLM API - Simula respuestas de OpenAI
para pruebas sin coste.

DISCLAIMER:
Este servidor es unicamente para fines
educativos y de prueba.

Las respuestas son ficticias y no provienen
de ningun modelo real.
"""

import time
import uuid

from fastapi import (
    FastAPI,
    Request,
)

from fastapi.responses import JSONResponse


app = FastAPI(
    title="Mock LLM API",
    version="1.0.0"
)


MOCK_RESPONSES = {

    "gpt-3.5-turbo":
        "Respuesta simulada del modelo "
        "tier-1 (gpt-3.5-turbo). [MOCK]",

    "gpt-4o":
        "Respuesta simulada del modelo "
        "tier-2 (gpt-4o). [MOCK]",

    "default":
        "Respuesta simulada generica. [MOCK]",
}


@app.post("/v1/chat/completions")
async def chat_completions(
    request: Request
):

    body = await request.json()

    model = body.get(
        "model",
        "default"
    )

    content = MOCK_RESPONSES.get(
        model,
        MOCK_RESPONSES["default"]
    )

    return JSONResponse({

        "id":
            f"chatcmpl-mock-"
            f"{uuid.uuid4().hex[:8]}",

        "object":
            "chat.completion",

        "created":
            int(time.time()),

        "model":
            model,

        "choices": [
            {
                "index": 0,

                "message": {
                    "role": "assistant",
                    "content": content
                },

                "finish_reason": "stop"
            }
        ],

        "usage": {
            "prompt_tokens": 10,
            "completion_tokens": 20,
            "total_tokens": 30
        }
    })


@app.get("/health")
async def health():

    return {
        "status": "ok",
        "service": "mock-llm"
    }
'@ | Out-File .\mock-llm\main.py -Encoding utf8
```


**Salida esperada**: Los archivos del mock LLM están creados correctamente.

**Verificación**:
```bash
# Listar contenido de la carpeta mock-llm
Get-ChildItem .\mock-llm\
```

---

### Paso 2: Configurar el Servicio de Mock de Identidad (JWT Generator)

**Objetivo**: Crear un microservicio que genere JWTs con claims de rol para simular un Identity Provider (IdP) real. Este servicio emitirá tokens para los roles `analyst`, `developer` y `admin`.

#### Instrucciones

**2.1** Crear el Dockerfile del mock de identidad:

```powershell id="3yajxj"
# Crear carpeta mock-identity si no existe

New-Item `
    -ItemType Directory `
    -Path .\mock-identity `
    -Force

# Crear Dockerfile de mock-identity

@'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 8080

CMD uvicorn main:app --host 0.0.0.0 --port 8080
'@ | Out-File .\mock-identity\Dockerfile -Encoding utf8
```


**2.2** Crear las dependencias:

```powershell id="9n8h5m"
# Crear requirements.txt de mock-identity

@'
fastapi==0.109.2
uvicorn==0.27.1
python-jose[cryptography]==3.3.0
'@ | Out-File .\mock-identity\requirements.txt -Encoding utf8
```


**2.3** Crear el generador de JWTs:

```powershell id="h2nq7v"
# Crear main.py de mock-identity

@'
# -*- coding: utf-8 -*-
"""
Mock Identity Provider

Genera JWTs con claims de rol
para pruebas de RBAC.

DISCLAIMER:
Solo para uso en laboratorio.
No usar en produccion.
"""

import os
import time

from fastapi import (
    FastAPI,
    HTTPException,
)

from jose import jwt

from pydantic import BaseModel


app = FastAPI(
    title="Mock Identity Provider",
    version="1.0.0"
)


JWT_SECRET = os.environ.get(
    "JWT_SECRET",
    "super-secret-jwt-key-for-lab-only"
)

JWT_ALGORITHM = "HS256"

JWT_EXPIRY_SECONDS = 3600


VALID_ROLES = {
    "analyst",
    "developer",
    "admin"
}


ROLE_PERMISSIONS = {

    "analyst": {
        "tier": "tier-1",
        "rpm_limit": 10,
        "monthly_budget_usd": 50,
    },

    "developer": {
        "tier": "tier-2",
        "rpm_limit": 50,
        "monthly_budget_usd": 200,
    },

    "admin": {
        "tier": "tier-all",
        "rpm_limit": -1,
        "monthly_budget_usd": -1,
    },
}


class TokenRequest(BaseModel):

    user_id: str

    role: str

    team: str = "default"


@app.post("/token")
async def generate_token(
    req: TokenRequest
):

    if req.role not in VALID_ROLES:

        raise HTTPException(

            status_code=400,

            detail=(
                "Rol invalido. "
                f"Validos: {VALID_ROLES}"
            )
        )

    perms = ROLE_PERMISSIONS[
        req.role
    ]

    now = int(time.time())

    payload = {

        "sub":
            req.user_id,

        "role":
            req.role,

        "team":
            req.team,

        "tier":
            perms["tier"],

        "rpm_limit":
            perms["rpm_limit"],

        "monthly_budget":
            perms["monthly_budget_usd"],

        "iat":
            now,

        "exp":
            now + JWT_EXPIRY_SECONDS,

        "iss":
            "mock-identity-provider-lab",
    }

    token = jwt.encode(
        payload,
        JWT_SECRET,
        algorithm=JWT_ALGORITHM
    )

    return {

        "access_token":
            token,

        "token_type":
            "bearer",

        "role":
            req.role,

        "expires_in":
            JWT_EXPIRY_SECONDS,
    }


@app.get("/health")
async def health():

    return {
        "status": "ok",
        "service": "mock-identity"
    }
'@ | Out-File .\mock-identity\main.py -Encoding utf8
```

**Salida esperada**: Archivos del mock de identidad creados.

**Verificación**:
```bash
# Listar contenido de la carpeta mock-identity
Get-ChildItem .\mock-identity\ -Force
```

---

### Paso 3: Escribir las Políticas Rego en OPA

**Objetivo**: Implementar las reglas RBAC con restricciones temporales en el lenguaje Rego de OPA. Estas políticas serán el núcleo del control de acceso.

#### Instrucciones

**3.1** Crear el catálogo de modelos por tier:

```powershell id="q7m2ra"
# Crear estructura de carpetas para OPA

New-Item `
    -ItemType Directory `
    -Path .\opa\data `
    -Force

# Crear model_tiers.json

@'
{
  "tiers": {

    "tier-1": {
      "models": [
        "gpt-3.5-turbo",
        "chat-standard"
      ],

      "description":
        "Modelos economicos para tareas rutinarias",

      "production": false
    },

    "tier-2": {
      "models": [
        "gpt-4o",
        "gpt-4o-mini",
        "chat-advanced"
      ],

      "description":
        "Modelos avanzados para tareas complejas",

      "production": true
    },

    "tier-all": {
      "models": [
        "gpt-3.5-turbo",
        "chat-standard",
        "gpt-4o",
        "gpt-4o-mini",
        "chat-advanced",
        "claude-3-sonnet",
        "chat-premium"
      ],

      "description":
        "Acceso completo solo administradores",

      "production": true
    }
  },

  "business_hours": {

    "start_hour_utc": 8,

    "end_hour_utc": 20,

    "timezone_note":
      "UTC ajustar segun zona horaria del equipo"
  }
}
'@ | Out-File .\opa\data\model_tiers.json -Encoding utf8
```


**3.2** Crear la política Rego principal:

```powershell id="v2j6nk"
# Crear politica principal de OPA

@'
package llm.access

import future.keywords.if
import future.keywords.in

default allow := false

allow if {
    role_exists
    model_allowed_for_role
    not production_model_outside_hours
}

valid_roles := {
    "analyst",
    "developer",
    "admin"
}

role_exists if {
    input.jwt_claims.role in valid_roles
}

# Analyst solo tier-1

model_allowed_for_role if {
    input.jwt_claims.role == "analyst"
    input.model in data.tiers["tier-1"].models
}

# Developer tier-1

model_allowed_for_role if {
    input.jwt_claims.role == "developer"
    input.model in data.tiers["tier-1"].models
}

# Developer tier-2

model_allowed_for_role if {
    input.jwt_claims.role == "developer"
    input.model in data.tiers["tier-2"].models
}

# Admin acceso total

model_allowed_for_role if {
    input.jwt_claims.role == "admin"
    input.model in data.tiers["tier-all"].models
}

is_production_model if {
    input.model in data.tiers["tier-2"].models
}

is_production_model if {
    input.model in data.tiers["tier-all"].models
    not input.model in data.tiers["tier-1"].models
    not input.model in data.tiers["tier-2"].models
}

within_business_hours if {

    hour := time.clock(input.request_time_ns)[0]

    hour >= data.business_hours.start_hour_utc

    hour < data.business_hours.end_hour_utc
}

production_model_outside_hours if {
    is_production_model
    not within_business_hours
}

deny_reason := reason if {

    not role_exists

    reason := sprintf(
        "Rol no reconocido o no presente en el JWT",
        []
    )
}

deny_reason := reason if {

    role_exists
    not model_allowed_for_role

    reason := sprintf(
        "Rol sin permisos para usar el modelo solicitado",
        []
    )
}

deny_reason := reason if {

    role_exists
    model_allowed_for_role
    production_model_outside_hours

    reason := sprintf(
        "Modelo disponible solo en horario laboral UTC",
        []
    )
}

deny_reason := "Solicitud denegada por politica de acceso LLM" if {
    not allow
}
'@ | Out-File .\opa\policies\llm_access.rego -Encoding utf8
```


**Salida esperada**: Archivos de política OPA creados sin errores de sintaxis.

**Verificación** (requiere OPA CLI instalado localmente, o se verifica en el Paso 6):
```bash
# Verificación de sintaxis (opcional si tienes OPA CLI instalado)
# opa check .\opa\policies\llm_access.rego

Write-Host "Archivos de política creados:"


# Listar policies
Get-ChildItem .\opa\policies\ -Force


# Listar data
Get-ChildItem .\opa\data\ -Force
```

---

### Paso 4: Configurar LiteLLM con Virtual Keys y Rate Limiting

**Objetivo**: Crear el archivo de configuración de LiteLLM con los tres roles, sus virtual keys, rate limits diferenciados y presupuestos mensuales. Configurar también el hook de autorización que consulta OPA.

#### Instrucciones

**4.1** Crear la configuración de LiteLLM:

```powershell id="j7m4za"
# Crear config.yaml de LiteLLM

@'
# Configuracion de LiteLLM AI Gateway
# Lab 04-00-01

# Las credenciales reales se inyectan
# usando variables de entorno

model_list:

  # Tier 1

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


  # Tier 2

  - model_name: chat-advanced

    litellm_params:
      model: gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY

    model_info:
      tier: tier-2
      id: chat-advanced


  - model_name: gpt-4o

    litellm_params:
      model: gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY

    model_info:
      tier: tier-2
      id: gpt-4o


# LiteLLM Settings

litellm_settings:

  request_timeout: 30

  max_tokens: 256

  temperature: 0.1

  drop_params: true

  # Hook de autorizacion
  callbacks: []


# General Settings

general_settings:

  master_key: os.environ/LITELLM_MASTER_KEY

  database_url: null

  store_model_in_db: false


# Virtual Keys por rol

# analyst-key
# rate_limit: 10 RPM
# budget: 50 USD por mes
# modelos: tier-1

# developer-key
# rate_limit: 50 RPM
# budget: 200 USD por mes
# modelos: tier-1 y tier-2

# admin-key
# rate_limit: sin limite
# budget: sin limite
# modelos: todos

'@ | Out-File .\litellm\config.yaml -Encoding utf8
```


**4.2** Crear el script de inicialización de virtual keys:

```powershell id="g8k2wr"
# Crear create_virtual_keys.py

@'
# -*- coding: utf-8 -*-
"""
Script para crear virtual keys de LiteLLM
usando Admin API.

Ejecutar despues de que el contenedor
de LiteLLM este disponible.
"""

import os
import sys
import time
import requests


LITELLM_URL = os.environ.get(
    "LITELLM_URL",
    "http://localhost:4000"
)

MASTER_KEY = os.environ.get(
    "LITELLM_MASTER_KEY",
    ""
)


if not MASTER_KEY:

    print(
        "ERROR: "
        "LITELLM_MASTER_KEY "
        "no esta definida."
    )

    sys.exit(1)


HEADERS = {

    "Authorization":
        f"Bearer {MASTER_KEY}",

    "Content-Type":
        "application/json"
}


VIRTUAL_KEYS = [

    {
        "key_alias":
            "analyst-key",

        "models": [
            "chat-standard",
            "gpt-3.5-turbo"
        ],

        "tpm_limit":
            5000,

        "rpm_limit":
            10,

        "max_budget":
            50.0,

        "budget_duration":
            "1mo",

        "metadata": {
            "role": "analyst",
            "team": "data-analytics",
            "tier": "tier-1"
        }
    },

    {
        "key_alias":
            "developer-key",

        "models": [
            "chat-standard",
            "gpt-3.5-turbo",
            "chat-advanced",
            "gpt-4o"
        ],

        "tpm_limit":
            20000,

        "rpm_limit":
            50,

        "max_budget":
            200.0,

        "budget_duration":
            "1mo",

        "metadata": {
            "role": "developer",
            "team": "engineering",
            "tier": "tier-2"
        }
    },

    {
        "key_alias":
            "admin-key",

        # Lista vacia acceso total
        "models": [],

        "tpm_limit":
            None,

        "rpm_limit":
            None,

        "max_budget":
            None,

        "budget_duration":
            "1mo",

        "metadata": {
            "role": "admin",
            "team": "platform",
            "tier": "tier-all"
        }
    },
]


def wait_for_litellm(
    max_retries: int = 30,
    delay: float = 2.0
):
    """
    Espera hasta que LiteLLM
    responda en /health.
    """

    print(
        "Esperando que LiteLLM "
        f"este disponible en {LITELLM_URL}..."
    )

    for i in range(max_retries):

        try:

            r = requests.get(
                f"{LITELLM_URL}/health",
                timeout=5
            )

            if r.status_code == 200:

                print(
                    "LiteLLM disponible."
                )

                return True

        except requests.exceptions.ConnectionError:

            pass

        print(
            f"Intento "
            f"{i+1}/{max_retries} "
            f"... reintentando en {delay}s"
        )

        time.sleep(delay)

    print(
        "LiteLLM no respondio a tiempo."
    )

    return False


def create_key(
    key_config: dict
) -> dict:
    """
    Crea una virtual key
    usando Admin API.
    """

    payload = {

        k: v

        for k, v in key_config.items()

        if v is not None
    }

    r = requests.post(

        f"{LITELLM_URL}/key/generate",

        json=payload,

        headers=HEADERS,

        timeout=10
    )

    r.raise_for_status()

    return r.json()


if __name__ == "__main__":

    if not wait_for_litellm():

        sys.exit(1)

    created_keys = {}

    for key_cfg in VIRTUAL_KEYS:

        alias = key_cfg["key_alias"]

        try:

            result = create_key(
                key_cfg
            )

            api_key = result.get(
                "key",
                "N/A"
            )

            created_keys[
                alias
            ] = api_key

            print(
                f"{alias}: "
                f"{api_key[:20]}..."
            )

        except Exception as e:

            print(
                f"Error creando "
                f"{alias}: {e}"
            )

    print()

    print(
        "Resumen de Virtual Keys"
    )

    for alias, key in created_keys.items():

        print(
            f"{alias:15s} "
            f"-> {key}"
        )

    print()

    print(
        "Guardar estas keys "
        "en el archivo .env."
    )
'@ | Out-File .\litellm\create_virtual_keys.py -Encoding utf8
```


**Salida esperada**: Archivos de configuración de LiteLLM creados.

**Verificación**:
```bash
# Listar contenido de la carpeta litellm
Get-ChildItem .\litellm\ -Force
```

---

### Paso 5: Crear el Docker Compose Stack

**Objetivo**: Ensamblar todos los servicios en un único `docker-compose.yml` con networking correcto, health checks y orden de arranque.

#### Instrucciones

**5.1** Crear el archivo Docker Compose:

```powershell id="p8m2xa"
# Crear docker-compose.yml

@'
version: "3.9"

networks:

  lab-net:
    driver: bridge


services:

  # Mock LLM Provider

  mock-llm:

    build:
      context: ./mock-llm
      dockerfile: Dockerfile

    container_name: lab-mock-llm

    networks:
      - lab-net

    ports:
      - "9090:9090"

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "wget -qO- http://localhost:9090/health || exit 1"
        ]

      interval: 10s
      timeout: 5s
      retries: 5

    restart: unless-stopped


  # Mock Identity Provider

  mock-identity:

    build:
      context: ./mock-identity
      dockerfile: Dockerfile

    container_name: lab-mock-identity

    networks:
      - lab-net

    ports:
      - "8080:8080"

    environment:
      - JWT_SECRET=${JWT_SECRET:-super-secret-jwt-key-for-lab-only}

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "wget -qO- http://localhost:8080/health || exit 1"
        ]

      interval: 10s
      timeout: 5s
      retries: 5

    restart: unless-stopped


  # Open Policy Agent

  opa:

    image: openpolicyagent/opa:0.65.0

    container_name: lab-opa

    networks:
      - lab-net

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
      test:
        [
          "CMD-SHELL",
          "wget -qO- http://localhost:8181/ || exit 1"
        ]

      interval: 10s
      timeout: 5s
      retries: 10

    restart: unless-stopped


  # LiteLLM AI Gateway

  litellm:

    image: ghcr.io/berriai/litellm:main-v1.40.10

    container_name: lab-litellm

    networks:
      - lab-net

    ports:
      - "4000:4000"

    volumes:
      - ./litellm/config.yaml:/app/config.yaml:ro

    environment:

      # Credenciales proveedor LLM

      - OPENAI_API_KEY=${OPENAI_API_KEY:-dummy-key-for-mock}

      - LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY:-sk-lab-master-key}

      # Mock backend opcional

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
      test:
        [
          "CMD-SHELL",
          "wget -qO- http://localhost:4000/health || exit 1"
        ]

      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 30s

    restart: unless-stopped
'@ | Out-File .\docker-compose.yml -Encoding utf8
```


**5.2** Si vas a usar el mock LLM, añadir `OPENAI_API_BASE` al `.env`:

```powershell id="x4m8qp"
# Agregar variables al archivo .env

Add-Content `
    -Path .env `
    -Value "OPENAI_API_BASE=http://mock-llm:9090/v1"

Add-Content `
    -Path .env `
    -Value "USE_MOCK_LLM=true"
```


**5.3** Levantar el stack:

```bash
# Construir imágenes y levantar servicios
docker compose up --build -d


# Verificar estado de contenedores
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
```powershell id="p4m8qx"
# Health check Mock Identity

Invoke-RestMethod `
    -Uri "http://localhost:8080/health" `
    -Method GET `
| ConvertTo-Json -Depth 10


# Health check OPA

Invoke-RestMethod `
    -Uri "http://localhost:8181/health" `
    -Method GET `
| ConvertTo-Json -Depth 10


# Health check LiteLLM

Invoke-RestMethod `
    -Uri "http://localhost:4000/health" `
    -Method GET `
| ConvertTo-Json -Depth 10


# Verificar politicas cargadas en OPA

(
    Invoke-RestMethod `
        -Uri "http://localhost:8181/v1/policies" `
        -Method GET
).result | ConvertTo-Json -Depth 10
```


---

### Paso 6: Crear las Virtual Keys y Probar OPA Directamente

**Objetivo**: Inicializar las virtual keys en LiteLLM y verificar manualmente que OPA evalúa correctamente las políticas Rego antes de ejecutar la batería de tests completa.

#### Instrucciones

**6.1** Instalar dependencias Python para el lab:

```powershell id="z8m2qa"
# Crear requirements.txt de tests

@'
requests==2.32.3
python-jose[cryptography]==3.3.0
pytest==8.2.2
pytest-html==4.1.1
tabulate==0.9.0
'@ | Out-File .\tests\requirements.txt -Encoding utf8


# Instalar dependencias

pip install -r .\tests\requirements.txt
```


**6.2** Crear las virtual keys ejecutando el script de inicialización:

```powershell id="x7m4qp"
# Cargar variables del archivo .env

Get-Content .env | ForEach-Object {

    # Ignorar comentarios y lineas vacias

    if (
        $_ -match '^\s*#' -or
        $_ -match '^\s*$'
    ) {
        return
    }

    $name, $value = $_ -split '=', 2

    [System.Environment]::SetEnvironmentVariable(
        $name,
        $value,
        "Process"
    )
}
```


**6.3** Probar OPA directamente con casos manuales:

```powershell id="m5q8ra"
# Timestamp actual en nanosegundos UTC

$timestampNs = (
    [DateTimeOffset]::UtcNow.ToUnixTimeSeconds().ToString()
) + "000000000"


# TEST MANUAL 1
# Analyst + tier-1
# Resultado esperado: allow = true

$body1 = @{
    input = @{
        jwt_claims = @{
            role = "analyst"
            sub  = "user-001"
            team = "data-analytics"
        }

        model = "gpt-3.5-turbo"

        request_time_ns = [Int64]$timestampNs
    }
} | ConvertTo-Json -Depth 10


Invoke-RestMethod `
    -Uri "http://localhost:8181/v1/data/llm/access" `
    -Method POST `
    -ContentType "application/json" `
    -Body $body1 `
| ConvertTo-Json -Depth 10


# TEST MANUAL 2
# Analyst + tier-2
# Resultado esperado: allow = false

$body2 = @{
    input = @{
        jwt_claims = @{
            role = "analyst"
            sub  = "user-001"
            team = "data-analytics"
        }

        model = "gpt-4o"

        request_time_ns = [Int64]$timestampNs
    }
} | ConvertTo-Json -Depth 10


Invoke-RestMethod `
    -Uri "http://localhost:8181/v1/data/llm/access" `
    -Method POST `
    -ContentType "application/json" `
    -Body $body2 `
| ConvertTo-Json -Depth 10


# TEST MANUAL 3
# Developer + tier-2
# Resultado esperado: allow = true

$body3 = @{
    input = @{
        jwt_claims = @{
            role = "developer"
            sub  = "user-002"
            team = "engineering"
        }

        model = "gpt-4o"

        request_time_ns = [Int64]$timestampNs
    }
} | ConvertTo-Json -Depth 10


Invoke-RestMethod `
    -Uri "http://localhost:8181/v1/data/llm/access" `
    -Method POST `
    -ContentType "application/json" `
    -Body $body3 `
| ConvertTo-Json -Depth 10
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
```powershell id="n4q7ra"
# Mostrar ultimas decisiones registradas por OPA

docker logs lab-opa 2>&1 `
| Select-String "decision_id" `
| Select-Object -Last 5
```


---

### Paso 7: Ejecutar la Batería Completa de 12 Tests

**Objetivo**: Crear y ejecutar el script de pruebas automatizado que cubre las 12 combinaciones de rol × modelo × horario, verificando que OPA permite y deniega correctamente cada caso.

#### Instrucciones

**7.1** Crear el script de tests:

```powershell id="w3m8qp"
# Crear test_policies.py

@'
# -*- coding: utf-8 -*-
"""
Bateria de tests para verificar enforcement
de politicas RBAC en OPA.

Cubre:
roles x modelos x restriccion temporal.

DISCLAIMER:
Los usuarios y datos usados
son completamente ficticios.
"""

import os
import time
import requests
import pytest

from jose import jwt
from tabulate import tabulate


OPA_URL = os.environ.get(
    "OPA_URL",
    "http://localhost:8181"
)

IDENTITY_URL = os.environ.get(
    "IDENTITY_URL",
    "http://localhost:8080"
)

JWT_SECRET = os.environ.get(
    "JWT_SECRET",
    "super-secret-jwt-key-for-lab-only"
)

OPA_POLICY = (
    f"{OPA_URL}/v1/data/llm/access"
)


# Casos de prueba

TEST_CASES = [

    (
        1,
        "analyst",
        "gpt-3.5-turbo",
        10,
        True,
        "Analyst tier-1 horario laboral"
    ),

    (
        2,
        "analyst",
        "chat-standard",
        14,
        True,
        "Analyst alias tier-1"
    ),

    (
        3,
        "analyst",
        "gpt-4o",
        10,
        False,
        "Analyst tier-2"
    ),

    (
        4,
        "analyst",
        "chat-advanced",
        10,
        False,
        "Analyst alias tier-2"
    ),

    (
        5,
        "developer",
        "gpt-3.5-turbo",
        11,
        True,
        "Developer tier-1"
    ),

    (
        6,
        "developer",
        "gpt-4o",
        15,
        True,
        "Developer tier-2"
    ),

    (
        7,
        "developer",
        "chat-advanced",
        9,
        True,
        "Developer alias tier-2"
    ),

    (
        8,
        "developer",
        "gpt-4o",
        3,
        False,
        "Developer tier-2 fuera horario"
    ),

    (
        9,
        "admin",
        "gpt-3.5-turbo",
        12,
        True,
        "Admin tier-1"
    ),

    (
        10,
        "admin",
        "gpt-4o",
        16,
        True,
        "Admin tier-2"
    ),

    (
        11,
        "admin",
        "chat-premium",
        22,
        False,
        "Admin produccion fuera horario"
    ),

    (
        12,
        "unknown",
        "gpt-3.5-turbo",
        10,
        False,
        "Rol invalido"
    ),
]


def build_opa_input(
    role: str,
    model: str,
    hour_utc: int
) -> dict:
    """
    Construye payload para OPA.
    """

    now = int(time.time())

    current_hour = int(
        time.strftime(
            "%H",
            time.gmtime(now)
        )
    )

    offset_seconds = (
        hour_utc - current_hour
    ) * 3600

    forced_time_ns = (
        now + offset_seconds
    ) * 1000000000

    return {

        "input": {

            "jwt_claims": {

                "role":
                    role,

                "sub":
                    f"test-user-{role}",

                "team":
                    f"team-{role}",

                "iat":
                    now,

                "exp":
                    now + 3600
            },

            "model":
                model,

            "request_time_ns":
                forced_time_ns
        }
    }


def evaluate_policy(
    role: str,
    model: str,
    hour_utc: int
) -> dict:
    """
    Ejecuta evaluacion en OPA.
    """

    payload = build_opa_input(
        role,
        model,
        hour_utc
    )

    resp = requests.post(
        OPA_POLICY,
        json=payload,
        timeout=10
    )

    resp.raise_for_status()

    return resp.json().get(
        "result",
        {}
    )


def run_all_tests():
    """
    Ejecuta todos los tests.
    """

    results = []

    passed = 0

    failed = 0

    for (
        test_id,
        role,
        model,
        hour,
        expected,
        description
    ) in TEST_CASES:

        try:

            result = evaluate_policy(
                role,
                model,
                hour
            )

            actual_allow = result.get(
                "allow",
                False
            )

            deny_reason = result.get(
                "deny_reason",
                ""
            )

            status = (
                "PASS"
                if actual_allow == expected
                else "FAIL"
            )

            if actual_allow == expected:

                passed += 1

            else:

                failed += 1

        except Exception as e:

            actual_allow = None

            deny_reason = str(e)

            status = "ERROR"

            failed += 1

        results.append([

            test_id,

            role,

            model,

            f"{hour:02d}:00 UTC",

            "ALLOW"
            if expected
            else "DENY",

            "ALLOW"
            if actual_allow
            else "DENY",

            status,

            (
                deny_reason[:60]
                if deny_reason
                else ""
            )
        ])

    return results, passed, failed


# Pytest parametrizado

@pytest.mark.parametrize(
    (
        "test_id,"
        "role,"
        "model,"
        "hour,"
        "expected,"
        "description"
    ),
    TEST_CASES
)
def test_opa_policy(
    test_id,
    role,
    model,
    hour,
    expected,
    description
):

    result = evaluate_policy(
        role,
        model,
        hour
    )

    actual = result.get(
        "allow",
        False
    )

    deny_reason = result.get(
        "deny_reason",
        ""
    )

    assert actual == expected, (

        f"\nTest #{test_id}: {description}\n"

        f"Rol: {role}\n"

        f"Modelo: {model}\n"

        f"Hora: {hour:02d}:00 UTC\n"

        f"Esperado: "
        f"{'ALLOW' if expected else 'DENY'}\n"

        f"Obtenido: "
        f"{'ALLOW' if actual else 'DENY'}\n"

        f"Razon OPA: {deny_reason}"
    )


# Ejecucion directa

if __name__ == "__main__":

    print()

    print("=" * 80)

    print(
        "BATERIA DE TESTS "
        "LiteLLM OPA RBAC"
    )

    print("=" * 80)

    print()

    results, passed, failed = run_all_tests()

    headers = [

        "#",
        "Rol",
        "Modelo",
        "Hora",
        "Esperado",
        "Obtenido",
        "Estado",
        "Razon OPA"
    ]

    print(
        tabulate(
            results,
            headers=headers,
            tablefmt="grid"
        )
    )

    print()

    print("=" * 80)

    print(
        f"Resultados: "
        f"{passed} PASS | "
        f"{failed} FAIL | "
        f"{len(TEST_CASES)} TOTAL"
    )

    print("=" * 80)

    print()

    if failed > 0:

        print(
            "Algunos tests fallaron."
        )

        print(
            "Revisar politicas Rego "
            "y archivos en opa/data/"
        )

        exit(1)

    else:

        print(
            "Todos los tests pasaron."
        )
'@ | Out-File .\tests\test_policies.py -Encoding utf8
```

**7.2** Ejecutar la batería de tests:

```powershell id="p7m4qa"
# OPCION A
# Entrar a la carpeta tests

Set-Location .\tests


# Cargar variables del archivo .env

Get-Content ..\.env | ForEach-Object {

    # Ignorar comentarios y lineas vacias

    if (
        $_ -match '^\s*#' -or
        $_ -match '^\s*$'
    ) {
        return
    }

    $name, $value = $_ -split '=', 2

    [System.Environment]::SetEnvironmentVariable(
        $name,
        $value,
        "Process"
    )
}


# Ejecutar bateria de pruebas

python .\test_policies.py
```

```powershell id="u5m8qp"
# OPCION B
# Ejecutar pruebas con pytest y generar reporte HTML

# Entrar a la carpeta tests

Set-Location .\tests


# Cargar variables del archivo .env

Get-Content ..\.env | ForEach-Object {

    # Ignorar comentarios y lineas vacias

    if (
        $_ -match '^\s*#' -or
        $_ -match '^\s*$'
    ) {
        return
    }

    $name, $value = $_ -split '=', 2

    [System.Environment]::SetEnvironmentVariable(
        $name,
        $value,
        "Process"
    )
}


# Ejecutar pytest y generar reporte HTML

pytest `
    .\test_policies.py `
    -v `
    --html=report.html `
    --self-contained-html


# Mensaje final

Write-Host "Reporte generado en tests/report.html"
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
```powershell id="r7m2qa"
# Ejecutar pytest mostrando resumen final

pytest `
    .\tests\test_policies.py `
    -v `
    --tb=short 2>&1 `
| Select-Object -Last 5
```


---


### Referencias

- [Documentación oficial de LiteLLM — Virtual Keys](https://docs.litellm.ai/docs/proxy/virtual_keys)
- [OPA — Rego Language Reference](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [OWASP LLM Top 10 — LLM08: Excessive Agency](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [MITRE ATLAS — ML Model Access](https://atlas.mitre.org/)
- [LiteLLM GitHub — BerriAI](https://github.com/BerriAI/litellm)

---
*Banco Ficticio S.A. y todos los datos de usuario usados en este lab son completamente ficticios. Cualquier semejanza con entidades reales es coincidencia.*
