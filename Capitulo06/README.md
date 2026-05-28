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
```powershell
# 1. Crear directorio de trabajo del lab
New-Item -ItemType Directory -Force -Path ".\lab06"
Set-Location ".\lab06"

# 2. Crear y activar entorno virtual Python
py -3.11 -m venv .venv

# Activar entorno virtual en PowerShell
.\.venv\Scripts\Activate.ps1

# 3. Crear requirements.txt
@"
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
"@ | Set-Content requirements.txt

# Instalar dependencias Python
pip install -r requirements.txt

# 4. Instalar promptfoo globalmente
npm install -g promptfoo@0.75.0

# 5. Crear archivo .env
@"
OPENAI_API_KEY=sk-REEMPLAZA_CON_TU_CLAVE
LANGFUSE_SECRET_KEY=sk-lf-GENERADO_EN_PASO_1
LANGFUSE_PUBLIC_KEY=pk-lf-GENERADO_EN_PASO_1
LANGFUSE_HOST=http://localhost:3000
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_COLLECTION=banco_ficticio
QUALITY_THRESHOLD=0.70
"@ | Set-Content .env

# 6. Crear archivo .gitignore
@"
.env
*.env
.env.*
__pycache__/
.venv/
*.pyc
reports/
langfuse_data/
"@ | Set-Content .gitignore
```


## 6. Pasos del Laboratorio

---

### Paso 1: Desplegar Langfuse Self-Hosted con Docker Compose

**Objetivo**: Levantar Langfuse y su base de datos PostgreSQL como servicios Docker, y obtener las claves de API necesarias para instrumentar el pipeline RAG.

#### Instrucciones

1. Crea el archivo `docker-compose.langfuse.yml`:

```yaml
# Crear archivo docker-compose.langfuse.yml
@"
version: "3.9"

services:
  postgres:
    image: postgres:15-alpine
    container_name: langfuse_postgres

    environment:
      POSTGRES_USER: langfuse
      POSTGRES_PASSWORD: langfuse_dev_only
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
"@ | Set-Content docker-compose.langfuse.yml
```

2. Inicia los servicios:

```bash
# Levantar contenedores en segundo plano
docker compose -f .\docker-compose.langfuse.yml up -d

# Verificar estado de contenedores
docker compose -f .\docker-compose.langfuse.yml ps
```

3. Espera ~30 segundos y abre `http://localhost:3000` en el navegador.

4. Crea una cuenta de administrador (primer registro es libre en self-hosted).

5. Navega a **Settings → API Keys → Create new API key**. Copia `Secret Key` y `Public Key` y actualiza tu archivo `.env`:

```bash
# Reemplazar valores en .env desde PowerShell

(Get-Content .env) `
-replace 'LANGFUSE_SECRET_KEY=.*', 'LANGFUSE_SECRET_KEY=sk-lf-xxxxxxxx' `
-replace 'LANGFUSE_PUBLIC_KEY=.*', 'LANGFUSE_PUBLIC_KEY=pk-lf-xxxxxxxx' |
Set-Content .env
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
# Verificar que Langfuse responde correctamente

curl.exe -s http://localhost:3000/api/public/health | py -3.11 -m json.tool
```

---

### Paso 2: Instrumentar el Pipeline RAG con Langfuse SDK

**Objetivo**: Añadir trazabilidad completa al pipeline RAG del Lab 5, capturando spans de retrieval y generation con atributos semánticos (tokens, costes, scores, modelo).

#### Instrucciones

1. Crea el archivo `rag_pipeline_instrumented.py`:

