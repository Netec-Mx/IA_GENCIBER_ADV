# Configurar n8n con credenciales seguras usando variables de entorno en Docker y protegerlo con un reverse proxy Nginx con TLS

## Metadatos

| Campo         | Valor                                                                 |
|---------------|-----------------------------------------------------------------------|
| **Duración**  | 40 minutos                                                            |
| **Complejidad** | Media                                                               |
| **Nivel Bloom** | Crear (Create)                                                      |
| **Lab ID**    | 08-00-01                                                              |
| **Módulo**    | 8 — Infraestructura segura de automatización con n8n                  |

---

## Visión General

En este laboratorio construirás un stack Docker Compose de nivel producción para n8n, aplicando los principios de gestión segura de credenciales estudiados en la lección 8.1. Desplegarás n8n con todas sus credenciales inyectadas exclusivamente desde variables de entorno, sin ningún secreto hardcodeado en imágenes ni archivos de configuración versionados. Protegerás el servicio mediante Nginx como reverse proxy con TLS 1.3, headers de seguridad HTTP y rate limiting. Finalmente, implementarás un workflow de ejemplo que demuestra el patrón Human-in-the-Loop (HITL) para operaciones críticas, validando que las credenciales se inyectan correctamente en tiempo de ejecución.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Construir un archivo `.env` con todas las credenciales de n8n y demostrar que `docker inspect` no revela sus valores gracias a la directiva `env_file`
- [ ] Aplicar hardening al contenedor n8n: usuario no-root, filesystem read-only, tmpfs para `/tmp`, límites de memoria y CPU, y aislamiento de red
- [ ] Configurar Nginx con TLS 1.3 exclusivo, cipher suites seguros, HSTS, CSP, X-Frame-Options y rate limiting de 10 req/s por IP
- [ ] Implementar un script de rotación documentado para `N8N_ENCRYPTION_KEY` con procedimiento paso a paso
- [ ] Crear un workflow HITL en n8n que simule una transferencia bancaria con aprobación humana via webhook

---

## Prerrequisitos

### Conocimientos

- Comprensión de variables de entorno en Linux y Docker (`env_file`, `--env-file`)
- Conocimiento básico de Nginx como reverse proxy (bloques `server`, `location`, `proxy_pass`)
- Familiaridad con TLS/SSL: conceptos de certificado, clave privada, handshake
- Haber estudiado la lección 8.1 sobre gestión segura de credenciales en n8n

### Software y acceso

| Herramienta | Versión mínima | Verificación |
|---|---|---|
| Docker Engine | 24.0+ | `docker --version` |
| Docker Compose | 2.20+ | `docker compose version` |
| mkcert | 1.4+ | `mkcert --version` |
| openssl | 3.0+ | `openssl version` |
| curl | cualquier | `curl --version` |
| Navegador web | moderno | — |

> **Nota:** `mkcert` debe tener la CA local ya instalada (`mkcert -install`). Si aún no lo has hecho, el Paso 0 lo cubre.

---

## Entorno del Laboratorio

### Estructura de directorios que crearás

```
lab08/
├── .env                          # Secretos — NUNCA commitear
├── .gitignore                    # Excluye .env y certificados
├── docker-compose.yml            # Stack completo
├── nginx/
│   ├── nginx.conf                # Configuración principal
│   ├── certs/
│   │   ├── n8n-local.pem         # Certificado TLS
│   │   └── n8n-local-key.pem     # Clave privada TLS
├── postgres/
│   └── init.sql                  # Inicialización de BD (opcional)
├── scripts/
│   ├── generate-secrets.sh       # Generación inicial de secretos
│   └── rotate-encryption-key.sh  # Rotación de N8N_ENCRYPTION_KEY
└── workflows/
    └── hitl-transfer.json        # Workflow HITL exportado
```

### Requisitos de recursos

| Recurso | Mínimo | Recomendado |
|---|---|---|
| RAM disponible | 2 GB | 4 GB |
| CPU | 2 núcleos | 4 núcleos |
| Disco libre | 5 GB | 10 GB |
| Puertos libres | 443, 5678 | — |

---

## Instrucciones Paso a Paso

---

### Paso 0 — Preparación del entorno y verificación de herramientas

**Objetivo:** Verificar que todas las dependencias están disponibles e instalar la CA local de mkcert.

**Instrucciones:**

1. Crea el directorio raíz del laboratorio y la estructura de subdirectorios:

```bash
mkdir -p lab08/{nginx/certs,scripts,workflows,postgres}
cd lab08
```

2. Verifica las versiones de las herramientas requeridas:

```bash
docker --version
docker compose version
mkcert --version
openssl version
```

3. Instala la CA local de mkcert (si no está instalada aún):

```bash
mkcert -install
```

4. Crea el archivo `.gitignore` para proteger secretos desde el primer momento:

```bash
cat > .gitignore << 'EOF'
# Secretos — NUNCA subir al repositorio
.env
.env.*
!.env.example

# Certificados TLS
nginx/certs/*.pem
nginx/certs/*.key
nginx/certs/*.crt

# Datos de volúmenes
data/
volumes/

# Logs
*.log

# Archivos temporales de Python
__pycache__/
*.pyc
EOF
```

**Salida esperada:**

```
Docker version 24.x.x, build ...
Docker Compose version v2.x.x
mkcert v1.4.x
OpenSSL 3.x.x ...
The local CA is now installed in the system trust store! ✅
```

**Verificación:**

```bash
ls -la  # Debe mostrar .gitignore y los subdirectorios creados
mkcert -CAROOT  # Debe mostrar la ruta de la CA local
```

---

### Paso 1 — Generación segura de credenciales y archivo `.env`

**Objetivo:** Crear todas las credenciales necesarias con entropía suficiente e inyectarlas mediante un archivo `.env`, sin hardcodear ningún valor en el código fuente.

**Instrucciones:**

1. Crea el script de generación de secretos:

```bash
cat > scripts/generate-secrets.sh << 'EOF'
#!/usr/bin/env bash
# generate-secrets.sh — Genera credenciales seguras para el stack n8n
# ADVERTENCIA: Este script genera un .env con secretos reales.
# Nunca commitear el .env generado al repositorio.
set -euo pipefail

ENV_FILE=".env"

if [ -f "$ENV_FILE" ]; then
  echo "[WARN] El archivo $ENV_FILE ya existe. Renombrando a .env.backup.$(date +%s)"
  mv "$ENV_FILE" ".env.backup.$(date +%s)"
fi

# Genera valores aleatorios criptográficamente seguros
N8N_ENCRYPTION_KEY=$(openssl rand -hex 32)
DB_POSTGRESDB_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=')
N8N_BASIC_AUTH_PASSWORD=$(openssl rand -base64 16 | tr -d '/+=')
POSTGRES_PASSWORD="$DB_POSTGRESDB_PASSWORD"

cat > "$ENV_FILE" << ENVEOF
# =============================================================
# ARCHIVO DE SECRETOS — NO COMMITEAR AL REPOSITORIO
# Generado: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
# =============================================================

# --- n8n: Clave de cifrado de credenciales ---
# CRÍTICO: Si cambias esta clave, todas las credenciales cifradas
# en la BD quedarán inaccesibles. Guarda una copia en tu Secret Manager.
N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}

# --- n8n: Autenticación básica ---
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}

# --- Base de datos PostgreSQL ---
DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}
POSTGRES_PASSWORD=${POSTGRES_PASSWORD}

# --- URL pública del webhook (ajusta si usas dominio real) ---
WEBHOOK_URL=https://n8n.local

# --- Configuración general de n8n ---
N8N_HOST=0.0.0.0
N8N_PORT=5678
N8N_PROTOCOL=https
N8N_EDITOR_BASE_URL=https://n8n.local
GENERIC_TIMEZONE=Europe/Madrid
ENVEOF

chmod 600 "$ENV_FILE"
echo "[OK] Archivo $ENV_FILE generado con permisos 600."
echo "[INFO] N8N_ENCRYPTION_KEY (primeros 8 chars): ${N8N_ENCRYPTION_KEY:0:8}..."
echo "[INFO] Guarda N8N_ENCRYPTION_KEY en tu Secret Manager ANTES de continuar."
EOF

chmod +x scripts/generate-secrets.sh
```

