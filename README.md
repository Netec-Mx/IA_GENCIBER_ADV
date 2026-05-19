# IA Generativa en Ciberseguridad Avanzada

Curso avanzado orientado al diseño e implementación de guardrails y controles de seguridad a nivel de software para sistemas basados en LLM, utilizando herramientas open source y servicios cloud de AWS y Azure, con enfoque en defensa en profundidad, RAG seguro, validación de IO, observabilidad y red teaming.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [Construcción de matriz de riesgos basada en OWASP + ATLAS para un caso bancario](Capitulo01/README.md#construcción-de-matriz-de-riesgos-basada-en-owasp-atlas-para-un-caso-bancario)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración estimada: 25 min

### Capítulo 2

- [Configurar un guardrail en Bedrock o Foundry y validar bloqueo de prompt injection](Capitulo02/README.md#configurar-un-guardrail-en-bedrock-o-foundry-y-validar-bloqueo-de-prompt-injection)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración estimada: 20 min

### Capítulo 3

- [Integrar LLM Guard + Presidio en un gateway mock y bloquear PII + injection](Capitulo03/README.md#integrar-llm-guard-presidio-en-un-gateway-mock-y-bloquear-pii-injection)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración estimada: 20 min

### Capítulo 4

- [Desplegar LiteLLM + OPA y restringir acceso a modelos según rol](Capitulo04/README.md#desplegar-litellm-opa-y-restringir-acceso-a-modelos-según-rol)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración estimada: 30 min

### Capítulo 5

- [Pipeline seguro de ingestión → validación → indexación con RBAC activo](Capitulo05/README.md#pipeline-seguro-de-ingestión-validación-indexación-con-rbac-activo)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración estimada: 30 min

### Capítulo 6

- [Crear suite básica de evaluación automática para RAG + detección de degradación](Capitulo06/README.md#crear-suite-básica-de-evaluación-automática-para-rag-detección-de-degradación)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración estimada: 35 min

### Capítulo 7

- [Ejecutar pruebas con garak o PyRIT contra el gateway y medir reducción de attack success rate](Capitulo07/README.md#ejecutar-pruebas-con-garak-o-pyrit-contra-el-gateway-y-medir-reducción-de-attack-success-rate)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración estimada: 30 min

### Capítulo 8

- [Configurar n8n con credenciales seguras usando variables de entorno en Docker y protegerlo con un reverse proxy Nginx con TLS](Capitulo08/README.md#configurar-n8n-con-credenciales-seguras-usando-variables-de-entorno-en-docker-y-protegerlo-con-un-reverse-proxy-nginx-con-tls)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración estimada: 40 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.