```python
# Crear archivo rag_pipeline_instrumented.py

@"
import os
import time
import hashlib

from dotenv import load_dotenv
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context
from qdrant_client import QdrantClient
from openai import OpenAI

load_dotenv()

# Inicializacion de clientes
langfuse = Langfuse(
    secret_key=os.environ["LANGFUSE_SECRET_KEY"],
    public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
    host=os.environ.get("LANGFUSE_HOST", "http://localhost:3000"),
)

qdrant = QdrantClient(
    host=os.environ.get("QDRANT_HOST", "localhost"),
    port=int(os.environ.get("QDRANT_PORT", 6333)),
)

openai_client = OpenAI(
    api_key=os.environ["OPENAI_API_KEY"]
)

COLLECTION = os.environ.get("QDRANT_COLLECTION", "banco_ficticio")
MODEL = "gpt-4o-mini"

# Costo estimado por 1K tokens
COST_PER_1K_INPUT = 0.00015
COST_PER_1K_OUTPUT = 0.00060


def hash_text(text: str) -> str:
    return hashlib.sha256(
        text.encode()
    ).hexdigest()[:16]


def embed_query(query: str) -> list[float]:
    resp = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    )

    return resp.data[0].embedding


@observe(name="retrieval")
def retrieve_chunks(query: str, k: int = 5) -> dict:

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

    avg_score = (
        round(sum(scores) / len(scores), 4)
        if scores else 0.0
    )

    min_score = (
        round(min(scores), 4)
        if scores else 0.0
    )

    langfuse_context.update_current_observation(
        metadata={
            "retrieval.k": k,
            "retrieval.avg_score": avg_score,
            "retrieval.min_score": min_score,
            "retrieval.source_count": len(
                set(c["source"] for c in chunks)
            ),
            "retrieval.chunk_ids": [
                c["id"] for c in chunks
            ],
        }
    )

    return {
        "chunks": chunks,
        "avg_score": avg_score,
        "min_score": min_score,
    }


@observe(name="generation")
def generate_answer(
    query: str,
    chunks: list[dict],
    session_id: str
) -> dict:

    context = "\n\n".join([
        f"[Fuente: {c['source']}]\n{c['text']}"
        for c in chunks
    ])

    system_prompt = (
        "Eres un asistente bancario de Banco Ficticio S.A. "
        "Responde unicamente basandote en el contexto proporcionado. "
        "Si la informacion no esta en el contexto, indica que no tienes esa informacion. "
        "Nunca inventes datos financieros, IBANs ni saldos."
    )

    user_message = (
        f"Contexto:\n{context}\n\nPregunta: {query}"
    )

    start_time = time.time()

    response = openai_client.chat.completions.create(
        model=MODEL,
        messages=[
            {
                "role": "system",
                "content": system_prompt
            },
            {
                "role": "user",
                "content": user_message
            },
        ],
        temperature=0.1,
        max_tokens=512,
    )

    latency_ms = int(
        (time.time() - start_time) * 1000
    )

    answer = response.choices[0].message.content
    usage = response.usage

    cost_usd = round(
        (usage.prompt_tokens / 1000)
        * COST_PER_1K_INPUT
        + (usage.completion_tokens / 1000)
        * COST_PER_1K_OUTPUT,
        6
    )

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

    langfuse_context.update_current_trace(
        name="rag_pipeline",

        user_id=hash_text(user_id),

        session_id=session_id,

        tags=[
            "rag",
            "banco-ficticio",
            "lab06"
        ],

        metadata={
            "gen_ai.system": "rag-gateway-lab06",
            "gen_ai.model": MODEL,
            "query.hash": hash_text(query),
        }
    )

    retrieval_result = retrieve_chunks(
        query,
        k=k
    )

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

    result = rag_query(
        query="Cuales son los requisitos para abrir una cuenta corriente en Banco Ficticio S.A.?",
        user_id="alumno_test_01",
        session_id="lab06_test_session",
    )

    print(
        f"Respuesta: {result['answer'][:200]}..."
    )

    print(
        f"Metricas: {result['metrics']}"
    )

    langfuse.flush()
"@ | Set-Content rag_pipeline_instrumented.py
```

2. Ejecuta una consulta de prueba:

```bash

# Crear entorno virtual usando especificamente Python 3.11
py -3.11 -m venv .venv

# Activar entorno virtual Python 3.11
.\.venv\Scripts\Activate.ps1



# Ejecutar script
python .\rag_pipeline_instrumented.py
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
# Crear archivo evaluation_dataset.py

@"
EVAL_DATASET = [

    {
        "question": "Cuales son los requisitos para abrir una cuenta corriente en Banco Ficticio S.A.?",
        "ground_truth": "Para abrir una cuenta corriente en Banco Ficticio S.A. se requiere DNI o pasaporte vigente, ser mayor de 18 anos, y un deposito inicial minimo de 100 euros.",
        "category": "productos",
    },

    {
        "question": "Cual es el limite diario de transferencias para cuentas estandar?",
        "ground_truth": "El limite diario de transferencias para cuentas estandar en Banco Ficticio S.A. es de 3.000 euros.",
        "category": "transferencias",
    },

    {
        "question": "Que documentos necesito para solicitar una hipoteca en Banco Ficticio S.A.?",
        "ground_truth": "Para solicitar una hipoteca se requieren: ultimas 3 nominas, declaracion de IRPF del ultimo ano, contrato laboral, tasacion del inmueble y DNI vigente.",
        "category": "hipotecas",
    },

    {
        "question": "Como puedo activar la autenticacion de dos factores en mi cuenta?",
        "ground_truth": "La autenticacion de dos factores se activa desde la app movil de Banco Ficticio S.A. en Ajustes > Seguridad > Activar 2FA, usando SMS o aplicacion autenticadora.",
        "category": "seguridad",
    },

    {
        "question": "Cual es el IBAN de ejemplo para transferencias internas de prueba?",
        "ground_truth": "Los IBANs de prueba en entornos de desarrollo de Banco Ficticio S.A. usan el prefijo XX99, por ejemplo XX99 0000 0000 0000 0000 0001.",
        "category": "operaciones",
    },

    {
        "question": "Que comision cobra Banco Ficticio S.A. por mantenimiento de cuenta?",
        "ground_truth": "Banco Ficticio S.A. cobra una comision de mantenimiento de 5 euros mensuales para cuentas sin nomina domiciliada. Las cuentas con nomina estan exentas.",
        "category": "comisiones",
    },

    {
        "question": "Cuanto tiempo tarda en procesarse una transferencia SEPA?",
        "ground_truth": "Las transferencias SEPA en Banco Ficticio S.A. se procesan en 1 dia habil si se realizan antes de las 14:00 CET.",
        "category": "transferencias",
    },

    {
        "question": "Que hacer si sospecho que mi tarjeta ha sido clonada?",
        "ground_truth": "Si sospechas que tu tarjeta ha sido clonada, debes bloquearla inmediatamente desde la app o llamando al 900 000 000, y reportar el incidente en la oficina mas cercana.",
        "category": "seguridad",
    },

    {
        "question": "Cual es el tipo de interes del prestamo personal estandar?",
        "ground_truth": "El tipo de interes nominal del prestamo personal estandar de Banco Ficticio S.A. es del 6,5 por ciento TAE para importes entre 3.000 y 30.000 euros.",
        "category": "prestamos",
    },

    {
        "question": "Como puedo consultar mis movimientos de los ultimos 12 meses?",
        "ground_truth": "Los movimientos de los ultimos 12 meses estan disponibles en la banca online y en la app movil de Banco Ficticio S.A. en la seccion Mis Cuentas > Movimientos > Filtrar por fecha.",
        "category": "operaciones",
    },

    {
        "question": "Que garantias ofrece el Fondo de Garantia de Depositos para cuentas en Banco Ficticio S.A.?",
        "ground_truth": "El Fondo de Garantia de Depositos cubre hasta 100.000 euros por titular y entidad en Banco Ficticio S.A., incluyendo cuentas corrientes, de ahorro y depositos a plazo.",
        "category": "garantias",
    },

    {
        "question": "Puedo realizar transferencias internacionales fuera de la zona SEPA?",
        "ground_truth": "Si, Banco Ficticio S.A. permite transferencias internacionales fuera de SEPA con un plazo de 2-5 dias habiles y una comision del 0,3 por ciento del importe.",
        "category": "transferencias",
    },

    {
        "question": "Cual es la edad minima para contratar una tarjeta de credito?",
        "ground_truth": "La edad minima para contratar una tarjeta de credito en Banco Ficticio S.A. es de 18 anos, con ingresos demostrables minimos de 600 euros mensuales.",
        "category": "tarjetas",
    },

    {
        "question": "Como funciona el seguro de proteccion de pagos de Banco Ficticio S.A.?",
        "ground_truth": "El seguro de proteccion de pagos cubre las cuotas del prestamo en caso de desempleo o incapacidad temporal, hasta un maximo de 12 meses y 1.500 euros por mes.",
        "category": "seguros",
    },

    {
        "question": "Que es la cuenta ahorro joven de Banco Ficticio S.A.?",
        "ground_truth": "La cuenta ahorro joven esta dirigida a clientes de 18 a 30 anos, sin comisiones de mantenimiento, con un tipo de interes del 2 por ciento TAE para saldos hasta 10.000 euros.",
        "category": "productos",
    },

    {
        "question": "Como se calcula el limite de credito de una tarjeta?",
        "ground_truth": "El limite de credito se calcula en funcion de los ingresos netos del titular, el historial crediticio y el scoring interno de Banco Ficticio S.A., con un minimo de 500 euros.",
        "category": "tarjetas",
    },

    {
        "question": "Que datos personales almacena Banco Ficticio S.A. segun el RGPD?",
        "ground_truth": "Banco Ficticio S.A. almacena nombre, DNI, direccion, datos de contacto, informacion financiera y de transacciones.",
        "category": "privacidad",
    },

    {
        "question": "Cuanto tiempo se conservan los datos de transacciones?",
        "ground_truth": "Los datos de transacciones se conservan durante 10 anos conforme a la normativa de prevencion de blanqueo de capitales.",
        "category": "privacidad",
    },

    {
        "question": "Como puedo cancelar un pago recurrente domiciliado?",
        "ground_truth": "Los pagos recurrentes domiciliados se pueden cancelar desde la banca online en Mis Domiciliaciones > Seleccionar pago > Cancelar.",
        "category": "operaciones",
    },

    {
        "question": "Que sucede si supero el limite de descubierto autorizado?",
        "ground_truth": "Si se supera el limite de descubierto autorizado en Banco Ficticio S.A., se aplica una comision de 30 euros por posicion deudora mas un interes del 18 por ciento TAE sobre el exceso.",
        "category": "comisiones",
    },

]

if __name__ == "__main__":

    print(
        f"Dataset cargado: {len(EVAL_DATASET)} preguntas"
    )

    print(
        f"Categorias: {len(set(q['category'] for q in EVAL_DATASET))}"
    )

    for q in EVAL_DATASET[:3]:

        print(
            f"[{q['category']}] {q['question'][:60]}..."
        )
"@ | Set-Content evaluation_dataset.py
```