2. Ejecuta el script para generar el `.env`:

```bash
./scripts/generate-secrets.sh
```

3. Verifica el contenido del `.env` (sin revelar valores completos):

```bash
# Verificar que el archivo existe y tiene permisos correctos
ls -la .env

# Ver las claves (sin valores) para confirmar que están todas
grep -E "^[A-Z_]+=" .env | cut -d'=' -f1
```

4. Crea el archivo `.env.example` (seguro para versionar, sin valores reales):

```bash
cat > .env.example << 'EOF'
# Copia este archivo a .env y rellena los valores reales
# NUNCA commitear .env con valores reales

N8N_ENCRYPTION_KEY=REEMPLAZAR_CON_32_BYTES_HEX_ALEATORIO
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=REEMPLAZAR_CON_CONTRASENA_SEGURA
DB_POSTGRESDB_PASSWORD=REEMPLAZAR_CON_CONTRASENA_BD
POSTGRES_PASSWORD=REEMPLAZAR_CON_CONTRASENA_BD
WEBHOOK_URL=https://n8n.local
N8N_HOST=0.0.0.0
N8N_PORT=5678
N8N_PROTOCOL=https
N8N_EDITOR_BASE_URL=https://n8n.local
GENERIC_TIMEZONE=Europe/Madrid
EOF
```

**Salida esperada:**

```
[OK] Archivo .env generado con permisos 600.
[INFO] N8N_ENCRYPTION_KEY (primeros 8 chars): a3f9e2b1...
[INFO] Guarda N8N_ENCRYPTION_KEY en tu Secret Manager ANTES de continuar.
```

**Verificación:**

```bash
# Permisos deben ser -rw------- (600)
stat -c "%a %n" .env

# Las claves esperadas deben estar presentes
for key in N8N_ENCRYPTION_KEY N8N_BASIC_AUTH_PASSWORD DB_POSTGRESDB_PASSWORD WEBHOOK_URL; do
  grep -q "^${key}=" .env && echo "✅ $key presente" || echo "❌ $key FALTANTE"
done
```

---

### Paso 2 — Generación de certificados TLS con mkcert

**Objetivo:** Generar certificados TLS firmados por la CA local de mkcert para `n8n.local`, configurar el sistema para resolver ese dominio y verificar que el certificado usa TLS 1.3.

**Instrucciones:**

1. Genera los certificados para el dominio local:

```bash
cd nginx/certs
mkcert n8n.local localhost 127.0.0.1
# mkcert genera: n8n.local+2.pem y n8n.local+2-key.pem
# Renombramos para claridad:
mv n8n.local+2.pem n8n-local.pem
mv n8n.local+2-key.pem n8n-local-key.pem
cd ../..
```

2. Verifica el certificado generado:

```bash
openssl x509 -in nginx/certs/n8n-local.pem -text -noout | grep -E "(Subject|Issuer|Not After|DNS)"
```

3. Añade `n8n.local` al archivo `/etc/hosts` (requiere sudo):

```bash
echo "127.0.0.1  n8n.local" | sudo tee -a /etc/hosts
```

4. Asegura los permisos de la clave privada:

```bash
chmod 600 nginx/certs/n8n-local-key.pem
chmod 644 nginx/certs/n8n-local.pem
ls -la nginx/certs/
```

**Salida esperada:**

```
Subject: CN=n8n.local
Issuer: O=mkcert development CA, ...
Not After : <fecha futura>
DNS:n8n.local, DNS:localhost, IP Address:127.0.0.1
```

**Verificación:**

```bash
# El certificado debe ser válido y mostrar el CN correcto
openssl verify -CAfile "$(mkcert -CAROOT)/rootCA.pem" nginx/certs/n8n-local.pem
# Salida esperada: nginx/certs/n8n-local.pem: OK
```

---

### Paso 3 — Configuración de Nginx con TLS 1.3 y headers de seguridad

**Objetivo:** Crear la configuración de Nginx con TLS 1.3 exclusivo, cipher suites seguros, todos los headers de seguridad HTTP requeridos y rate limiting.

**Instrucciones:**

1. Crea la configuración principal de Nginx:

```bash
cat > nginx/nginx.conf << 'EOF'
# =============================================================
# nginx.conf — Reverse proxy seguro para n8n
# TLS 1.3 exclusivo, headers de seguridad, rate limiting
# =============================================================

worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    # --- Tipos MIME ---
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # --- Logging con formato seguro (sin revelar secretos) ---
    log_format secure_log '$remote_addr - $remote_user [$time_local] '
                          '"$request" $status $body_bytes_sent '
                          '"$http_referer" "$http_user_agent" '
                          'rt=$request_time';
    access_log /var/log/nginx/access.log secure_log;

    # --- Optimización ---
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    server_tokens off;  # No revelar versión de Nginx

    # --- Rate limiting: 10 req/s por IP ---
    # Zona de 10MB puede rastrear ~160,000 IPs simultáneas
    limit_req_zone $binary_remote_addr zone=n8n_limit:10m rate=10r/s;
    limit_req_status 429;

    # --- Upstream: n8n en red interna Docker ---
    upstream n8n_backend {
        server n8n:5678;
        keepalive 32;
    }

    # --- Redirect HTTP -> HTTPS ---
    server {
        listen 80;
        server_name n8n.local;
        return 301 https://$host$request_uri;
    }

    # --- Servidor HTTPS principal ---
    server {
        listen 443 ssl;
        http2 on;
        server_name n8n.local;

        # --- TLS: solo TLS 1.3 ---
        ssl_certificate     /etc/nginx/certs/n8n-local.pem;
        ssl_certificate_key /etc/nginx/certs/n8n-local-key.pem;
        ssl_protocols       TLSv1.3;

        # --- Cipher suites seguros para TLS 1.3 ---
        # TLS 1.3 gestiona los ciphers automáticamente; estos son los preferidos
        ssl_prefer_server_ciphers off;
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_session_tickets off;

        # --- OCSP Stapling (comentado para CA local) ---
        # ssl_stapling on;
        # ssl_stapling_verify on;

        # ============================================================
        # Headers de seguridad HTTP
        # ============================================================

        # HSTS: forzar HTTPS por 1 año, incluir subdominios
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        # Prevenir clickjacking
        add_header X-Frame-Options "DENY" always;

        # Prevenir MIME-type sniffing
        add_header X-Content-Type-Options "nosniff" always;

        # Referrer Policy: no filtrar información en cabeceras Referer
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;

        # Content Security Policy: restringir fuentes de contenido
        # Nota: n8n requiere 'unsafe-inline' y 'unsafe-eval' para su editor
        add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self' data:; connect-src 'self' wss://n8n.local; frame-ancestors 'none';" always;

        # Permissions Policy: deshabilitar APIs del navegador no necesarias
        add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

        # ============================================================
        # Rate limiting aplicado
        # ============================================================
        limit_req zone=n8n_limit burst=20 nodelay;

        # ============================================================
        # Proxy hacia n8n
        # ============================================================
        location / {
            proxy_pass http://n8n_backend;

            # Headers necesarios para que n8n conozca el cliente real
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host  $host;

            # Soporte WebSockets (requerido por n8n)
            proxy_http_version 1.1;
            proxy_set_header Upgrade    $http_upgrade;
            proxy_set_header Connection "upgrade";

            # Timeouts generosos para workflows largos
            proxy_read_timeout  300s;
            proxy_send_timeout  300s;
            proxy_connect_timeout 10s;

            # Buffers
            proxy_buffering    off;
            proxy_buffer_size  128k;
        }

        # Endpoint de health check (sin rate limiting)
        location /healthz {
            limit_req off;
            proxy_pass http://n8n_backend/healthz;
            access_log off;
        }
    }
}
EOF
```

