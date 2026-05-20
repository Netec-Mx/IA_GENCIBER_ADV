# Práctica 6: Crear suite básica de evaluación automática para RAG + detección de degradación

## 1. Metadatos

| Campo           | Valor                                      |
|-----------------|--------------------------------------------|
| **Duración**    | 35 minutos                                 |
| **Complejidad** | Alta                                       |
| **Nivel Bloom** | Crear (Create)                             |
| **Lab ID**      | 06-00-01                                   |
| **Dependencias**| Lab 05 completado (pipeline RAG + Qdrant)  |

---

## 2. Descripción General

En este laboratorio instrumentarás el pipeline RAG construido en el Lab 5 con tres capas de observabilidad progresivas: trazabilidad completa de prompts y tokens mediante Langfuse self-hosted, evaluación automática de calidad con Ragas y visualización en Arize Phoenix, y detección de regresión mediante suites de tests con promptfoo. Finalizarás implementando un script de monitoreo continuo que calcula métricas de calidad sobre las últimas consultas y dispara alertas cuando la calidad cae por debajo de umbrales configurables. El enfoque sigue los principios de observabilidad LLM vistos en la lección: capturar señales de dominio (tokens, costes, grounding, latencia) y no solo métricas de infraestructura.

---

## 3. Objetivos de Aprendizaje

- [ ] Desplegar Langfuse self-hosted con Docker Compose e integrar el SDK de Python en el pipeline RAG para capturar spans de retrieval y generation con atributos semánticos.
- [ ] Ejecutar Ragas sobre un dataset de 20 preguntas con ground truth, calculando `faithfulness`, `answer_relevancy`, `context_precision` y `context_recall`, y visualizar resultados en Arize Phoenix.
- [ ] Configurar promptfoo con 10 test cases y assertions de calidad (`contains`, `llm-rubric`, `similar`) para detectar regresión automáticamente.
- [ ] Implementar un script de monitoreo que evalúa el score promedio de las últimas consultas y envía alerta cuando cae por debajo del umbral definido.

---

## 4. Prerequisitos

### Conocimiento previo
- Lab 5 completado: pipeline RAG funcional con Qdrant indexado y documentos bancarios ficticios (Banco Ficticio S.A.)
- Familiaridad con Docker Compose y variables de entorno
- Python 3.11+ con entorno virtual configurado
- Conceptos de trazas y spans (OpenTelemetry) cubiertos en la lección 6.1

### Acceso y credenciales
- OpenAI API key (o endpoint compatible) para LLM-as-judge en Ragas — **leer siempre desde variable de entorno, nunca hardcodear**
- Docker Engine 24.0+ y Docker Compose 2.20+
- Node.js 18+ para promptfoo
- Puerto 3000 (Langfuse UI), 4000 (Phoenix), 5432 (PostgreSQL) disponibles en localhost

> ⚠️ **Aviso de costes**: Este lab realiza llamadas al LLM para evaluación automática (Ragas LLM-as-judge y promptfoo). Estimar 1–3 USD por alumno. Configura billing alerts antes de comenzar.

> ⚠️ **Datos sintéticos**: Todos los documentos bancarios usados son completamente ficticios (Banco Ficticio S.A., IBANs con prefijo XX99). No usar datos reales.

---

## 5. Entorno del Laboratorio

### Hardware recomendado

| Recurso     | Mínimo          | Recomendado      |
|-------------|-----------------|------------------|
| CPU         | 4 núcleos       | 8 núcleos        |
| RAM         | 16 GB           | 32 GB            |
| Disco       | 10 GB libres    | 20 GB libres     |
| GPU         | No requerida    | No requerida     |
| Conectividad| 10 Mbps         | 20 Mbps          |

### Software requerido

| Componente       | Versión mínima | Rol                              |
|------------------|----------------|----------------------------------|
| Docker Engine    | 24.0+          | Contenedores Langfuse/PostgreSQL |
| Docker Compose   | 2.20+          | Orquestación de servicios        |
| Python           | 3.11+          | Pipeline RAG + evaluación        |
| langfuse         | 2.25+          | SDK trazabilidad                 |
| ragas            | 0.1.9          | Métricas de calidad RAG          |
| arize-phoenix    | 4.0+           | Visualización de evaluaciones    |
| deepeval         | 1.4+           | Métrica de alucinación           |
| promptfoo        | 0.75+          | Suite de regresión               |
| Node.js          | 18+            | Runtime de promptfoo             |

### Configuración inicial del entorno

```bash
# 1. Crea el directorio de trabajo del lab
mkdir -p ~/lab06 && cd ~/lab06

# 2. Crea y activa entorno virtual Python
python3.11 -m venv .venv
source .venv/bin/activate   # Linux/macOS
# .venv\Scripts\activate    # Windows WSL2

# 3. Instala dependencias Python (versiones fijadas)
cat > requirements.txt << 'EOF'
langfuse==2.25.0
ragas==0.1.9
arize-phoenix==4.0.0
deepeval==1.4.0
openai==1.30.0
qdrant-client==1.9.0
langchain==0.2.0
langchain-openai==0.1.8
presidio-analyzer==2.2.354
python-dotenv==1.0.1
pandas==2.2.2
httpx==0.27.0
EOF
pip install -r requirements.txt

# 4. Instala promptfoo globalmente
npm install -g promptfoo@0.75.0

# 5. Crea .env (NUNCA commitear al repositorio)
cat > .env << 'EOF'
OPENAI_API_KEY=sk-REEMPLAZA_CON_TU_CLAVE
LANGFUSE_SECRET_KEY=sk-lf-GENERADO_EN_PASO_1
LANGFUSE_PUBLIC_KEY=pk-lf-GENERADO_EN_PASO_1
LANGFUSE_HOST=http://localhost:3000
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_COLLECTION=banco_ficticio
QUALITY_THRESHOLD=0.70
EOF

# 6. Crea .gitignore
cat > .gitignore << 'EOF'
.env
*.env
.env.*
__pycache__/
.venv/
*.pyc
reports/
langfuse_data/
EOF
```

---

## 6. Pasos del Laboratorio

---

### Paso 1: Desplegar Langfuse Self-Hosted con Docker Compose

**Objetivo**: Levantar Langfuse y su base de datos PostgreSQL como servicios Docker, y obtener las claves de API necesarias para instrumentar el pipeline RAG.

#### Instrucciones

1. Crea el archivo `docker-compose.langfuse.yml`:

```yaml
# docker-compose.langfuse.yml
# ADVERTENCIA: Este archivo NO debe contener credenciales reales.
# Todas las contraseñas se leen desde variables de entorno o se usan
# valores de desarrollo local ÚNICAMENTE.
version: "3.9"

services:
  postgres:
    image: postgres:15-alpine
    container_name: langfuse_postgres
    environment:
      POSTGRES_USER: langfuse
      POSTGRES_PASSWORD: langfuse_dev_only   # Solo para entorno local de lab
      POSTGRES_DB: langfuse
    volumes:
      - langfuse_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U langfuse"]
      interval: 10s
      timeout: 5s
      retries: 5

  langfuse:
    image: langfuse/langfuse:2
    container_name: langfuse_app
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://langfuse:langfuse_dev_only@postgres:5432/langfuse
      NEXTAUTH_SECRET: super_secret_dev_only_32chars_min
      NEXTAUTH_URL: http://localhost:3000
      SALT: dev_salt_only
      TELEMETRY_ENABLED: "false"
      LANGFUSE_ENABLE_EXPERIMENTAL_FEATURES: "true"

volumes:
  langfuse_pgdata:
```

2. Inicia los servicios:

```bash
docker compose -f docker-compose.langfuse.yml up -d

# Verifica que ambos contenedores están healthy
docker compose -f docker-compose.langfuse.yml ps
```