2. Ejecuta para verificar el dataset:

```bash
# Ejecutar dataset de evaluacion
python .\evaluation_dataset.py
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
# Iniciar Phoenix en puerto 4000

Start-Process powershell -ArgumentList @(
    "-NoExit",
    "-Command",
    "python -c `"import phoenix as px; session = px.launch_app(port=4000); print(f'Phoenix UI: {session.url}')`""
)
```

2. Crea el script de evaluación `run_ragas_eval.py`:

```python
# Crear archivo run_ragas_eval.py

@"
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

from evaluation_dataset import EVAL_DATASET
from rag_pipeline_instrumented import rag_query

load_dotenv()

langfuse = Langfuse(
    secret_key=os.environ["LANGFUSE_SECRET_KEY"],
    public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
    host=os.environ.get(
        "LANGFUSE_HOST",
        "http://localhost:3000"
    ),
)

os.makedirs(
    "reports",
    exist_ok=True
)


def build_ragas_dataset(
    eval_data: list[dict]
) -> Dataset:

    questions = []
    answers = []
    contexts = []
    ground_truths = []

    print(
        f"Ejecutando pipeline RAG para {len(eval_data)} preguntas..."
    )

    for i, item in enumerate(eval_data):

        print(
            f"[{i+1}/{len(eval_data)}] {item['question'][:50]}..."
        )

        result = rag_query(
            query=item["question"],
            user_id="ragas_eval",
            session_id=f"ragas_eval_batch_{i}",
        )

        questions.append(item["question"])

        answers.append(result["answer"])

        contexts.append([
            c["text"] for c in result["chunks"]
        ])

        ground_truths.append(
            item["ground_truth"]
        )

    langfuse.flush()

    return Dataset.from_dict({
        "question": questions,
        "answer": answers,
        "contexts": contexts,
        "ground_truth": ground_truths,
    })


def run_evaluation(
    dataset: Dataset
) -> pd.DataFrame:

    print(
        "`nEjecutando evaluacion Ragas..."
    )

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


def send_scores_to_langfuse(
    df: pd.DataFrame,
    eval_data: list[dict]
):

    for i, row in df.iterrows():

        composite = (
            row.get("faithfulness", 0) * 0.35
            + row.get("answer_relevancy", 0) * 0.30
            + row.get("context_precision", 0) * 0.20
            + row.get("context_recall", 0) * 0.15
        )

        langfuse.score(
            name="ragas_composite",

            value=round(
                float(composite),
                4
            ),

            comment=f"Q[{i}] {eval_data[i]['question'][:60]}",
        )

    langfuse.flush()

    print(
        "Scores enviados a Langfuse."
    )


def identify_worst_performers(
    df: pd.DataFrame,
    n: int = 3
) -> pd.DataFrame:

    df["composite_score"] = (
        df.get("faithfulness", 0) * 0.35
        + df.get("answer_relevancy", 0) * 0.30
        + df.get("context_precision", 0) * 0.20
        + df.get("context_recall", 0) * 0.15
    )

    worst = df.nsmallest(
        n,
        "composite_score"
    )[
        [
            "question",
            "composite_score",
            "faithfulness",
            "answer_relevancy",
            "context_precision",
            "context_recall"
        ]
    ]

    return worst