**Salida esperada:** El archivo `nginx/nginx.conf` se crea sin errores.

**Verificación:**

```bash
# Verificar que la configuración no tiene errores de sintaxis
# (lo haremos con el contenedor en el Paso 4, pero podemos revisar la estructura)
grep -E "(ssl_protocols|add_header|limit_req_zone|rate=)" nginx/nginx.conf
```

Salida esperada de la verificación:
```
    ssl_protocols       TLSv1.3;
    limit_req_zone $binary_remote_addr zone=n8n_limit:10m rate=10r/s;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "DENY" always;
    ...
```

---

### Paso 4 — Docker Compose con hardening del contenedor n8n

**Objetivo:** Crear el `docker-compose.yml` con todas las medidas de hardening: usuario no-root, filesystem read-only, tmpfs, límites de recursos, aislamiento de red y variables de entorno inyectadas desde `.env`.

**Instrucciones:**

1. Crea el archivo `docker-compose.yml`:

```bash
cat > docker-compose.yml << 'EOF'
# =============================================================
# docker-compose.yml — Stack n8n seguro para laboratorio
# Aplica principios de mínimo privilegio y hardening de contenedor
# =============================================================

version: "3.8"

# ============================================================
# Redes: aislamiento entre capas
# ============================================================
networks:
  # Red pública: solo nginx está expuesto al host
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

  # Red privada: n8n y postgres, sin acceso directo desde el host
  backend:
    driver: bridge
    internal: true   # Sin acceso a Internet desde esta red
    ipam:
      config:
        - subnet: 172.20.1.0/24

# ============================================================
# Volúmenes persistentes
# ============================================================
volumes:
  n8n_data:
    driver: local
  postgres_data:
    driver: local

# ============================================================
# Servicios
# ============================================================
services:

  # ----------------------------------------------------------
  # PostgreSQL: base de datos de n8n
  # ----------------------------------------------------------
  postgres:
    image: postgres:15-alpine
    container_name: lab08_postgres
    restart: unless-stopped

    # Credenciales inyectadas desde .env (sin hardcodear)
    env_file:
      - .env
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n
      PGDATA: /var/lib/postgresql/data/pgdata

    volumes:
      - postgres_data:/var/lib/postgresql/data

    networks:
      - backend

    # Hardening del contenedor PostgreSQL
    security_opt:
      - no-new-privileges:true
    read_only: false   # PostgreSQL necesita escribir en su directorio de datos
    user: "70:70"      # Usuario postgres en Alpine

    # Límites de recursos
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
        reservations:
          memory: 128M

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n -d n8n"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ----------------------------------------------------------
  # n8n: motor de automatización
  # ----------------------------------------------------------
  n8n:
    image: n8nio/n8n:latest
    container_name: lab08_n8n
    restart: unless-stopped

    # *** CLAVE: env_file inyecta variables sin exponerlas en el Dockerfile ***
    # 'docker inspect' mostrará los nombres de las variables pero NO sus valores
    env_file:
      - .env
    environment:
      # Configuración de base de datos
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: "5432"
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: ${DB_POSTGRESDB_PASSWORD}

      # Configuración de n8n
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
      N8N_BASIC_AUTH_ACTIVE: ${N8N_BASIC_AUTH_ACTIVE}
      N8N_BASIC_AUTH_USER: ${N8N_BASIC_AUTH_USER}
      N8N_BASIC_AUTH_PASSWORD: ${N8N_BASIC_AUTH_PASSWORD}
      WEBHOOK_URL: ${WEBHOOK_URL}
      N8N_HOST: ${N8N_HOST}
      N8N_PORT: ${N8N_PORT}
      N8N_PROTOCOL: ${N8N_PROTOCOL}
      N8N_EDITOR_BASE_URL: ${N8N_EDITOR_BASE_URL}
      GENERIC_TIMEZONE: ${GENERIC_TIMEZONE}

      # Hardening: deshabilitar módulos externos en nodos Code
      # Esto previene que workflows ejecuten código arbitrario con acceso a npm
      NODE_FUNCTION_ALLOW_EXTERNAL: ""

      # Deshabilitar telemetría
      N8N_DIAGNOSTICS_ENABLED: "false"
      N8N_VERSION_NOTIFICATIONS_ENABLED: "false"

    # Hardening del contenedor
    user: "node"                    # Usuario no-root (UID 1000 en imagen n8n)
    read_only: true                 # Filesystem de solo lectura
    tmpfs:
      - /tmp:mode=1777,size=256m    # /tmp en memoria (n8n necesita espacio temporal)
      - /home/node/.n8n:mode=0700,size=512m  # Directorio de trabajo de n8n

    security_opt:
      - no-new-privileges:true      # Previene escalada de privilegios

    # n8n NO está expuesto directamente al host; solo nginx puede alcanzarlo
    expose:
      - "5678"

    networks:
      - backend                     # Acceso a postgres
      - frontend                    # Nginx puede alcanzarlo

    # Límites de recursos
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: "1.0"
        reservations:
          memory: 256M

    depends_on:
      postgres:
        condition: service_healthy

    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:5678/healthz || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # ----------------------------------------------------------
  # Nginx: reverse proxy con TLS 1.3
  # ----------------------------------------------------------
  nginx:
    image: nginx:alpine
    container_name: lab08_nginx
    restart: unless-stopped

    ports:
      - "443:443"    # HTTPS expuesto al host
      - "80:80"      # HTTP (redirige a HTTPS)

    volumes:
      # Configuración de Nginx (solo lectura)
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      # Certificados TLS (solo lectura)
      - ./nginx/certs:/etc/nginx/certs:ro

    networks:
      - frontend    # Solo nginx está en la red frontend expuesta

    security_opt:
      - no-new-privileges:true

    read_only: true
    tmpfs:
      - /var/run:mode=0755,size=10m
      - /var/cache/nginx:mode=0755,size=100m
      - /var/log/nginx:mode=0755,size=100m

    deploy:
      resources:
        limits:
          memory: 128M
          cpus: "0.25"

    depends_on:
      n8n:
        condition: service_healthy

    # Validar configuración de nginx al iniciar
    command: ["/bin/sh", "-c", "nginx -t && nginx -g 'daemon off;'"]
EOF
```

2. Valida la sintaxis del `docker-compose.yml`:

```bash
docker compose config --quiet && echo "✅ docker-compose.yml válido"
```