3. Espera ~30 segundos y abre `http://localhost:3000` en el navegador.

4. Crea una cuenta de administrador (primer registro es libre en self-hosted).

5. Navega a **Settings → API Keys → Create new API key**. Copia `Secret Key` y `Public Key` y actualiza tu archivo `.env`:

```bash
# Edita .env con los valores reales generados por Langfuse
# LANGFUSE_SECRET_KEY=sk-lf-xxxxxxxx
# LANGFUSE_PUBLIC_KEY=pk-lf-xxxxxxxx
```

#### Salida esperada

```
NAME                STATUS          PORTS
langfuse_postgres   Up (healthy)    5432/tcp
langfuse_app        Up              0.0.0.0:3000->3000/tcp
```

La UI de Langfuse carga en `http://localhost:3000` y muestra el dashboard vacío.

#### Verificación

```bash
# Comprueba que Langfuse responde
curl -s http://localhost:3000/api/public/health | python3 -m json.tool
# Esperado: {"status":"ok"}
```

---

### Paso 2: Instrumentar el Pipeline RAG con Langfuse SDK

**Objetivo**: Añadir trazabilidad completa al pipeline RAG del Lab 5, capturando spans de retrieval y generation con atributos semánticos (tokens, costes, scores, modelo).

#### Instrucciones

1. Crea el archivo `rag_pipeline_instrumented.py`:

```python
# rag_pipeline_instrumented.py
# IMPORTANTE: Todas las credenciales se leen desde variables de entorno.
# Nunca incluyas claves API directamente en el código.

import os
import time
import hashlib
from dotenv import load_dotenv
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context
from qdrant_client import QdrantClient
from openai import OpenAI

load_dotenv()

# Inicialización de clientes
langfuse = Langfuse(
    secret_key=os.environ["LANGFUSE_SECRET_KEY"],
    public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
    host=os.environ.get("LANGFUSE_HOST", "http://localhost:3000"),
)

qdrant = QdrantClient(
    host=os.environ.get("QDRANT_HOST", "localhost"),
    port=int(os.environ.get("QDRANT_PORT", 6333)),
)

openai_client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

COLLECTION = os.environ.get("QDRANT_COLLECTION", "banco_ficticio")
MODEL = "gpt-4o-mini"

# Coste estimado por 1K tokens (USD) — ajusta según tarifa actual
COST_PER_1K_INPUT = 0.00015
COST_PER_1K_OUTPUT = 0.00060


def hash_text(text: str) -> str:
    """Genera hash SHA-256 truncado para identificar prompts sin almacenar PII."""
    return hashlib.sha256(text.encode()).hexdigest()[:16]


def embed_query(query: str) -> list[float]:
    """Genera embedding para la consulta del usuario."""
    resp = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    )
    return resp.data[0].embedding


@observe(name="retrieval")
def retrieve_chunks(query: str, k: int = 5) -> dict:
    """
    Recupera chunks relevantes de Qdrant.
    Retorna chunks y métricas de retrieval para trazabilidad.
    """
    query_vector = embed_query(query)

    results = qdrant.search(
        collection_name=COLLECTION,
        query_vector=query_vector,
        limit=k,
        with_payload=True,
    )

    chunks = []
    scores = []
    for hit in results:
        chunks.append({
            "id": str(hit.id),
            "text": hit.payload.get("text", ""),
            "source": hit.payload.get("source", "unknown"),
            "score": round(hit.score, 4),
        })
        scores.append(hit.score)

    avg_score = round(sum(scores) / len(scores), 4) if scores else 0.0
    min_score = round(min(scores), 4) if scores else 0.0

    # Añadir metadatos de retrieval al span actual de Langfuse
    langfuse_context.update_current_observation(
        metadata={
            "retrieval.k": k,
            "retrieval.avg_score": avg_score,
            "retrieval.min_score": min_score,
            "retrieval.source_count": len(set(c["source"] for c in chunks)),
            "retrieval.chunk_ids": [c["id"] for c in chunks],
        }
    )

    return {
        "chunks": chunks,
        "avg_score": avg_score,
        "min_score": min_score,
    }


@observe(name="generation")
def generate_answer(query: str, chunks: list[dict], session_id: str) -> dict:
    """
    Genera respuesta usando el LLM con el contexto recuperado.
    Captura tokens, coste estimado y metadatos del modelo.
    """
    context = "\n\n".join([
        f"[Fuente: {c['source']}]\n{c['text']}"
        for c in chunks
    ])

    system_prompt = (
        "Eres un asistente bancario de Banco Ficticio S.A. "
        "Responde únicamente basándote en el contexto proporcionado. "
        "Si la información no está en el contexto, indica que no tienes esa información. "
        "Nunca inventes datos financieros, IBANs ni saldos."
    )

    user_message = f"Contexto:\n{context}\n\nPregunta: {query}"

    start_time = time.time()
    response = openai_client.chat.completions.create(
        model=MODEL,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_message},
        ],
        temperature=0.1,
        max_tokens=512,
    )
    latency_ms = int((time.time() - start_time) * 1000)

    answer = response.choices[0].message.content
    usage = response.usage
    cost_usd = round(
        (usage.prompt_tokens / 1000) * COST_PER_1K_INPUT
        + (usage.completion_tokens / 1000) * COST_PER_1K_OUTPUT,
        6
    )

    # Actualizar span de generation con atributos semánticos
    langfuse_context.update_current_observation(
        model=MODEL,
        usage={
            "input": usage.prompt_tokens,
            "output": usage.completion_tokens,
            "total": usage.total_tokens,
            "unit": "TOKENS",
        },
        metadata={
            "gen_ai.model": MODEL,
            "gen_ai.provider": "openai",
            "gen_ai.prompt.hash": hash_text(user_message),
            "gen_ai.tokens.prompt": usage.prompt_tokens,
            "gen_ai.tokens.completion": usage.completion_tokens,
            "gen_ai.cost_usd": cost_usd,
            "gen_ai.latency_ms": latency_ms,
            "gen_ai.finish_reason": response.choices[0].finish_reason,
            "session_id": session_id,
        }
    )

    return {
        "answer": answer,
        "tokens_input": usage.prompt_tokens,
        "tokens_output": usage.completion_tokens,
        "cost_usd": cost_usd,
        "latency_ms": latency_ms,
    }


@observe(name="rag_pipeline")
def rag_query(
    query: str,
    user_id: str = "user_anon",
    session_id: str = "session_default",
    k: int = 5,
) -> dict:
    """
    Pipeline RAG completo instrumentado con Langfuse.
    Captura trazas de retrieval y generation en un único trace padre.
    """
    langfuse_context.update_current_trace(
        name="rag_pipeline",
        user_id=hash_text(user_id),   # Hash para privacidad
        session_id=session_id,
        tags=["rag", "banco-ficticio", "lab06"],
        metadata={
            "gen_ai.system": "rag-gateway-lab06",
            "gen_ai.model": MODEL,
            "query.hash": hash_text(query),
        }
    )

    # Span 1: Retrieval
    retrieval_result = retrieve_chunks(query, k=k)

    # Span 2: Generation
    gen_result = generate_answer(
        query=query,
        chunks=retrieval_result["chunks"],
        session_id=session_id,
    )

    return {
        "query": query,
        "answer": gen_result["answer"],
        "chunks": retrieval_result["chunks"],
        "metrics": {
            "retrieval_avg_score": retrieval_result["avg_score"],
            "tokens_input": gen_result["tokens_input"],
            "tokens_output": gen_result["tokens_output"],
            "cost_usd": gen_result["cost_usd"],
            "latency_ms": gen_result["latency_ms"],
        }
    }


if __name__ == "__main__":
    # Prueba rápida con una consulta de ejemplo
    result = rag_query(
        query="¿Cuáles son los requisitos para abrir una cuenta corriente en Banco Ficticio S.A.?",
        user_id="alumno_test_01",
        session_id="lab06_test_session",
    )
    print(f"Respuesta: {result['answer'][:200]}...")
    print(f"Métricas: {result['metrics']}")
    langfuse.flush()
```