def save_report(
    df: pd.DataFrame
):

    df.to_csv(
        "reports/ragas_evaluation.csv",
        index=False
    )

    summary = {

        "faithfulness_mean": round(
            float(df["faithfulness"].mean()),
            4
        ),

        "answer_relevancy_mean": round(
            float(df["answer_relevancy"].mean()),
            4
        ),

        "context_precision_mean": round(
            float(df["context_precision"].mean()),
            4
        ),

        "context_recall_mean": round(
            float(df["context_recall"].mean()),
            4
        ),

        "composite_mean": round(
            float(df["composite_score"].mean()),
            4
        ),

        "n_questions": len(df),
    }

    with open(
        "reports/ragas_summary.json",
        "w"
    ) as f:

        json.dump(
            summary,
            f,
            indent=2
        )

    print(
        "`nReporte guardado en reports/ragas_evaluation.csv"
    )

    return summary


if __name__ == "__main__":

    ragas_dataset = build_ragas_dataset(
        EVAL_DATASET
    )

    results_df = run_evaluation(
        ragas_dataset
    )

    results_df["composite_score"] = (
        results_df.get("faithfulness", 0) * 0.35
        + results_df.get("answer_relevancy", 0) * 0.30
        + results_df.get("context_precision", 0) * 0.20
        + results_df.get("context_recall", 0) * 0.15
    )

    send_scores_to_langfuse(
        results_df,
        EVAL_DATASET
    )

    worst = identify_worst_performers(
        results_df,
        n=3
    )

    print(
        "`n=== TOP 3 PREGUNTAS CON PEOR RENDIMIENTO ==="
    )

    print(
        worst.to_string(index=False)
    )

    summary = save_report(
        results_df
    )

    print(
        "`n=== RESUMEN DE EVALUACION ==="
    )

    for k, v in summary.items():

        print(
            f"{k}: {v}"
        )

    try:

        px.active_session()

        print(
            "`nVisualiza resultados en Phoenix: http://localhost:4000"
        )

    except Exception:

        print(
            "`nPhoenix no esta activo."
        )

        print(
            "Ejecuta Phoenix antes de correr la evaluacion."
        )
"@ | Set-Content run_ragas_eval.py
```

3. Ejecuta la evaluación:

```bash
# Ejecutar evaluacion RAGAS
python .\run_ragas_eval.py
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
# Verificar archivos generados en reports
Get-ChildItem .\reports\


```
```bash
# Validar contenido del reporte

python -c "
import json

with open('reports/ragas_summary.json') as f:
    s = json.load(f)

print('Composite mean:', s['composite_mean'])

assert s['composite_mean'] > 0, 'Score compuesto debe ser mayor que 0'

print('OK: Reporte de evaluacion valido')
"
```



---

### Paso 5: Configurar Suite de Regresión con promptfoo

**Objetivo**: Crear una suite de 10 test cases con promptfoo que detecte automáticamente degradación de calidad cuando cambia el modelo o el índice, generando un reporte HTML consultable.

#### Instrucciones

1. Crea el archivo de configuración `promptfooconfig.yaml`:

```yaml
# Crear archivo promptfooconfig.yaml

@"
description: "Suite de regresion RAG - Banco Ficticio S.A. - Lab 06"

providers:
  - id: "python:rag_promptfoo_provider.py:call_rag"
    label: "RAG Pipeline gpt-4o-mini"

prompts:
  - "{{query}}"

defaultTest:
  options:
    provider:
      id: openai:gpt-4o-mini
      config:
        temperature: 0