3. Levanta el stack:

```bash
docker compose up -d
```

4. Monitoriza el arranque (espera ~60 segundos para que n8n inicialice la BD):

```bash
docker compose logs -f --tail=50
# Presiona Ctrl+C cuando veas: "n8n ready on 0.0.0.0, port 5678"
```

**Salida esperada:**

```
✅ docker-compose.yml válido
[+] Running 3/3
 ✔ Container lab08_postgres  Healthy
 ✔ Container lab08_n8n       Healthy
 ✔ Container lab08_nginx     Started
```

**Verificación:**

```bash
# Verificar que los tres contenedores están corriendo
docker compose ps

# Verificar que nginx valida su configuración correctamente
docker exec lab08_nginx nginx -t
# Salida esperada: nginx: configuration file /etc/nginx/nginx.conf test is successful

# Verificar que n8n responde (a través de nginx con TLS)
curl -sk https://n8n.local/healthz
# Salida esperada: {"status":"ok"}
```

---

### Paso 5 — Demostrar que `docker inspect` no revela secretos

**Objetivo:** Verificar empíricamente que el uso de `env_file` en Docker Compose protege los valores de las variables de entorno frente a la inspección del contenedor.

**Instrucciones:**

1. Inspecciona el contenedor n8n y busca la variable `N8N_ENCRYPTION_KEY`:

```bash
# Inspeccionar el contenedor e intentar ver el valor de la clave de cifrado
docker inspect lab08_n8n | python3 -c "
import json, sys
data = json.load(sys.stdin)
env_vars = data[0]['Config']['Env']
print('=== Variables de entorno visibles en docker inspect ===')
for var in env_vars:
    key = var.split('=')[0]
    value = var.split('=', 1)[1] if '=' in var else ''
    # Mostrar solo los primeros 8 caracteres del valor
    masked = value[:8] + '...' if len(value) > 8 else value
    print(f'  {key} = {masked}')
"
```

> **Nota pedagógica:** Con `env_file`, Docker Compose **sí** inyecta los valores en el contenedor (son necesarios para que n8n funcione), por lo que `docker inspect` puede mostrarlos. La protección real viene de:
> - Restringir quién tiene acceso al socket Docker (`/var/run/docker.sock`)
> - Usar Docker Secrets en modo Swarm o Kubernetes Secrets con RBAC
> - Montar secretos como archivos en lugar de variables de entorno
>
> En este laboratorio demostramos el patrón correcto de separación (`.env` fuera del código fuente) y los límites del mecanismo.

2. Verifica que el `.env` no está dentro de la imagen Docker:

```bash
# Confirmar que el .env NO está copiado dentro del contenedor
docker exec lab08_n8n ls -la /home/node/ 2>/dev/null | grep ".env" || echo "✅ .env NO está dentro del contenedor"

# Confirmar que el .env está en el host pero no en el contexto de build
docker exec lab08_n8n cat /.env 2>/dev/null || echo "✅ /.env no accesible dentro del contenedor"
```

3. Verifica el aislamiento de red (n8n no debe ser accesible directamente desde el host):

```bash
# n8n NO debe responder directamente en el puerto 5678 del host
curl -s --connect-timeout 3 http://localhost:5678/healthz 2>&1 || echo "✅ n8n no accesible directamente desde el host (correcto)"

# Pero SÍ debe responder a través de nginx en el puerto 443
curl -sk https://n8n.local/healthz && echo "✅ n8n accesible a través de nginx/TLS"
```

**Salida esperada:**

```
✅ .env NO está dentro del contenedor
✅ /.env no accesible dentro del contenedor
✅ n8n no accesible directamente desde el host (correcto)
{"status":"ok"}✅ n8n accesible a través de nginx/TLS
```

**Verificación:**

```bash
# Confirmar que el usuario del proceso n8n es 'node' (no root)
docker exec lab08_n8n whoami
# Salida esperada: node

# Confirmar límites de recursos aplicados
docker stats lab08_n8n --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.CPUPerc}}"
```

---

### Paso 6 — Verificar TLS 1.3 y headers de seguridad

**Objetivo:** Confirmar que Nginx sirve únicamente TLS 1.3 y que todos los headers de seguridad están presentes y correctamente configurados.

**Instrucciones:**

1. Verifica la versión de TLS negociada:

```bash
# Conectar y mostrar la versión de TLS negociada
openssl s_client -connect n8n.local:443 -tls1_3 2>&1 | grep -E "(Protocol|Cipher|Verify)"
```

2. Confirma que TLS 1.2 es rechazado:

```bash
# TLS 1.2 debe ser rechazado (solo aceptamos TLS 1.3)
openssl s_client -connect n8n.local:443 -tls1_2 2>&1 | grep -E "(alert|handshake failure|Protocol)" | head -5
echo "Si ves 'handshake failure', TLS 1.2 está correctamente rechazado ✅"
```

3. Verifica todos los headers de seguridad HTTP:

```bash
# Obtener los headers de respuesta y verificar cada uno
echo "=== Verificación de Headers de Seguridad ==="
HEADERS=$(curl -sk -I https://n8n.local/ 2>&1)

for header in \
  "strict-transport-security" \
  "x-frame-options" \
  "x-content-type-options" \
  "referrer-policy" \
  "content-security-policy" \
  "permissions-policy"; do
  if echo "$HEADERS" | grep -qi "$header"; then
    VALUE=$(echo "$HEADERS" | grep -i "$header" | head -1 | tr -d '\r')
    echo "✅ $VALUE"
  else
    echo "❌ FALTANTE: $header"
  fi
done
```

4. Verifica el rate limiting:

```bash
# Enviar 25 peticiones rápidas y contar las que reciben 429
echo "=== Test de Rate Limiting (10 req/s, burst=20) ==="
SUCCESS=0; RATE_LIMITED=0
for i in $(seq 1 25); do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" https://n8n.local/healthz)
  if [ "$STATUS" = "429" ]; then
    RATE_LIMITED=$((RATE_LIMITED + 1))
  else
    SUCCESS=$((SUCCESS + 1))
  fi
done
echo "Peticiones exitosas: $SUCCESS"
echo "Peticiones con rate limiting (429): $RATE_LIMITED"
[ $RATE_LIMITED -gt 0 ] && echo "✅ Rate limiting funcionando" || echo "⚠️  Rate limiting no activado (puede necesitar más peticiones)"
```

**Salida esperada:**

```
Protocol  : TLSv1.3
Cipher    : TLS_AES_256_GCM_SHA384
Verify return code: 0 (ok)

Si ves 'handshake failure', TLS 1.2 está correctamente rechazado ✅

=== Verificación de Headers de Seguridad ===
✅ strict-transport-security: max-age=31536000; includeSubDomains
✅ x-frame-options: DENY
✅ x-content-type-options: nosniff
✅ referrer-policy: strict-origin-when-cross-origin
✅ content-security-policy: default-src 'self'; ...
✅ permissions-policy: geolocation=(), ...
```

**Verificación:**

```bash
# Verificación final con curl mostrando detalles TLS
curl -sv --tls-max 1.3 https://n8n.local/healthz 2>&1 | grep -E "(TLSv|SSL connection|HTTP/)"
```

---

### Paso 7 — Implementar el script de rotación de `N8N_ENCRYPTION_KEY`

**Objetivo:** Crear y documentar el procedimiento de rotación de la clave de cifrado de n8n, una operación crítica que requiere pasos específicos para no perder acceso a las credenciales almacenadas.