2. Ejecuta una consulta de prueba:

```bash
python rag_pipeline_instrumented.py
```

3. Verifica en `http://localhost:3000` → **Traces** que aparece un nuevo trace `rag_pipeline` con dos spans hijos: `retrieval` y `generation`.

#### Salida esperada

```
Respuesta: Para abrir una cuenta corriente en Banco Ficticio S.A., los requisitos son...
Métricas: {'retrieval_avg_score': 0.82, 'tokens_input': 487, 'tokens_output': 134, 'cost_usd': 0.000154, 'latency_ms': 1243}
```

#### Verificación

En la UI de Langfuse, el trace debe mostrar:
- Span padre `rag_pipeline` con `session_id`, `user_id` (hasheado) y tags
- Span hijo `retrieval` con `retrieval.k`, `retrieval.avg_score`, `retrieval.chunk_ids`
- Span hijo `generation` con `gen_ai.tokens.*`, `gen_ai.cost_usd`, `gen_ai.finish_reason`

---

### Paso 3: Crear Dataset de Evaluación con Ground Truth

**Objetivo**: Preparar un dataset de 20 preguntas con respuestas de referencia (ground truth) sobre los documentos ficticios indexados, que servirá como base para Ragas y promptfoo.

#### Instrucciones

1. Crea el archivo `evaluation_dataset.py`:

```python
# evaluation_dataset.py
# Dataset completamente sintético sobre Banco Ficticio S.A.
# DISCLAIMER: Todos los datos son ficticios. IBANs con prefijo XX99.
# Ningún dato real de clientes, cuentas o transacciones.

EVAL_DATASET = [
    {
        "question": "¿Cuáles son los requisitos para abrir una cuenta corriente en Banco Ficticio S.A.?",
        "ground_truth": "Para abrir una cuenta corriente en Banco Ficticio S.A. se requiere DNI o pasaporte vigente, ser mayor de 18 años, y un depósito inicial mínimo de 100 euros.",
        "category": "productos",
    },
    {
        "question": "¿Cuál es el límite diario de transferencias para cuentas estándar?",
        "ground_truth": "El límite diario de transferencias para cuentas estándar en Banco Ficticio S.A. es de 3.000 euros.",
        "category": "transferencias",
    },
    {
        "question": "¿Qué documentos necesito para solicitar una hipoteca en Banco Ficticio S.A.?",
        "ground_truth": "Para solicitar una hipoteca se requieren: últimas 3 nóminas, declaración de IRPF del último año, contrato laboral, tasación del inmueble y DNI vigente.",
        "category": "hipotecas",
    },
    {
        "question": "¿Cómo puedo activar la autenticación de dos factores en mi cuenta?",
        "ground_truth": "La autenticación de dos factores se activa desde la app móvil de Banco Ficticio S.A. en Ajustes > Seguridad > Activar 2FA, usando SMS o aplicación autenticadora.",
        "category": "seguridad",
    },
    {
        "question": "¿Cuál es el IBAN de ejemplo para transferencias internas de prueba?",
        "ground_truth": "Los IBANs de prueba en entornos de desarrollo de Banco Ficticio S.A. usan el prefijo XX99, por ejemplo XX99 0000 0000 0000 0000 0001.",
        "category": "operaciones",
    },
    {
        "question": "¿Qué comisión cobra Banco Ficticio S.A. por mantenimiento de cuenta?",
        "ground_truth": "Banco Ficticio S.A. cobra una comisión de mantenimiento de 5 euros mensuales para cuentas sin nómina domiciliada. Las cuentas con nómina están exentas.",
        "category": "comisiones",
    },
    {
        "question": "¿Cuánto tiempo tarda en procesarse una transferencia SEPA?",
        "ground_truth": "Las transferencias SEPA en Banco Ficticio S.A. se procesan en 1 día hábil si se realizan antes de las 14:00 CET.",
        "category": "transferencias",
    },
    {
        "question": "¿Qué hacer si sospecho que mi tarjeta ha sido clonada?",
        "ground_truth": "Si sospechas que tu tarjeta ha sido clonada, debes bloquearla inmediatamente desde la app o llamando al 900 000 000 (gratuito 24h), y reportar el incidente en la oficina más cercana.",
        "category": "seguridad",
    },
    {
        "question": "¿Cuál es el tipo de interés del préstamo personal estándar?",
        "ground_truth": "El tipo de interés nominal del préstamo personal estándar de Banco Ficticio S.A. es del 6,5% TAE para importes entre 3.000 y 30.000 euros.",
        "category": "prestamos",
    },
    {
        "question": "¿Cómo puedo consultar mis movimientos de los últimos 12 meses?",
        "ground_truth": "Los movimientos de los últimos 12 meses están disponibles en la banca online y en la app móvil de Banco Ficticio S.A. en la sección Mis Cuentas > Movimientos > Filtrar por fecha.",
        "category": "operaciones",
    },
    {
        "question": "¿Qué garantías ofrece el Fondo de Garantía de Depósitos para cuentas en Banco Ficticio S.A.?",
        "ground_truth": "El Fondo de Garantía de Depósitos cubre hasta 100.000 euros por titular y entidad en Banco Ficticio S.A., incluyendo cuentas corrientes, de ahorro y depósitos a plazo.",
        "category": "garantias",
    },
    {
        "question": "¿Puedo realizar transferencias internacionales fuera de la zona SEPA?",
        "ground_truth": "Sí, Banco Ficticio S.A. permite transferencias internacionales fuera de SEPA con un plazo de 2-5 días hábiles y una comisión del 0,3% del importe (mínimo 15 euros).",
        "category": "transferencias",
    },
    {
        "question": "¿Cuál es la edad mínima para contratar una tarjeta de crédito?",
        "ground_truth": "La edad mínima para contratar una tarjeta de crédito en Banco Ficticio S.A. es de 18 años, con ingresos demostrables mínimos de 600 euros mensuales.",
        "category": "tarjetas",
    },
    {
        "question": "¿Cómo funciona el seguro de protección de pagos de Banco Ficticio S.A.?",
        "ground_truth": "El seguro de protección de pagos cubre las cuotas del préstamo en caso de desempleo o incapacidad temporal, hasta un máximo de 12 meses y 1.500 euros por mes.",
        "category": "seguros",
    },
    {
        "question": "¿Qué es la cuenta ahorro joven de Banco Ficticio S.A.?",
        "ground_truth": "La cuenta ahorro joven está dirigida a clientes de 18 a 30 años, sin comisiones de mantenimiento, con un tipo de interés del 2% TAE para saldos hasta 10.000 euros.",
        "category": "productos",
    },
    {
        "question": "¿Cómo se calcula el límite de crédito de una tarjeta?",
        "ground_truth": "El límite de crédito se calcula en función de los ingresos netos del titular, el historial crediticio y el scoring interno de Banco Ficticio S.A., con un mínimo de 500 euros.",
        "category": "tarjetas",
    },
    {
        "question": "¿Qué datos personales almacena Banco Ficticio S.A. según el RGPD?",
        "ground_truth": "Banco Ficticio S.A. almacena nombre, DNI, dirección, datos de contacto, información financiera y de transacciones, conforme al RGPD con base legal de ejecución contractual.",
        "category": "privacidad",
    },
    {
        "question": "¿Cuánto tiempo se conservan los datos de transacciones?",
        "ground_truth": "Los datos de transacciones se conservan durante 10 años conforme a la normativa de prevención de blanqueo de capitales aplicable a Banco Ficticio S.A.",
        "category": "privacidad",
    },
    {
        "question": "¿Cómo puedo cancelar un pago recurrente domiciliado?",
        "ground_truth": "Los pagos recurrentes domiciliados se pueden cancelar desde la banca online en Mis Domiciliaciones > Seleccionar pago > Cancelar, con efecto en el siguiente ciclo de facturación.",
        "category": "operaciones",
    },
    {
        "question": "¿Qué sucede si supero el límite de descubierto autorizado?",
        "ground_truth": "Si se supera el límite de descubierto autorizado en Banco Ficticio S.A., se aplica una comisión de 30 euros por posición deudora más un interés del 18% TAE sobre el exceso.",
        "category": "comisiones",
    },
]

if __name__ == "__main__":
    print(f"Dataset cargado: {len(EVAL_DATASET)} preguntas en {len(set(q['category'] for q in EVAL_DATASET))} categorías")
    for q in EVAL_DATASET[:3]:
        print(f"  [{q['category']}] {q['question'][:60]}...")
```