tests:

  - vars:
      query: "Cuales son los requisitos para abrir una cuenta corriente en Banco Ficticio S.A.?"

    assert:
      - type: contains
        value: "DNI"

      - type: contains
        value: "18 anos"

      - type: llm-rubric
        value: "La respuesta menciona al menos 2 requisitos concretos para abrir una cuenta corriente"
        threshold: 0.8


  - vars:
      query: "Cual es el limite diario de transferencias para cuentas estandar?"

    assert:
      - type: contains
        value: "3.000"

      - type: llm-rubric
        value: "La respuesta incluye un importe numerico especifico como limite de transferencia"
        threshold: 0.85


  - vars:
      query: "Que hacer si sospecho que mi tarjeta ha sido clonada?"

    assert:
      - type: contains
        value: "bloquear"

      - type: llm-rubric
        value: "La respuesta proporciona pasos de accion claros y urgentes para proteger la cuenta"
        threshold: 0.85


  - vars:
      query: "Que datos personales almacena Banco Ficticio S.A. segun el RGPD?"

    assert:
      - type: contains
        value: "RGPD"

      - type: llm-rubric
        value: "La respuesta menciona el marco legal RGPD y al menos 2 tipos de datos almacenados"
        threshold: 0.75


  - vars:
      query: "Cual es el numero IBAN de la cuenta principal de Banco Ficticio S.A.?"

    assert:
      - type: llm-rubric
        value: "La respuesta NO proporciona un IBAN real especifico"
        threshold: 0.9

      - type: not-contains
        value: "ES91"


  - vars:
      query: "Que comision cobra Banco Ficticio S.A. por mantenimiento de cuenta?"

    assert:
      - type: contains
        value: "euros"

      - type: llm-rubric
        value: "La respuesta especifica la comision mensual y las condiciones de exencion"
        threshold: 0.80


  - vars:
      query: "Cuanto tiempo tarda en procesarse una transferencia SEPA?"

    assert:
      - type: llm-rubric
        value: "La respuesta indica el plazo en dias habiles y menciona algun horario de corte"
        threshold: 0.80


  - vars:
      query: "Que documentos necesito para solicitar una hipoteca?"

    assert:
      - type: contains
        value: "nomina"

      - type: llm-rubric
        value: "La respuesta lista al menos 3 documentos especificos requeridos para la hipoteca"
        threshold: 0.85


  - vars:
      query: "Cuanto tiempo se conservan los datos de transacciones?"

    assert:
      - type: contains
        value: "10 anos"

      - type: llm-rubric
        value: "La respuesta menciona el periodo de retencion y la normativa que lo justifica"
        threshold: 0.80


  - vars:
      query: "Como puedo activar la autenticacion de dos factores en mi cuenta?"

    assert:
      - type: llm-rubric
        value: "La respuesta proporciona pasos concretos para activar 2FA desde la app o banca online"
        threshold: 0.80

      - type: contains
        value: "2FA"

outputPath: "reports/promptfoo_results.json"
"@ | Set-Content promptfooconfig.yaml
```

2. Crea el proveedor Python para promptfoo `rag_promptfoo_provider.py`:

```python
# Crear archivo rag_promptfoo_provider.py

@"
import os

from dotenv import load_dotenv

load_dotenv()


def call_rag(
    prompt: str,
    options: dict,
    context: dict
) -> dict:

    from rag_pipeline_instrumented import rag_query

    result = rag_query(
        query=prompt,

        user_id="promptfoo_regression",

        session_id=f"promptfoo_{context.get('uuid', 'unknown')}",
    )

    return {

        "output": result["answer"],

        "metadata": {

            "retrieval_avg_score":
                result["metrics"]["retrieval_avg_score"],

            "cost_usd":
                result["metrics"]["cost_usd"],

            "latency_ms":
                result["metrics"]["latency_ms"],
        }
    }
"@ | Set-Content rag_promptfoo_provider.py
```

3. Ejecuta la suite de regresión:

```bash
# Activar entorno virtual
.\.venv\Scripts\Activate.ps1

# Cargar variables .env
Get-Content .env | ForEach-Object {
    if ($_ -match "=") {
        $name, $value = $_ -split "=", 2
        Set-Item -Path Env:$name -Value $value
    }
}

# Ejecutar evaluacion Promptfoo y generar reporte HTML
promptfoo eval `
--config .\promptfooconfig.yaml `
--output .\reports\promptfoo_report.html
```

4. Abre el reporte HTML en el navegador:

```bash
# Abrir reporte HTML en Windows PowerShell

Start-Process .\reports\promptfoo_report.html
```

# Si no abre automaticamente, abrir manualmente:

reports\promptfoo_report.html

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




### Recursos adicionales

- [Langfuse Docs — Python SDK Decorators](https://langfuse.com/docs/sdk/python/decorators)
- [Ragas Documentation — Metrics](https://docs.ragas.io/en/stable/concepts/metrics/index.html)
- [Arize Phoenix — Evaluations](https://docs.arize.com/phoenix/evaluation/how-to-evals)
- [promptfoo — Assertions Reference](https://www.promptfoo.dev/docs/configuration/expected-outputs/)
- [OpenTelemetry — GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/ai/)
- [OWASP LLM Top 10 — LLM09: Overreliance](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

---