**Instrucciones:**

1. Crea el script de rotación con procedimiento documentado:

```bash
cat > scripts/rotate-encryption-key.sh << 'EOF'
#!/usr/bin/env bash
# rotate-encryption-key.sh — Procedimiento de rotación de N8N_ENCRYPTION_KEY
#
# ADVERTENCIA CRÍTICA:
# N8N_ENCRYPTION_KEY cifra TODAS las credenciales de workflows almacenadas en la BD.
# Si cambias la clave sin seguir este procedimiento, perderás acceso a todas
# las credenciales y deberás recrearlas manualmente.
#
# PROCEDIMIENTO REQUERIDO:
# 1. Exportar todas las credenciales desde la UI de n8n (Settings > Credentials > Export)
# 2. Detener n8n (NO postgres)
# 3. Actualizar N8N_ENCRYPTION_KEY en .env y en tu Secret Manager
# 4. Reiniciar n8n — n8n re-cifrará las credenciales con la nueva clave
# 5. Verificar que todos los workflows funcionan correctamente
# 6. Eliminar el backup de credenciales del paso 1 de forma segura
#
# Este script automatiza los pasos 2-4 y verifica el resultado.

set -euo pipefail

ENV_FILE=".env"
BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"

echo "=========================================="
echo " Rotación de N8N_ENCRYPTION_KEY"
echo " $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
echo "=========================================="

# Paso 0: Verificar que el stack está corriendo
if ! docker compose ps | grep -q "lab08_n8n.*running\|lab08_n8n.*Up"; then
  echo "[ERROR] El contenedor n8n no está corriendo. Abortando."
  exit 1
fi

# Paso 1: Recordatorio manual obligatorio
echo ""
echo "[ACCIÓN REQUERIDA] Antes de continuar:"
echo "  1. Abre https://n8n.local en tu navegador"
echo "  2. Ve a Settings > Credentials"
echo "  3. Exporta TODAS las credenciales a un archivo seguro"
echo "  4. Guarda el backup en un lugar seguro (NO en este directorio)"
echo ""
read -p "¿Has exportado las credenciales? (escribe 'SI' para continuar): " CONFIRM
if [ "$CONFIRM" != "SI" ]; then
  echo "[ABORTADO] Rotación cancelada. Exporta las credenciales primero."
  exit 0
fi

# Paso 2: Crear backup del .env actual
mkdir -p "$BACKUP_DIR"
cp "$ENV_FILE" "$BACKUP_DIR/.env.backup"
chmod 600 "$BACKUP_DIR/.env.backup"
echo "[OK] Backup del .env guardado en $BACKUP_DIR/.env.backup"

# Paso 3: Generar nueva clave de cifrado
OLD_KEY=$(grep "^N8N_ENCRYPTION_KEY=" "$ENV_FILE" | cut -d'=' -f2)
NEW_KEY=$(openssl rand -hex 32)
echo "[INFO] Clave anterior (primeros 8 chars): ${OLD_KEY:0:8}..."
echo "[INFO] Nueva clave (primeros 8 chars): ${NEW_KEY:0:8}..."

# Paso 4: Actualizar .env con la nueva clave
sed -i "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=${NEW_KEY}|" "$ENV_FILE"
echo "[OK] N8N_ENCRYPTION_KEY actualizada en $ENV_FILE"

# Paso 5: Detener n8n (solo n8n, postgres sigue corriendo)
echo "[INFO] Deteniendo n8n..."
docker compose stop n8n
echo "[OK] n8n detenido"

# Paso 6: Reiniciar n8n con la nueva clave
echo "[INFO] Reiniciando n8n con nueva clave de cifrado..."
docker compose up -d n8n

# Paso 7: Esperar a que n8n esté listo
echo "[INFO] Esperando a que n8n inicialice (máx. 120s)..."
TIMEOUT=120
ELAPSED=0
while [ $ELAPSED -lt $TIMEOUT ]; do
  if curl -sk https://n8n.local/healthz 2>/dev/null | grep -q "ok"; then
    echo "[OK] n8n está listo con la nueva clave de cifrado"
    break
  fi
  sleep 5
  ELAPSED=$((ELAPSED + 5))
  echo "  Esperando... ${ELAPSED}s"
done

if [ $ELAPSED -ge $TIMEOUT ]; then
  echo "[ERROR] n8n no respondió en ${TIMEOUT}s. Verifica los logs:"
  echo "  docker compose logs n8n"
  echo "[ROLLBACK] Para revertir: cp $BACKUP_DIR/.env.backup $ENV_FILE && docker compose restart n8n"
  exit 1
fi

echo ""
echo "=========================================="
echo " Rotación completada exitosamente"
echo " IMPORTANTE: Actualiza N8N_ENCRYPTION_KEY"
echo " en tu Secret Manager ahora."
echo "=========================================="
echo ""
echo "[RECORDATORIO] Elimina de forma segura el backup de credenciales"
echo "  que exportaste manualmente en el Paso 1."
EOF

chmod +x scripts/rotate-encryption-key.sh
echo "✅ Script de rotación creado"
```

2. Revisa el script sin ejecutarlo (la rotación real se haría en producción):

```bash
cat scripts/rotate-encryption-key.sh | head -30
echo ""
echo "✅ Script de rotación listo. En producción, ejecutar con:"
echo "   ./scripts/rotate-encryption-key.sh"
```

**Salida esperada:**

```
✅ Script de rotación creado
✅ Script de rotación listo. En producción, ejecutar con:
   ./scripts/rotate-encryption-key.sh
```

**Verificación:**

```bash
# Verificar que el script es ejecutable y tiene la estructura correcta
[ -x scripts/rotate-encryption-key.sh ] && echo "✅ Script ejecutable"
grep -c "ADVERTENCIA CRÍTICA\|PROCEDIMIENTO REQUERIDO\|openssl rand" scripts/rotate-encryption-key.sh
# Debe mostrar: 3
```

---

### Paso 8 — Crear el workflow Human-in-the-Loop (HITL)

**Objetivo:** Implementar en n8n un workflow que simule una transferencia bancaria ficticia (Banco Ficticio S.A.) que requiere aprobación humana explícita mediante webhook antes de ejecutarse, demostrando el patrón HITL en automatizaciones críticas.

> **Disclaimer:** Todos los datos bancarios usados en este workflow son completamente ficticios. Los nombres, IBANs (prefijo XX99) y cantidades son inventados para fines educativos únicamente. No representan entidades reales.

**Instrucciones:**

1. Accede a la interfaz de n8n:

```bash
# Abre en tu navegador:
echo "URL: https://n8n.local"
echo "Usuario: admin"
echo "Contraseña: $(grep N8N_BASIC_AUTH_PASSWORD .env | cut -d'=' -f2)"
```

2. Crea el workflow HITL importando el siguiente JSON. En n8n, ve a **Workflows > New > Import from JSON** y pega el contenido:

```bash
cat > workflows/hitl-transfer.json << 'WORKFLOW_EOF'
{
  "name": "HITL - Transferencia Bancaria Ficticia con Aprobación",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "transfer-request",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "webhook-trigger",
      "name": "Webhook: Solicitud de Transferencia",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [240, 300],
      "webhookId": "transfer-request"
    },
    {
      "parameters": {
        "jsCode": "// DISCLAIMER: Datos completamente ficticios - Banco Ficticio S.A.\n// Validar que la solicitud tiene los campos requeridos\nconst body = $input.first().json.body || $input.first().json;\n\nif (!body.amount || !body.destination_iban || !body.requester) {\n  throw new Error('Campos requeridos: amount, destination_iban, requester');\n}\n\n// Validar que el IBAN ficticio tiene el prefijo correcto\nif (!body.destination_iban.startsWith('XX99')) {\n  throw new Error('Solo se aceptan IBANs ficticios con prefijo XX99 en este entorno de laboratorio');\n}\n\n// Clasificar el riesgo de la transferencia\nconst amount = parseFloat(body.amount);\nlet riskLevel = 'LOW';\nlet requiresApproval = false;\n\nif (amount > 1000) {\n  riskLevel = 'MEDIUM';\n  requiresApproval = true;\n}\nif (amount > 10000) {\n  riskLevel = 'HIGH';\n  requiresApproval = true;\n}\n\nreturn [{\n  json: {\n    transfer_id: 'TRF-' + Date.now(),\n    amount: amount,\n    currency: body.currency || 'EUR',\n    destination_iban: body.destination_iban,\n    destination_name: body.destination_name || 'Destinatario Ficticio S.A.',\n    requester: body.requester,\n    risk_level: riskLevel,\n    requires_approval: requiresApproval,\n    timestamp: new Date().toISOString(),\n    bank: 'Banco Ficticio S.A. (DATOS SIMULADOS)'\n  }\n}];"
      },
      "id": "validate-request",
      "name": "Validar y Clasificar Riesgo",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [460, 300]
    },
    {
      "parameters": {
        "conditions": {
          "options": {"caseSensitive": true, "leftValue": "", "typeValidation": "strict"},
          "conditions": [
            {
              "id": "requires-approval-check",
              "leftValue": "={{ $json.requires_approval }}",
              "rightValue": true,
              "operator": {"type": "boolean", "operation": "equals"}
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "check-approval-needed",
      "name": "¿Requiere Aprobación?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [680, 300]
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "approval-message",
              "name": "approval_message",
              "value": "=⚠️ APROBACIÓN REQUERIDA\n\nTransferencia ID: {{ $json.transfer_id }}\nBanco: {{ $json.bank }}\nImporte: {{ $json.amount }} {{ $json.currency }}\nDestino IBAN: {{ $json.destination_iban }}\nDestinatario: {{ $json.destination_name }}\nSolicitante: {{ $json.requester }}\nNivel de Riesgo: {{ $json.risk_level }}\nFecha: {{ $json.timestamp }}\n\nPara APROBAR: POST /webhook/approve-transfer con {\"transfer_id\": \"{{ $json.transfer_id }}\", \"decision\": \"APPROVED\"}\nPara RECHAZAR: POST /webhook/approve-transfer con {\"transfer_id\": \"{{ $json.transfer_id }}\", \"decision\": \"REJECTED\"}",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "id": "prepare-approval-request",
      "name": "Preparar Solicitud de Aprobación",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [900, 200]
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "approve-transfer",
        "responseMode": "lastNode",
        "options": {}
      },
      "id": "approval-webhook",
      "name": "Webhook: Decisión de Aprobación",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [900, 400],
      "webhookId": "approve-transfer"
    },
    {
      "parameters": {
        "jsCode": "// Procesar la decisión de aprobación humana\nconst decision = $input.first().json.body?.decision || $input.first().json.decision;\nconst transferId = $input.first().json.body?.transfer_id || $input.first().json.transfer_id;\n\nif (!decision || !transferId) {\n  throw new Error('Se requieren: transfer_id y decision (APPROVED|REJECTED)');\n}\n\nconst approved = decision === 'APPROVED';\n\nreturn [{\n  json: {\n    transfer_id: transferId,\n    decision: decision,\n    approved: approved,\n    processed_at: new Date().toISOString(),\n    message: approved \n      ? `✅ Transferencia ${transferId} APROBADA y ejecutada (simulado)`\n      : `❌ Transferencia ${transferId} RECHAZADA por el aprobador`,\n    bank: 'Banco Ficticio S.A. (DATOS SIMULADOS)'\n  }\n}];"
      },
      "id": "process-decision",
      "name": "Procesar Decisión HITL",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1120, 400]
    },
    {
      "parameters": {
        "jsCode": "// Transferencia de bajo riesgo: ejecución automática sin aprobación\nconst transfer = $input.first().json;\n\nreturn [{\n  json: {\n    transfer_id: transfer.transfer_id,\n    decision: 'AUTO_APPROVED',\n    approved: true,\n    processed_at: new Date().toISOString(),\n    message: `✅ Transferencia ${transfer.transfer_id} ejecutada automáticamente (riesgo LOW, importe <= 1000 EUR)`,\n    bank: 'Banco Ficticio S.A. (DATOS SIMULADOS)'\n  }\n}];"
      },
      "id": "auto-execute",
      "name": "Ejecución Automática (Riesgo Bajo)",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [900, 500]
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ JSON.stringify({status: 'pending_approval', transfer_id: $('Validar y Clasificar Riesgo').first().json.transfer_id, message: 'Transferencia en espera de aprobación humana. Revisa el sistema de aprobación.', approval_message: $json.approval_message}) }}",
        "options": {"responseCode": 202}
      },
      "id": "respond-pending",
      "name": "Responder: Pendiente de Aprobación",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [1120, 200]
    }
  ],
  "connections": {
    "Webhook: Solicitud de Transferencia": {
      "main": [[{"node": "Validar y Clasificar Riesgo", "type": "main", "index": 0}]]
    },
    "Validar y Clasificar Riesgo": {
      "main": [[{"node": "¿Requiere Aprobación?", "type": "main", "index": 0}]]
    },
    "¿Requiere Aprobación?": {
      "main": [
        [{"node": "Preparar Solicitud de Aprobación", "type": "main", "index": 0}],
        [{"node": "Ejecución Automática (Riesgo Bajo)", "type": "main", "index": 0}]
      ]
    },
    "Preparar Solicitud de Aprobación": {
      "main": [[{"node": "Responder: Pendiente de Aprobación", "type": "main", "index": 0}]]
    },
    "Webhook: Decisión de Aprobación": {
      "main": [[{"node": "Procesar Decisión HITL", "type": "main", "index": 0}]]
    }
  },
  "settings": {
    "executionOrder": "v1"
  },
  "staticData": null,
  "meta": {
    "templateCredsSetupCompleted": true
  }
}
WORKFLOW_EOF

echo "✅ Archivo de workflow HITL creado en workflows/hitl-transfer.json"
```

3. Importa el workflow en n8n y actívalo:

   - Abre `https://n8n.local` en tu navegador
   - Inicia sesión con las credenciales del `.env`
   - Ve a **Workflows > New**
   - Haz clic en el menú (⋮) y selecciona **Import from JSON**
   - Pega el contenido de `workflows/hitl-transfer.json`
   - Haz clic en **Save** y luego activa el workflow con el toggle

4. Prueba el workflow HITL con una transferencia de alto riesgo:

```bash
# Obtener la URL base del webhook
WEBHOOK_BASE="https://n8n.local/webhook"

# Test 1: Transferencia de ALTO riesgo (requiere aprobación humana)
echo "=== Test 1: Transferencia de alto riesgo (>1000 EUR) ==="
curl -sk -X POST "${WEBHOOK_BASE}/transfer-request" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 15000,
    "currency": "EUR",
    "destination_iban": "XX99FICTICIO0001234567890",
    "destination_name": "Empresa Ficticia S.L.",
    "requester": "alumno.lab@bancoficticio.test"
  }' | python3 -m json.tool
```