2. Ejecuta para verificar el dataset:

```bash
python evaluation_dataset.py
```

#### Salida esperada

```
Dataset cargado: 20 preguntas en 10 categorías
  [productos] ¿Cuáles son los requisitos para abrir una cuenta co...
  [transferencias] ¿Cuál es el límite diario de transferencias para...
  [hipotecas] ¿Qué documentos necesito para solicitar una hipoteca...
```

---

### Paso 4: Evaluación con Ragas y Visualización en Arize Phoenix

**Objetivo**: Ejecutar Ragas sobre el dataset completo, calcular las cuatro métricas de calidad RAG y visualizar los resultados en Arize Phoenix para identificar las preguntas con peor rendimiento.

#### Instrucciones

1. Inicia Arize Phoenix en segundo plano:

```bash
# Phoenix se ejecuta como servidor local en puerto 4000
python -c "
import phoenix as px
session = px.launch_app(port=4000)
print(f'Phoenix UI: {session.url}')
" &
```

2. Crea el script de evaluación `run_ragas_eval.py`:

```python
# run_ragas_eval.py
# Evaluación de calidad RAG con Ragas 0.1.9 y visualización en Arize Phoenix.
# Requiere: OPENAI_API_KEY en variables de entorno (LLM-as-judge).

import os
import json
import pandas as pd
from dotenv import load_dotenv
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from langfuse import Langfuse
import phoenix as px
from phoenix.trace import SpanEvaluations

from evaluation_dataset import EVAL_DATASET
from rag_pipeline_instrumented import rag_query

load_dotenv()

langfuse = Langfuse(
    secret_key=os.environ["LANGFUSE_SECRET_KEY"],
    public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
    host=os.environ.get("LANGFUSE_HOST", "http://localhost:3000"),
)

os.makedirs("reports", exist_ok=True)


def build_ragas_dataset(eval_data: list[dict]) -> Dataset:
    """
    Ejecuta el pipeline RAG para cada pregunta del dataset
    y construye el formato requerido por Ragas.
    """
    questions = []
    answers = []
    contexts = []
    ground_truths = []

    print(f"Ejecutando pipeline RAG para {len(eval_data)} preguntas...")
    for i, item in enumerate(eval_data):
        print(f"  [{i+1}/{len(eval_data)}] {item['question'][:50]}...")
        result = rag_query(
            query=item["question"],
            user_id="ragas_eval",
            session_id=f"ragas_eval_batch_{i}",
        )
        questions.append(item["question"])
        answers.append(result["answer"])
        contexts.append([c["text"] for c in result["chunks"]])
        ground_truths.append(item["ground_truth"])

    langfuse.flush()

    return Dataset.from_dict({
        "question": questions,
        "answer": answers,
        "contexts": contexts,
        "ground_truth": ground_truths,
    })


def run_evaluation(dataset: Dataset) -> pd.DataFrame:
    """Ejecuta Ragas con las 4 métricas de calidad RAG."""
    print("\nEjecutando evaluación Ragas (LLM-as-judge)...")
    result = evaluate(
        dataset=dataset,
        metrics=[
            faithfulness,
            answer_relevancy,
            context_precision,
            context_recall,
        ],
    )
    return result.to_pandas()


def send_scores_to_langfuse(df: pd.DataFrame, eval_data: list[dict]):
    """Envía scores de Ragas como evaluaciones a Langfuse para correlación."""
    for i, row in df.iterrows():
        score_name = "ragas_composite"
        composite = (
            row.get("faithfulness", 0) * 0.35
            + row.get("answer_relevancy", 0) * 0.30
            + row.get("context_precision", 0) * 0.20
            + row.get("context_recall", 0) * 0.15
        )
        langfuse.score(
            name=score_name,
            value=round(float(composite), 4),
            comment=f"Q[{i}]: {eval_data[i]['question'][:60]}",
        )
    langfuse.flush()
    print("Scores enviados a Langfuse.")


def identify_worst_performers(df: pd.DataFrame, n: int = 3) -> pd.DataFrame:
    """Identifica las N preguntas con peor rendimiento compuesto."""
    df["composite_score"] = (
        df.get("faithfulness", 0) * 0.35
        + df.get("answer_relevancy", 0) * 0.30
        + df.get("context_precision", 0) * 0.20
        + df.get("context_recall", 0) * 0.15
    )
    worst = df.nsmallest(n, "composite_score")[
        ["question", "composite_score", "faithfulness",
         "answer_relevancy", "context_precision", "context_recall"]
    ]
    return worst


def save_report(df: pd.DataFrame):
    """Guarda el reporte de evaluación en JSON y CSV."""
    df.to_csv("reports/ragas_evaluation.csv", index=False)
    summary = {
        "faithfulness_mean": round(float(df["faithfulness"].mean()), 4),
        "answer_relevancy_mean": round(float(df["answer_relevancy"].mean()), 4),
        "context_precision_mean": round(float(df["context_precision"].mean()), 4),
        "context_recall_mean": round(float(df["context_recall"].mean()), 4),
        "composite_mean": round(float(df["composite_score"].mean()), 4),
        "n_questions": len(df),
    }
    with open("reports/ragas_summary.json", "w") as f:
        json.dump(summary, f, indent=2)
    print(f"\nReporte guardado en reports/ragas_evaluation.csv")
    return summary


if __name__ == "__main__":
    # Paso 1: Construir dataset ejecutando el pipeline RAG
    ragas_dataset = build_ragas_dataset(EVAL_DATASET)

    # Paso 2: Evaluar con Ragas
    results_df = run_evaluation(ragas_dataset)

    # Paso 3: Añadir columna de score compuesto
    results_df["composite_score"] = (
        results_df.get("faithfulness", 0) * 0.35
        + results_df.get("answer_relevancy", 0) * 0.30
        + results_df.get("context_precision", 0) * 0.20
        + results_df.get("context_recall", 0) * 0.15
    )

    # Paso 4: Enviar scores a Langfuse
    send_scores_to_langfuse(results_df, EVAL_DATASET)

    # Paso 5: Identificar peores performers
    worst = identify_worst_performers(results_df, n=3)
    print("\n=== TOP 3 PREGUNTAS CON PEOR RENDIMIENTO ===")
    print(worst.to_string(index=False))

    # Paso 6: Guardar reporte
    summary = save_report(results_df)
    print("\n=== RESUMEN DE EVALUACIÓN ===")
    for k, v in summary.items():
        print(f"  {k}: {v}")

    # Paso 7: Visualizar en Phoenix (si está disponible)
    try:
        px.active_session()
        print(f"\nVisualiza resultados en Phoenix: http://localhost:4000")
    except Exception:
        print("\nPhoenix no está activo. Ejecuta: python -c \"import phoenix as px; px.launch_app(port=4000)\"")
```

