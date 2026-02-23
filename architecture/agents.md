# Agentes — src/agents/

> [← Pipeline](pipeline.md) · [Cognición →](cognition.md)

**Directorio**: `src/agents/` (878 líneas, 8 archivos)

---

## Sobre este documento

Este documento describe los **5 agentes LLM** que componen el equipo de Django — las entidades especializadas que generan respuestas, revisan contenido y aseguran cumplimiento. Cada agente tiene un rol específico, un modelo LLM asignado, y un estilo de operación propio. Juntos, forman el "personal" que Django coordina para responder a cada interacción.

### ¿Qué cubre este documento?

Documenta cada uno de los 5 agentes (IdentityCore, Business, Communication, Technical, Governance), sus roles específicos, qué modelo LLM usa cada uno, cómo se inicializan via `AgentCrew`, cómo participan en distintos pasos del pipeline, y los 2 perfiles de modelo disponibles (balanced, max_quality).

### ¿Cuál es su función en la arquitectura?

Los agentes son los **ejecutores creativos** — la contraparte humana de la cognición determinística. Mientras que la [Cognición](cognition.md) decide *qué* hacer y *quién* lo hace, los agentes son los que realmente *generan texto*. Son los únicos componentes del sistema que hacen llamadas a modelos de lenguaje (via el Model Router), así que toda la inteligencia lingüística de Django — su capacidad de conversar, analizar, redactar y revisar — pasa por ellos.

### ¿Cómo afecta al comportamiento de Django?

- **IdentityCoreAgent** es el agente principal — responde AS Harold (como si fuera Harold), cargado con su persona completa desde `persona.yaml`. La mayoría de conversaciones usan este agente
- **GovernanceAgent** revisa en modo "juez" a temperatura baja (0.2) con output JSON estricto — decide si una respuesta cumple las reglas
- El perfil de modelo activo cambia completamente el rendimiento: `max_quality` usa Gemini para todo (más lento, más caro), `balanced` (default) mezcla Gemini y Groq para equilibrar costo y calidad
- **Detalle importante**: IdentityCoreAgent NO extiende BaseAgent — tiene su propia implementación con `respond()` y carga de persona YAML independiente

### ¿Cómo interactúa con las demás piezas?

Los agentes son invocados por el [Pipeline](pipeline.md) en momentos específicos:
- **Paso 7 (LLM Generate)**: El agente seleccionado por [Cognición](cognition.md) genera la respuesta principal → la llamada va al [Model Router](../integrations/README.md) → al proveedor LLM (Gemini/Groq)
- **Paso 8 (Identity Review)**: Si el Plan lo requiere, `IdentityCoreAgent` revisa que la respuesta sea consistente con Harold
- **Paso 9 (Governance Review)**: Si el Plan lo requiere, `GovernanceAgent` evalúa cumplimiento y riesgo
- La [Config](../config/README.md) (`persona.yaml`) define el system prompt de cada agente, y `models.json` define qué modelo LLM usa cada uno
- El [Startup](startup.md) los inicializa como `AgentCrew` y los inyecta en el Orchestrator
- Cada llamada LLM genera un evento de token usage que se persiste en la [Base de Datos](database.md) via el callback del Model Router

---

## Tabla de Agentes

| Agente | Clase | Extiende | Rol | Modelo Default |
|--------|-------|----------|-----|----------------|
| Identity Core | `IdentityCoreAgent` | Standalone | Guardián de persona, responde COMO el principal | Gemini |
| Business | `BusinessAgent` | `BaseAgent` | Estrategia, deals, pricing | Groq |
| Communication | `CommunicationAgent` | `BaseAgent` | Emails, propuestas en estilo del principal | Gemini |
| Technical | `TechnicalAgent` | `BaseAgent` | Código, arquitectura, debugging | Groq |
| Governance | `GovernanceAgent` | `BaseAgent` | Meta-agente revisor de compliance (JSON output, temp 0.2) | Gemini |

> **Nota**: `IdentityCoreAgent` NO extiende `BaseAgent` — tiene su propia implementación con `respond()` y carga directa de persona desde YAML.

---

## AgentCrew

`AgentCrew` inicializa los 5 agentes con `ModelRouter` + paths de persona/governance.

- El orchestrator selecciona qué agente usar via [DecisionEngine](cognition.md) (paso 2)
- Los modelos son reasignables en runtime via [Model Manager](../dashboard/model-manager.md)
- Las asignaciones se guardan en [models.json](../config/README.md)

---

## Roles en el Pipeline

| Paso del Pipeline | Agente Usado | Cuándo |
|-------------------|-------------|--------|
| 7 (LLM Generate) | Seleccionado por DecisionEngine | Siempre (excepto Mode 3) |
| 8 (Identity Review) | Identity Core | Cuando `plan.has_step("identity_review")` — non-conversation tasks |
| 9 (Governance Review) | Governance Agent | Cuando `plan.has_step("governance_review")` — non-conversation tasks |

Ver: [Pipeline completo](pipeline.md)

---

## Configuración de Modelos por Agente

Los 2 perfiles preconfigurados cambian las asignaciones:

| Perfil | Identity Core | Business | Communication | Technical | Governance |
|--------|:---:|:---:|:---:|:---:|:---:|
| **balanced** (default) | Gemini | Groq | Gemini | Groq | Gemini |
| **max_quality** | Gemini | Gemini | Gemini | Gemini | Gemini |

Gestión: [Model Manager page](../dashboard/model-manager.md) · API: `PUT /models/assignment`, `PUT /models/profile` → ver [API](../api/README.md)

---

## Temas Relacionados

- [Cognición](cognition.md) — Cómo se selecciona el agente (DecisionEngine)
- [Pipeline](pipeline.md) — Dónde ejecutan los agentes (pasos 7, 8, 9)
- [Model Router](../integrations/README.md) — Cadena de fallback y circuit breaker
- [Model Manager](../dashboard/model-manager.md) — UI para reasignar modelos
- [Identidad](identity.md) — Los 22 módulos que influyen el comportamiento del Identity Core
- [Configuración](../config/README.md) — models.json, persona.yaml