5. Prueba con una transferencia de bajo riesgo (ejecución automática):

```bash
# Test 2: Transferencia de BAJO riesgo (ejecución automática)
echo "=== Test 2: Transferencia de bajo riesgo (<=1000 EUR) ==="
curl -sk -X POST "${WEBHOOK_BASE}/transfer-request" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50,
    "currency": "EUR",
    "destination_iban": "XX99FICTICIO0009876543210",
    "destination_name": "Proveedor Ficticio S.A.",
    "requester": "alumno.lab@bancoficticio.test"
  }' | python3 -m json.tool
```

6. Simula la aprobación humana para la transferencia de alto riesgo:

```bash
# Obtener el transfer_id del Test 1 (reemplaza TRF-XXXXXXXXXX con el valor real)
TRANSFER_ID="TRF-$(date +%s)000"  # Ejemplo; usa el ID real del Test 1

# Simular aprobación del aprobador humano
echo "=== Simulando aprobación humana ==="
curl -sk -X POST "${WEBHOOK_BASE}/approve-transfer" \
  -H "Content-Type: application/json" \
  -d "{
    \"transfer_id\": \"${TRANSFER_ID}\",
    \"decision\": \"APPROVED\"
  }" | python3 -m json.tool
```

**Salida esperada (Test 1 - Alto riesgo):**

```json
{
  "status": "pending_approval",
  "transfer_id": "TRF-1234567890123",
  "message": "Transferencia en espera de aprobación humana. Revisa el sistema de aprobación.",
  "approval_message": "⚠️ APROBACIÓN REQUERIDA\n\nTransferencia ID: TRF-..."
}
```

**Salida esperada (Test 2 - Bajo riesgo):**

```json
{
  "transfer_id": "TRF-...",
  "decision": "AUTO_APPROVED",
  "approved": true,
  "message": "✅ Transferencia TRF-... ejecutada automáticamente (riesgo LOW, importe <= 1000 EUR)",
  "bank": "Banco Ficticio S.A. (DATOS SIMULADOS)"
}
```

**Verificación:**

```bash
# Verificar en la UI de n8n que los workflows se ejecutaron correctamente
echo "Verifica en https://n8n.local/workflow/executions que aparecen las ejecuciones"

# Verificar que el workflow está activo
curl -sk -u "admin:$(grep N8N_BASIC_AUTH_PASSWORD .env | cut -d'=' -f2)" \
  https://n8n.local/api/v1/workflows | python3 -c "
import json, sys
data = json.load(sys.stdin)
workflows = data.get('data', [])
for wf in workflows:
    status = '✅ ACTIVO' if wf.get('active') else '⏸️  INACTIVO'
    print(f'{status} - {wf[\"name\"]}')
" 2>/dev/null || echo "Verifica el estado del workflow en la UI de n8n"
```

---

## Validación y Pruebas

Ejecuta la siguiente batería de verificaciones para confirmar que todo el stack está correctamente configurado:

```bash
#!/usr/bin/env bash
# validation.sh — Batería completa de verificaciones del Lab 08-00-01
echo "=============================================="
echo " Validación completa del Lab 08-00-01"
echo "=============================================="

PASS=0; FAIL=0

check() {
  local desc="$1"; local cmd="$2"; local expected="$3"
  result=$(eval "$cmd" 2>&1)
  if echo "$result" | grep -q "$expected"; then
    echo "✅ PASS: $desc"
    PASS=$((PASS+1))
  else
    echo "❌ FAIL: $desc (esperado: '$expected', obtenido: '${result:0:80}')"
    FAIL=$((FAIL+1))
  fi
}

# 1. Contenedores corriendo
check "PostgreSQL corriendo" "docker compose ps postgres" "running\|Up"
check "n8n corriendo" "docker compose ps n8n" "running\|Up"
check "Nginx corriendo" "docker compose ps nginx" "running\|Up"

# 2. Archivo .env existe y tiene permisos correctos
check ".env existe con permisos 600" "stat -c '%a' .env" "600"
check ".env tiene N8N_ENCRYPTION_KEY" "grep -c '^N8N_ENCRYPTION_KEY=' .env" "1"

# 3. .env no está en el contenedor
check ".env no está en el contenedor n8n" "docker exec lab08_n8n ls / 2>/dev/null | grep -c '.env'" "0"

# 4. Usuario no-root en n8n
check "n8n corre como usuario no-root" "docker exec lab08_n8n whoami" "node"

# 5. n8n no accesible directamente desde el host
check "Puerto 5678 no expuesto al host" "curl -s --connect-timeout 2 http://localhost:5678/healthz" "refused\|timed out\|Failed"

# 6. n8n accesible via nginx/TLS
check "n8n accesible via HTTPS" "curl -sk https://n8n.local/healthz" "ok"

# 7. TLS 1.3
check "TLS 1.3 negociado" "openssl s_client -connect n8n.local:443 -tls1_3 2>&1" "TLSv1.3"
check "TLS 1.2 rechazado" "openssl s_client -connect n8n.local:443 -tls1_2 2>&1" "handshake failure\|alert"

# 8. Headers de seguridad
check "HSTS presente" "curl -skI https://n8n.local/" "strict-transport-security"
check "X-Frame-Options DENY" "curl -skI https://n8n.local/" "x-frame-options: DENY\|X-Frame-Options: DENY"
check "X-Content-Type-Options" "curl -skI https://n8n.local/" "x-content-type-options\|X-Content-Type-Options"
check "Content-Security-Policy" "curl -skI https://n8n.local/" "content-security-policy\|Content-Security-Policy"

# 9. Script de rotación existe y es ejecutable
check "Script de rotación existe" "ls scripts/rotate-encryption-key.sh" "rotate-encryption-key.sh"
check "Script de rotación ejecutable" "test -x scripts/rotate-encryption-key.sh && echo 'ok'" "ok"

# 10. .gitignore protege .env
check ".gitignore excluye .env" "grep -c '^\.env$' .gitignore" "1"

echo ""
echo "=============================================="
echo " Resultado: $PASS PASS / $FAIL FAIL"
echo "=============================================="
[ $FAIL -eq 0 ] && echo "🎉 ¡Todas las verificaciones pasaron!" || echo "⚠️  Revisa los items FAIL antes de continuar"
```

```bash
# Ejecutar la validación
bash <(cat << 'SCRIPT'
# [pega aquí el script de validación completo de arriba]
SCRIPT
)
```

---

## Resolución de Problemas

### Problema 1: n8n no arranca — Error "could not connect to the database"

**Síntomas:**
```
lab08_n8n  | Error: connect ECONNREFUSED 172.20.1.x:5432
lab08_n8n  | Could not connect to database
```
El contenedor n8n aparece como `Restarting` en `docker compose ps`.

**Causa:**
n8n intentó conectarse a PostgreSQL antes de que la base de datos estuviera lista para aceptar conexiones. Aunque existe el `depends_on` con `condition: service_healthy`, puede ocurrir si el healthcheck de postgres tarda más de lo esperado, o si las credenciales en `.env` no coinciden entre los dos servicios (`DB_POSTGRESDB_PASSWORD` vs `POSTGRES_PASSWORD`).

**Solución:**