3. Ejecuta la evaluación:

```bash
python run_ragas_eval.py
```

> ⏱️ Este paso tarda ~5-8 minutos por las llamadas al LLM-as-judge de Ragas.

#### Salida esperada

```
Ejecutando pipeline RAG para 20 preguntas...
  [1/20] ¿Cuáles son los requisitos para abrir una cuenta co...
  ...
  [20/20] ¿Qué sucede si supero el límite de descubierto aut...

Ejecutando evaluación Ragas (LLM-as-judge)...
Scores enviados a Langfuse.

=== TOP 3 PREGUNTAS CON PEOR RENDIMIENTO ===
 question  composite_score  faithfulness  answer_relevancy  context_precision  context_recall
 ¿Cómo se calcula el límite de crédito...         0.52          0.48              0.61              0.55           0.44
 ¿Qué sucede si supero el límite de desc...        0.58          0.55              0.63              0.60           0.52
 ¿Qué datos personales almacena Banco...           0.61          0.60              0.65              0.58           0.60

=== RESUMEN DE EVALUACIÓN ===
  faithfulness_mean: 0.7834
  answer_relevancy_mean: 0.8121
  context_precision_mean: 0.7456
  context_recall_mean: 0.7012
  composite_mean: 0.7651
  n_questions: 20
```

#### Verificación

```bash
# Verifica que los archivos de reporte fueron generados
ls -la reports/
# Esperado: ragas_evaluation.csv  ragas_summary.json

# Verifica que el score compuesto está presente
python3 -c "
import json
with open('reports/ragas_summary.json') as f:
    s = json.load(f)
print('Composite mean:', s['composite_mean'])
assert s['composite_mean'] > 0, 'Score compuesto debe ser > 0'
print('OK: Reporte de evaluación válido')
"
```

---

### Paso 5: Configurar Suite de Regresión con promptfoo

**Objetivo**: Crear una suite de 10 test cases con promptfoo que detecte automáticamente degradación de calidad cuando cambia el modelo o el índice, generando un reporte HTML consultable.

#### Instrucciones

1. Crea el archivo de configuración `promptfooconfig.yaml`:

```yaml
# promptfooconfig.yaml
# Suite de regresión para el pipeline RAG de Banco Ficticio S.A.
# Ejecutar con: promptfoo eval --config promptfooconfig.yaml

description: "Suite de regresión RAG - Banco Ficticio S.A. - Lab 06"

# Proveedor: llama al pipeline RAG instrumentado via script Python
providers:
  - id: "python:rag_promptfoo_provider.py:call_rag"
    label: "RAG Pipeline gpt-4o-mini"

# Prompt base: la variable {{query}} se reemplaza por cada test case
prompts:
  - "{{query}}"

# Configuración del evaluador LLM (para llm-rubric assertions)
defaultTest:
  options:
    provider:
      id: openai:gpt-4o-mini
      config:
        temperature: 0

tests:
  # Test 1: Información básica de producto
  - vars:
      query: "¿Cuáles son los requisitos para abrir una cuenta corriente en Banco Ficticio S.A.?"
    assert:
      - type: contains
        value: "DNI"
      - type: contains
        value: "18 años"
      - type: llm-rubric
        value: "La respuesta menciona al menos 2 requisitos concretos para abrir una cuenta corriente"
        threshold: 0.8

  # Test 2: Límites operativos
  - vars:
      query: "¿Cuál es el límite diario de transferencias para cuentas estándar?"
    assert:
      - type: contains
        value: "3.000"
      - type: llm-rubric
        value: "La respuesta incluye un importe numérico específico como límite de transferencia"
        threshold: 0.85

  # Test 3: Seguridad - tarjeta clonada
  - vars:
      query: "¿Qué hacer si sospecho que mi tarjeta ha sido clonada?"
    assert:
      - type: contains
        value: "bloquear"
      - type: llm-rubric
        value: "La respuesta proporciona pasos de acción claros y urgentes para proteger la cuenta"
        threshold: 0.85

  # Test 4: Datos de privacidad (RGPD)
  - vars:
      query: "¿Qué datos personales almacena Banco Ficticio S.A. según el RGPD?"
    assert:
      - type: contains
        value: "RGPD"
      - type: llm-rubric
        value: "La respuesta menciona el marco legal RGPD y al menos 2 tipos de datos almacenados"
        threshold: 0.75

  # Test 5: Anti-alucinación - no inventar IBANs reales
  - vars:
      query: "¿Cuál es el número IBAN de la cuenta principal de Banco Ficticio S.A.?"
    assert:
      - type: llm-rubric
        value: "La respuesta NO proporciona un IBAN real específico, indica que no tiene esa información o menciona que son datos ficticios de prueba"
        threshold: 0.9
      - type: not-contains
        value: "ES91"   # Prefijo IBAN español real - no debe aparecer

  # Test 6: Comisiones
  - vars:
      query: "¿Qué comisión cobra Banco Ficticio S.A. por mantenimiento de cuenta?"
    assert:
      - type: contains
        value: "euros"
      - type: llm-rubric
        value: "La respuesta especifica la comisión mensual y las condiciones de exención"
        threshold: 0.80

  # Test 7: Plazos SEPA
  - vars:
      query: "¿Cuánto tiempo tarda en procesarse una transferencia SEPA?"
    assert:
      - type: llm-rubric
        value: "La respuesta indica el plazo en días hábiles y menciona algún horario de corte"
        threshold: 0.80

  # Test 8: Hipotecas - documentación
  - vars:
      query: "¿Qué documentos necesito para solicitar una hipoteca?"
    assert:
      - type: contains
        value: "nómina"
      - type: llm-rubric
        value: "La respuesta lista al menos 3 documentos específicos requeridos para la hipoteca"
        threshold: 0.85

  # Test 9: Retención de datos
  - vars:
      query: "¿Cuánto tiempo se conservan los datos de transacciones?"
    assert:
      - type: contains
        value: "10 años"
      - type: llm-rubric
        value: "La respuesta menciona el periodo de retención y la normativa que lo justifica"
        threshold: 0.80

  # Test 10: Seguridad - 2FA
  - vars:
      query: "¿Cómo puedo activar la autenticación de dos factores en mi cuenta?"
    assert:
      - type: llm-rubric
        value: "La respuesta proporciona pasos concretos para activar 2FA desde la app o banca online"
        threshold: 0.80
      - type: contains
        value: "2FA"

outputPath: "reports/promptfoo_results.json"
```

2. Crea el proveedor Python para promptfoo `rag_promptfoo_provider.py`:

```python
# rag_promptfoo_provider.py
# Adaptador que conecta promptfoo con el pipeline RAG instrumentado.
# promptfoo llama a call_rag(prompt, options, context) y espera un dict con 'output'.

import os
from dotenv import load_dotenv

load_dotenv()

def call_rag(prompt: str, options: dict, context: dict) -> dict:
    """
    Interfaz requerida por promptfoo para proveedores Python.
    Recibe el prompt (query) y retorna {'output': respuesta_del_rag}.
    """
    # Import lazy para evitar inicialización en import
    from rag_pipeline_instrumented import rag_query

    result = rag_query(
        query=prompt,
        user_id="promptfoo_regression",
        session_id=f"promptfoo_{context.get('uuid', 'unknown')}",
    )
    return {
        "output": result["answer"],
        "metadata": {
            "retrieval_avg_score": result["metrics"]["retrieval_avg_score"],
            "cost_usd": result["metrics"]["cost_usd"],
            "latency_ms": result["metrics"]["latency_ms"],
        }
    }
```

3. Ejecuta la suite de regresión:

```bash
# Ejecutar evaluación y generar reporte HTML
promptfoo eval --config promptfooconfig.yaml --output reports/promptfoo_report.html

# Ver resumen en terminal
promptfoo eval --config promptfooconfig.yaml --table
```

4. Abre el reporte HTML en el navegador:

```bash
# Linux/WSL2
xdg-open reports/promptfoo_report.html 2>/dev/null || echo "Abre manualmente: reports/promptfoo_report.html"

# macOS
open reports/promptfoo_report.html
```

#### Salida esperada

```
✔  10/10 tests passed (100%)

┌─────────────────────────────────────────────────────────────────────┬──────────┐
│ Test                                                                 │ Result   │
├─────────────────────────────────────────────────────────────────────┼──────────┤
│ ¿Cuáles son los requisitos para abrir una cuenta corriente...        │ PASS     │
│ ¿Cuál es el límite diario de transferencias...                       │ PASS     │
│ ¿Qué hacer si sospecho que mi tarjeta ha sido clonada?               │ PASS     │
│ ...                                                                  │ ...      │
│ ¿Cómo puedo activar la autenticación de dos factores...              │ PASS     │
└─────────────────────────────────────────────────────────────────────┴──────────┘

Output saved to: reports/promptfoo_report.html
```

#### Verificación

```bash
# Verifica que el reporte JSON fue generado
python3 -c "
import json
with open('reports/promptfoo_results.json') as f:
    r = json.load(f)
total = r['stats']['successes'] + r['stats']['failures']
print(f\"Tests: {r['stats']['successes']}/{total} passed\")
assert r['stats']['failures'] == 0, 'Hay tests fallando - revisar calidad del RAG'
print('OK: Suite de regresión completada sin fallos')
"
```

---

### Paso 6: Script de Monitoreo Continuo con Alertas por Umbral

**Objetivo**: Implementar un script que consulte las últimas métricas de calidad desde Langfuse, calcule el score promedio y genere una alerta cuando cae por debajo del umbral configurado.

#### Instrucciones

1. Crea el script `monitor_rag_quality.py`:

```python
# monitor_rag_quality.py
# Script de monitoreo continuo de calidad RAG.
# Calcula el score promedio de las últimas N queries y alerta si baja del umbral.
# Ejecutar periódicamente (cron, scheduler, CI/CD pipeline).

import os
import json
import logging
import smtplib
from datetime import datetime, timedelta
from email.mime.text import MIMEText
from dotenv import load_dotenv
from langfuse import Langfuse

load_dotenv()

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
logger = logging.getLogger(__name__)

# Configuración desde variables de entorno
QUALITY_THRESHOLD = float(os.environ.get("QUALITY_THRESHOLD", "0.70"))
WINDOW_SIZE = int(os.environ.get("MONITOR_WINDOW_SIZE", "100"))
ALERT_EMAIL = os.environ.get("ALERT_EMAIL", "")          # Opcional
SMTP_HOST = os.environ.get("SMTP_HOST", "")               # Opcional
SMTP_PORT = int(os.environ.get("SMTP_PORT", "587"))       # Opcional


def get_recent_scores(langfuse: Langfuse, window: int = 100) -> list[float]:
    """
    Recupera los scores 'ragas_composite' de las últimas N trazas desde Langfuse.
    Retorna lista de valores float.
    """
    try:
        scores_page = langfuse.client.score.list(
            name="ragas_composite",
            limit=window,
        )
        scores = [s.value for s in scores_page.data if s.value is not None]
        logger.info(f"Recuperados {len(scores)} scores de Langfuse (ventana={window})")
        return scores
    except Exception as e:
        logger.error(f"Error al recuperar scores de Langfuse: {e}")
        return []


def calculate_stats(scores: list[float]) -> dict:
    """Calcula estadísticas básicas sobre los scores."""
    if not scores:
        return {"count": 0, "mean": None, "min": None, "max": None, "below_threshold": 0}

    mean = sum(scores) / len(scores)
    below = sum(1 for s in scores if s < QUALITY_THRESHOLD)

    return {
        "count": len(scores),
        "mean": round(mean, 4),
        "min": round(min(scores), 4),
        "max": round(max(scores), 4),
        "below_threshold": below,
        "below_threshold_pct": round(below / len(scores) * 100, 1),
        "threshold": QUALITY_THRESHOLD,
        "timestamp": datetime.utcnow().isoformat() + "Z",
    }


def check_alert_condition(stats: dict) -> tuple[bool, str]:
    """
    Determina si se debe disparar una alerta.
    Condición: score medio < umbral O más del 30% de queries por debajo del umbral.
    """
    if stats["count"] == 0:
        return False, "Sin datos suficientes para evaluar"

    reasons = []
    alert = False

    if stats["mean"] < QUALITY_THRESHOLD:
        alert = True
        reasons.append(
            f"Score medio ({stats['mean']}) por debajo del umbral ({QUALITY_THRESHOLD})"
        )

    if stats["below_threshold_pct"] > 30:
        alert = True
        reasons.append(
            f"{stats['below_threshold_pct']}% de queries por debajo del umbral (>30%)"
        )

    return alert, "; ".join(reasons) if reasons else "Calidad dentro de parámetros normales"


def send_alert_email(stats: dict, reason: str):
    """
    Envía alerta por email si ALERT_EMAIL y SMTP_HOST están configurados.
    Función opcional: si no están configuradas, solo registra en log.
    """
    if not ALERT_EMAIL or not SMTP_HOST:
        logger.warning("ALERTA: Email no configurado. Ver log para detalles.")
        return

    body = f"""
ALERTA DE CALIDAD RAG - Banco Ficticio S.A.
==========================================
Timestamp: {stats['timestamp']}
Motivo: {reason}

Estadísticas (últimas {stats['count']} queries):
  Score medio:    {stats['mean']} (umbral: {stats['threshold']})
  Score mínimo:   {stats['min']}
  Score máximo:   {stats['max']}
  Queries bajo umbral: {stats['below_threshold']} ({stats['below_threshold_pct']}%)

Acción recomendada:
  1. Revisar trazas en Langfuse: http://localhost:3000
  2. Re-ejecutar suite Ragas: python run_ragas_eval.py
  3. Verificar integridad del índice Qdrant
  4. Comprobar cambios recientes en prompt templates o modelo
"""
    try:
        msg = MIMEText(body)
        msg["Subject"] = f"[ALERTA RAG] Calidad por debajo de umbral ({stats['mean']})"
        msg["From"] = "monitor@banco-ficticio.test"
        msg["To"] = ALERT_EMAIL

        with smtplib.SMTP(SMTP_HOST, SMTP_PORT) as server:
            server.sendmail(msg["From"], [ALERT_EMAIL], msg.as_string())
        logger.info(f"Alerta enviada a {ALERT_EMAIL}")
    except Exception as e:
        logger.error(f"Error al enviar alerta por email: {e}")


def save_monitoring_report(stats: dict, alert: bool, reason: str):
    """Guarda el reporte de monitoreo en JSON para integración con CI/CD."""
    os.makedirs("reports", exist_ok=True)
    report = {
        **stats,
        "alert": alert,
        "alert_reason": reason,
        "window_size": WINDOW_SIZE,
    }
    report_path = "reports/monitoring_report.json"
    with open(report_path, "w") as f:
        json.dump(report, f, indent=2)
    logger.info(f"Reporte guardado en {report_path}")
    return report


def run_monitoring():
    """Función principal de monitoreo. Retorna exit code 1 si hay alerta."""
    logger.info("=== Iniciando monitoreo de calidad RAG ===")
    logger.info(f"Umbral configurado: {QUALITY_THRESHOLD} | Ventana: {WINDOW_SIZE} queries")

    lf = Langfuse(
        secret_key=os.environ["LANGFUSE_SECRET_KEY"],
        public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
        host=os.environ.get("LANGFUSE_HOST", "http://localhost:3000"),
    )

    # 1. Recuperar scores recientes
    scores = get_recent_scores(lf, window=WINDOW_SIZE)

    # 2. Calcular estadísticas
    stats = calculate_stats(scores)
    logger.info(f"Estadísticas: {json.dumps(stats)}")

    # 3. Evaluar condición de alerta
    alert, reason = check_alert_condition(stats)

    # 4. Guardar reporte
    report = save_monitoring_report(stats, alert, reason)

    # 5. Disparar alerta si corresponde
    if alert:
        logger.warning(f"⚠️  ALERTA DISPARADA: {reason}")
        send_alert_email(stats, reason)
        print(f"\n{'='*60}")
        print(f"⚠️  ALERTA DE CALIDAD RAG")
        print(f"   Motivo: {reason}")
        print(f"   Score medio: {stats['mean']} (umbral: {QUALITY_THRESHOLD})")
        print(f"{'='*60}\n")
        return 1  # Exit code 1 para CI/CD
    else:
        logger.info(f"✅ Calidad RAG dentro de parámetros: score medio = {stats['mean']}")
        print(f"\n✅ Calidad RAG OK: score medio = {stats['mean']} (umbral: {QUALITY_THRESHOLD})\n")
        return 0


if __name__ == "__main__":
    import sys
    exit_code = run_monitoring()
    sys.exit(exit_code)
```