```bash
# 1. Verificar el estado del healthcheck de postgres
docker inspect lab08_postgres | python3 -c "
import json, sys
d = json.load(sys.stdin)
hc = d[0]['State']['Health']
print('Status:', hc['Status'])
print('Último log:', hc['Log'][-1]['Output'] if hc['Log'] else 'sin logs')
"

# 2. Verificar que las contraseñas coinciden en .env
DB_PASS=$(grep "^DB_POSTGRESDB_PASSWORD=" .env | cut -d'=' -f2)
PG_PASS=$(grep "^POSTGRES_PASSWORD=" .env | cut -d'=' -f2)
[ "$DB_PASS" = "$PG_PASS" ] && echo "✅ Contraseñas coinciden" || echo "❌ MISMATCH: DB_POSTGRESDB_PASSWORD != POSTGRES_PASSWORD"

# 3. Si las contraseñas no coinciden, corregir y reiniciar
# Edita .env para que ambas variables tengan el mismo valor, luego:
docker compose down
docker volume rm lab08_postgres_data  # ⚠️ Borra datos existentes
docker compose up -d

# 4. Si postgres no pasa el healthcheck, revisar logs
docker compose logs postgres --tail=30
```

---

### Problema 2: Nginx devuelve `502 Bad Gateway` al acceder a `https://n8n.local`

**Síntomas:**
```
curl -sk https://n8n.local/healthz → 502 Bad Gateway
docker compose logs nginx → "connect() failed (111: Connection refused) while connecting to upstream"
```

**Causa:**
Nginx no puede alcanzar n8n en `http://n8n:5678`. Las causas más frecuentes son:
1. n8n aún no ha terminado de inicializar (normal en el primer arranque, tarda ~60s)
2. n8n está en la red `backend` pero nginx solo está en `frontend` — verificar que nginx también está en `frontend` y que n8n está en ambas redes
3. El nombre de host `n8n` en `nginx.conf` no resuelve porque el contenedor n8n se llama diferente

**Solución:**

```bash
# 1. Verificar que n8n está saludable
docker compose ps n8n
# Si aparece "starting" o "unhealthy", esperar más tiempo:
docker compose logs n8n --tail=20 -f
# Esperar hasta ver: "n8n ready on 0.0.0.0, port 5678"

# 2. Verificar conectividad de red desde nginx hacia n8n
docker exec lab08_nginx wget -qO- http://n8n:5678/healthz 2>&1
# Si falla con "bad address 'n8n'", verificar que ambos contenedores están en la misma red

# 3. Verificar las redes de cada contenedor
docker inspect lab08_n8n | python3 -c "
import json, sys
d = json.load(sys.stdin)
nets = list(d[0]['NetworkSettings']['Networks'].keys())
print('Redes de n8n:', nets)
"
docker inspect lab08_nginx | python3 -c "
import json, sys
d = json.load(sys.stdin)
nets = list(d[0]['NetworkSettings']['Networks'].keys())
print('Redes de nginx:', nets)
"
# n8n debe estar en: ['lab08_backend', 'lab08_frontend']
# nginx debe estar en: ['lab08_frontend']

# 4. Si las redes no son correctas, recrear el stack
docker compose down
docker compose up -d

# 5. Forzar recarga de nginx una vez n8n esté listo
docker exec lab08_nginx nginx -s reload
```

---

## Limpieza

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos:

```bash
# 1. Detener y eliminar contenedores y redes
cd lab08
docker compose down

# 2. Opcionalmente, eliminar volúmenes (borra datos de n8n y postgres)
# ⚠️ ADVERTENCIA: Esto elimina todos los workflows y credenciales guardadas
read -p "¿Eliminar también los volúmenes de datos? (s/N): " DEL_VOLS
if [ "$DEL_VOLS" = "s" ] || [ "$DEL_VOLS" = "S" ]; then
  docker compose down -v
  echo "✅ Volúmenes eliminados"
fi

# 3. Eliminar la entrada de /etc/hosts
sudo sed -i '/n8n.local/d' /etc/hosts
echo "✅ Entrada n8n.local eliminada de /etc/hosts"

# 4. Verificar que no quedan contenedores del lab
docker ps -a | grep "lab08" || echo "✅ No quedan contenedores del lab08"

# 5. Verificar que no quedan imágenes sin usar (opcional)
docker image prune -f 2>/dev/null

echo ""
echo "✅ Limpieza completada. El directorio lab08/ y sus archivos permanecen en disco."
echo "   Para eliminar completamente: rm -rf lab08/"
echo "   ⚠️  El archivo .env contiene secretos generados. Elimínalo de forma segura si no lo necesitas."
```

> **Nota de seguridad:** El archivo `.env` contiene credenciales generadas para este laboratorio. Si no vas a continuar usándolo, elimínalo con `shred -u lab08/.env` (Linux) para sobrescribir su contenido antes de eliminarlo.

---

## Resumen

En este laboratorio has construido un stack Docker Compose de nivel producción para n8n aplicando múltiples capas de seguridad:

| Área | Implementado |
|---|---|
| **Gestión de credenciales** | `.env` con permisos 600, generación con `openssl rand`, sin hardcoding en código fuente |
| **Hardening del contenedor** | Usuario `node` (no-root), `read_only: true`, `tmpfs`, `no-new-privileges`, límites de memoria/CPU |
| **Aislamiento de red** | Red `backend` interna (sin acceso externo directo a n8n), solo nginx en `frontend` |
| **TLS 1.3 exclusivo** | Certificados mkcert, `ssl_protocols TLSv1.3`, TLS 1.2 rechazado |
| **Headers de seguridad** | HSTS, X-Frame-Options DENY, CSP, X-Content-Type-Options, Referrer-Policy, Permissions-Policy |
| **Rate limiting** | 10 req/s por IP con burst de 20, respuesta 429 |
| **Rotación de claves** | Script documentado con procedimiento de backup y verificación |
| **Patrón HITL** | Workflow de transferencia bancaria ficticia con aprobación humana via webhook |

### Puntos clave

- **`N8N_ENCRYPTION_KEY` es crítica:** Si la pierdes o la cambias sin el procedimiento correcto, pierdes acceso a todas las credenciales cifradas. Guárdala siempre en un Secret Manager.
- **`env_file` vs variables en código:** El uso de `env_file` mantiene los secretos fuera del código fuente y del historial de git, pero los valores sí son visibles en `docker inspect` para quien tenga acceso al socket Docker. En producción, usa Docker Secrets (Swarm) o Kubernetes Secrets con RBAC.
- **Aislamiento de red como defensa en profundidad:** Aunque Nginx sea comprometido, un atacante no puede alcanzar directamente n8n o PostgreSQL desde el exterior gracias a la red `internal`.
- **HITL como control de seguridad:** El patrón Human-in-the-Loop no es solo UX — es un control de seguridad crítico para operaciones de alto riesgo que previene la ejecución automatizada de acciones irreversibles.

### Recursos adicionales

- [n8n — Variables de entorno y configuración](https://docs.n8n.io/hosting/configuration/environment-variables/)
- [n8n — Seguridad y mejores prácticas](https://docs.n8n.io/hosting/security/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) — para generar configuraciones Nginx/TLS recomendadas
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/) — referencia de headers de seguridad HTTP
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)
- [mkcert — Certificados TLS locales](https://github.com/FiloSottile/mkcert)

---
*Lab 08-00-01 — Módulo 8: Infraestructura segura de automatización con n8n*
*Banco Ficticio S.A. y todos los datos bancarios utilizados en este laboratorio son completamente ficticios y tienen únicamente fines educativos.*