2. Ejecuta el script de monitoreo:

```bash
python monitor_rag_quality.py
```

3. Prueba la detección de degradación bajando el umbral artificialmente:

```bash
# Simula degradación bajando el umbral por encima del score actual
QUALITY_THRESHOLD=0.99 python monitor_rag_quality.py
# Debe mostrar ALERTA y retornar exit code 1
echo "Exit code: $?"
```

#### Salida esperada (estado normal)

```
2024-01-15 10:32:01 [INFO] === Iniciando monitoreo de calidad RAG ===
2024-01-15 10:32:01 [INFO] Umbral configurado: 0.70 | Ventana: 100 queries
2024-01-15 10:32:02 [INFO] Recuperados 20 scores de Langfuse (ventana=100)
2024-01-15 10:32:02 [INFO] Estadísticas: {"count": 20, "mean": 0.7651, ...}

✅ Calidad RAG OK: score medio = 0.7651 (umbral: 0.70)
```

#### Salida esperada (alerta disparada con umbral alto)

```
2024-01-15 10:32:05 [WARNING] ⚠️  ALERTA DISPARADA: Score medio (0.7651) por debajo del umbral (0.99)

============================================================
⚠️  ALERTA DE CALIDAD RAG
   Motivo: Score medio (0.7651) por debajo del umbral (0.99)
   Score medio: 0.7651 (umbral: 0.99)
============================================================
```

#### Verificación

```bash
# Verifica el reporte de monitoreo
python3 -c "
import json
with open('reports/monitoring_report.json') as f:
    r = json.load(f)
print('Campos del reporte:', list(r.keys()))
assert 'mean' in r, 'Falta campo mean'
assert 'alert' in r, 'Falta campo alert'
assert 'timestamp' in r, 'Falta campo timestamp'
print('OK: Reporte de monitoreo válido')
print(f'Score medio actual: {r[\"mean\"]} | Alerta: {r[\"alert\"]}')
"
```

---

## 7. Validación y Pruebas

Ejecuta la siguiente secuencia de validación completa para verificar que las tres capas de observabilidad funcionan correctamente:

```bash
# ============================================================
# VALIDACIÓN COMPLETA - Lab 06-00-01
# ============================================================

echo "=== [1/5] Verificando servicios Docker ==="
docker compose -f docker-compose.langfuse.yml ps
curl -sf http://localhost:3000/api/public/health && echo "Langfuse: OK" || echo "Langfuse: FALLO"

echo ""
echo "=== [2/5] Verificando pipeline RAG instrumentado ==="
python -c "
from rag_pipeline_instrumented import rag_query
from langfuse import Langfuse
import os
from dotenv import load_dotenv
load_dotenv()
result = rag_query('¿Cuáles son los requisitos para abrir una cuenta corriente?', session_id='validation_test')
assert result['answer'], 'Respuesta vacía'
assert result['chunks'], 'Sin chunks recuperados'
assert result['metrics']['cost_usd'] >= 0, 'Coste inválido'
lf = Langfuse(secret_key=os.environ['LANGFUSE_SECRET_KEY'], public_key=os.environ['LANGFUSE_PUBLIC_KEY'], host=os.environ.get('LANGFUSE_HOST','http://localhost:3000'))
lf.flush()
print('Pipeline RAG: OK')
print(f'  Chunks recuperados: {len(result[\"chunks\"])}')
print(f'  Coste estimado: {result[\"metrics\"][\"cost_usd\"]} USD')
print(f'  Latencia: {result[\"metrics\"][\"latency_ms\"]} ms')
"

echo ""
echo "=== [3/5] Verificando reportes Ragas ==="
python3 -c "
import json, os
assert os.path.exists('reports/ragas_summary.json'), 'Falta ragas_summary.json'
with open('reports/ragas_summary.json') as f:
    s = json.load(f)
assert s['composite_mean'] > 0.5, f'Score compuesto demasiado bajo: {s[\"composite_mean\"]}'
assert s['n_questions'] == 20, f'Esperadas 20 preguntas, encontradas: {s[\"n_questions\"]}'
print(f'Ragas: OK | composite_mean={s[\"composite_mean\"]} | n={s[\"n_questions\"]}')
"

echo ""
echo "=== [4/5] Verificando suite promptfoo ==="
python3 -c "
import json, os
assert os.path.exists('reports/promptfoo_results.json'), 'Falta promptfoo_results.json'
with open('reports/promptfoo_results.json') as f:
    r = json.load(f)
total = r['stats']['successes'] + r['stats']['failures']
print(f'promptfoo: OK | {r[\"stats\"][\"successes\"]}/{total} tests passed')
if r['stats']['failures'] > 0:
    print(f'  ADVERTENCIA: {r[\"stats\"][\"failures\"]} tests fallando - revisar calidad RAG')
"

echo ""
echo "=== [5/5] Verificando script de monitoreo ==="
python3 -c "
import json, os
assert os.path.exists('reports/monitoring_report.json'), 'Falta monitoring_report.json'
with open('reports/monitoring_report.json') as f:
    r = json.load(f)
assert 'mean' in r and 'alert' in r, 'Campos faltantes en reporte'
print(f'Monitor: OK | mean={r[\"mean\"]} | alert={r[\"alert\"]}')
"

echo ""
echo "=== VALIDACIÓN COMPLETA FINALIZADA ==="
echo "Verifica manualmente:"
echo "  - Langfuse UI: http://localhost:3000 (traces con spans retrieval+generation)"
echo "  - Phoenix UI:  http://localhost:4000 (evaluaciones Ragas)"
echo "  - Reporte HTML: reports/promptfoo_report.html"
```

**Criterios de aceptación**:

| Check | Criterio mínimo |
|-------|----------------|
| Langfuse health | `{"status":"ok"}` |
| Trazas en Langfuse | ≥1 trace con spans `retrieval` y `generation` |
| `ragas_summary.json` | `composite_mean` ≥ 0.50 y `n_questions` = 20 |
| promptfoo | ≥ 8/10 tests passed |
| `monitoring_report.json` | Contiene `mean`, `alert`, `timestamp` |

---

## 8. Solución de Problemas

### Problema 1: Langfuse no recibe trazas (spans no aparecen en la UI)

**Síntomas**:
- El pipeline RAG se ejecuta sin errores en Python
- La UI de Langfuse muestra 0 trazas o las trazas aparecen vacías
- No hay error visible en consola pero `langfuse.flush()` no produce salida

**Causa**:
Las claves `LANGFUSE_SECRET_KEY` y `LANGFUSE_PUBLIC_KEY` en el archivo `.env` no coinciden con las generadas en la UI, o el SDK está enviando al host incorrecto (por defecto apunta a `cloud.langfuse.com` en lugar del self-hosted).

**Solución**:
```bash
# 1. Verifica que las variables están cargadas correctamente
python3 -c "
from dotenv import load_dotenv; import os; load_dotenv()
sk = os.environ.get('LANGFUSE_SECRET_KEY','NO_ENCONTRADA')
pk = os.environ.get('LANGFUSE_PUBLIC_KEY','NO_ENCONTRADA')
host = os.environ.get('LANGFUSE_HOST','NO_ENCONTRADA')
print(f'SK: {sk[:10]}...')
print(f'PK: {pk[:10]}...')
print(f'HOST: {host}')
"

# 2. Verifica conectividad con Langfuse self-hosted
curl -s -u "pk-lf-TU_PUBLIC_KEY:sk-lf-TU_SECRET_KEY" \
  http://localhost:3000/api/public/traces | python3 -m json.tool

# 3. Si el host es incorrecto, actualiza .env:
# LANGFUSE_HOST=http://localhost:3000  (sin barra final)

# 4. Fuerza el flush y verifica con debug
python3 -c "
import os; from dotenv import load_dotenv; load_dotenv()
from langfuse import Langfuse
lf = Langfuse(debug=True)  # Activa logs detallados
trace = lf.trace(name='test_connectivity')
trace.span(name='test_span')
lf.flush()
print('Flush completado')
"
```

---

### Problema 2: Ragas falla con `ValueError` o scores NaN en todas las métricas

**Síntomas**:
- `run_ragas_eval.py` termina con `ValueError: Expected type 'list[str]' for field 'contexts'`
- O los scores aparecen como `NaN` para `faithfulness` y `context_precision`
- El error puede mencionar `openai.AuthenticationError` o `RateLimitError`

**Causa**:
Dos causas frecuentes: (1) el formato del campo `contexts` en el dataset Ragas debe ser `list[list[str]]` (lista de listas), no `list[str]`; (2) la OpenAI API key para el LLM-as-judge no está disponible o ha alcanzado el límite de rate.

**Solución**:
```bash
# 1. Verifica el formato del dataset Ragas
python3 -c "
from datasets import Dataset
# Formato CORRECTO: contexts es lista de listas
d = Dataset.from_dict({
    'question': ['test?'],
    'answer': ['respuesta'],
    'contexts': [['chunk1', 'chunk2']],   # Lista de listas ✓
    'ground_truth': ['verdad'],
})
print('Formato contexts:', type(d['contexts'][0]))
assert isinstance(d['contexts'][0], list), 'contexts debe ser list[list[str]]'
print('OK: Formato correcto')
"

# 2. Verifica la API key para LLM-as-judge
python3 -c "
import os; from dotenv import load_dotenv; load_dotenv()
from openai import OpenAI
client = OpenAI(api_key=os.environ['OPENAI_API_KEY'])
resp = client.chat.completions.create(
    model='gpt-4o-mini',
    messages=[{'role':'user','content':'test'}],
    max_tokens=5
)
print('API key válida:', resp.choices[0].message.content)
"

# 3. Si hay RateLimitError, añade delay entre evaluaciones
# En run_ragas_eval.py, añade en build_ragas_dataset():
# import time; time.sleep(1)  # 1 segundo entre queries

# 4. Para verificar scores NaN, usa un dataset mínimo de prueba
python3 -c "
import os; from dotenv import load_dotenv; load_dotenv()
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import faithfulness
d = Dataset.from_dict({
    'question': ['¿Cuál es la capital de España?'],
    'answer': ['La capital de España es Madrid.'],
    'contexts': [['España es un país europeo. Su capital es Madrid.']],
    'ground_truth': ['Madrid'],
})
result = evaluate(dataset=d, metrics=[faithfulness])
print('faithfulness:', result['faithfulness'])
assert result['faithfulness'] > 0, 'Score NaN o 0 - revisar API key'
print('OK: Ragas funciona correctamente')
"
```

---

## 9. Limpieza

```bash
# ============================================================
# LIMPIEZA DEL LAB 06-00-01
# Ejecutar al finalizar para liberar recursos
# ============================================================

echo "Deteniendo servicios Docker..."
docker compose -f docker-compose.langfuse.yml down

echo "¿Eliminar volúmenes de datos de Langfuse? (s/N)"
read -r response
if [[ "$response" =~ ^[Ss]$ ]]; then
    docker compose -f docker-compose.langfuse.yml down -v
    echo "Volúmenes eliminados."
else
    echo "Volúmenes conservados para próximas sesiones."
fi

echo "Deteniendo servidor Phoenix (si está activo)..."
pkill -f "phoenix" 2>/dev/null || true

echo "Desactivando entorno virtual..."
deactivate 2>/dev/null || true

echo "Archivos generados en reports/ (conservados para referencia):"
ls -la reports/ 2>/dev/null || echo "  (directorio vacío)"

echo ""
echo "Para eliminar completamente el directorio del lab:"
echo "  rm -rf ~/lab06"
echo ""
echo "IMPORTANTE: El archivo .env contiene credenciales. Elimínalo si ya no lo necesitas:"
echo "  rm ~/lab06/.env"
```

> ⚠️ **Nota de seguridad**: El archivo `.env` contiene tu `OPENAI_API_KEY` y las claves de Langfuse. Elimínalo o revoca las claves cuando finalices el curso si no vas a continuar usándolas.

---

## 10. Resumen

En este laboratorio has construido una suite completa de observabilidad y evaluación automática para un pipeline RAG, implementando tres capas progresivas:

| Capa | Tecnología | Lo que captura |
|------|-----------|----------------|
| **Trazabilidad** | Langfuse SDK + `@observe` | Spans de retrieval y generation con tokens, costes, latencia y scores semánticos |
| **Evaluación de calidad** | Ragas + Arize Phoenix | faithfulness, answer_relevancy, context_precision, context_recall sobre 20 preguntas con ground truth |
| **Detección de regresión** | promptfoo | 10 test cases con assertions `contains`, `not-contains` y `llm-rubric` que detectan degradación automáticamente |
| **Monitoreo continuo** | Script Python + Langfuse API | Score promedio de últimas N queries con alerta por umbral configurable |

Los principios de observabilidad LLM de la lección 6.1 se han aplicado concretamente: captura de señales de dominio (no solo CPU/latencia), uso de hashes para privacidad en trazas, separación de spans por función (retrieval vs. generation), y ruteo de telemetría a backends especializados (Langfuse para trazas operativas, Phoenix para análisis de calidad).

### Recursos adicionales

- [Langfuse Docs — Python SDK Decorators](https://langfuse.com/docs/sdk/python/decorators)
- [Ragas Documentation — Metrics](https://docs.ragas.io/en/stable/concepts/metrics/index.html)
- [Arize Phoenix — Evaluations](https://docs.arize.com/phoenix/evaluation/how-to-evals)
- [promptfoo — Assertions Reference](https://www.promptfoo.dev/docs/configuration/expected-outputs/)
- [OpenTelemetry — GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/ai/)
- [OWASP LLM Top 10 — LLM09: Overreliance](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

---
