# Cognición — src/cognition/

> [← Agentes](agents.md) · [Memoria →](memory.md)

**Directorio**: `src/cognition/` (349 líneas, 3 archivos)

---

## Sobre este documento

Este documento describe la **capa de cognición determinística** de Django — el componente que toma decisiones estratégicas **sin usar ningún LLM**. Mientras que los agentes LLM generan texto con creatividad, la cognición es pura lógica: analiza el mensaje, calcula el riesgo, selecciona la estrategia óptima y crea un plan de ejecución paso a paso. Todo esto ocurre en milisegundos, con cero costo de tokens y resultados 100% predecibles.

### ¿Qué cubre este documento?

Documenta tres componentes: el `DecisionEngine` (un objeto inmutable que evalúa cada mensaje y decide categoría, estrategia, agente, riesgo y si necesita revisión), el `Planner` (sin estado, construye un Plan con pasos secuenciales y puertas condicionales), y `TaskCategory` (el enum compartido de categorías de tareas que evita dependencias circulares entre módulos).

### ¿Cuál es su función en la arquitectura?

La cognición es el **cerebro analítico** de Django — la parte que piensa antes de actuar. Sin cognición, el Orchestrator tendría que enviar cada mensaje al LLM sin contexto estratégico. Con cognición, Django sabe antes de generar la respuesta si es una conversación casual (respuesta directa), un análisis de negocio (análisis estructurado), una investigación (requiere web search) o una tarea compleja (requiere múltiples agentes). Esto hace que Django sea **más eficiente** (no gasta tokens innecesarios), **más seguro** (evalúa riesgo antes de actuar) y **más coherente** (el plan guía toda la ejecución).

### ¿Cómo afecta al comportamiento de Django?

- El `DecisionEngine` determina **qué agente** responderá (IdentityCore, Business, Communication, Technical) — esto afecta directamente el tono y enfoque de la respuesta
- La estrategia (`DIRECT_RESPONSE` vs `STRUCTURED_ANALYSIS` vs `RESEARCH_REQUIRED`) determina **cuánto esfuerzo** dedicará Django al mensaje
- El `preliminary_risk` y las flags `requires_identity_review`/`requires_governance_review` determinan **qué controles** se aplicarán a la respuesta
- El `Plan` del Planner define **qué pasos** se ejecutarán — si no hay paso de "identity_review" en el Plan, esa revisión se salta

### ¿Cómo interactúa con las demás piezas?

La cognición está en el centro temprano del [Pipeline](pipeline.md):
- **Entrada**: Recibe el mensaje del usuario + `TaskCategory` (clasificado por el paso 1 del Orchestrator) + `IdentityProfile` (inyectado en el constructor durante el [Startup](startup.md))
- **Paso 2 → DecisionEngine.evaluate()**: Produce un `DecisionResult` que viaja por todo el pipeline — lo consumen:
  - [Identidad](identity.md): la modulación de decisiones (paso 3c) evalúa alineación decisión-identidad
  - [Identidad](identity.md): la confianza (paso 3d) usa el resultado como señal
  - El propio Orchestrator: selecciona el [Agente](agents.md) correcto
- **Paso 3 → Planner.build()**: Genera el `Plan` que el Orchestrator sigue — las puertas del Plan habilitan/deshabilitan los pasos de revisión
- El `Orchestrator` **no puede arrancar sin cognición** — tiene guardas `isinstance` en su constructor que lanzan `TypeError` si no recibe `DecisionEngine` y `Planner` válidos
- `TaskCategory` es importado tanto por `decision_engine.py` como por `orchestrator.py` desde `src/flows/categories.py` para romper la dependencia circular

---

## Principio Fundamental

Capa **100% determinística** — cero llamadas LLM, cero IO, cero acceso a repositorios. El [Orchestrator](pipeline.md) **no puede iniciar sin ella** (`isinstance` guards en constructor — `TypeError` si falta).

---

## DecisionEngine — `decision_engine.py` (150 líneas)

**Inmutable**: `__slots__` + `__setattr__` override → completamente frozen después de `__init__`.

### Strategy Enum

| Strategy | Cuándo |
|----------|--------|
| `DIRECT_RESPONSE` | Conversación, preguntas simples |
| `STRUCTURED_ANALYSIS` | Análisis de negocio, técnico |
| `RESEARCH_REQUIRED` | Requiere web search |
| `MULTI_AGENT` | Múltiples dominios → [ParallelExecutor](pipeline.md) |

### `evaluate(category, message, governance_enabled)` → `DecisionResult`

`DecisionResult` es un frozen dataclass con:

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `category` | TaskCategory | Categoría de la tarea |
| `strategy` | Strategy | Estrategia seleccionada |
| `selected_agent` | str | Agente recomendado |
| `autonomy_level` | int | Nivel de autonomía actual |
| `preliminary_risk` | str | low/medium/high/critical |
| `requires_identity_review` | bool | ¿Necesita revisión de identidad? |
| `requires_governance_review` | bool | ¿Necesita revisión de gobernanza? |
| `reasoning` | str | Explicación de la decisión |

### Identity Profile (read-only)

El DecisionEngine recibe opcionalmente un `IdentityProfile` en constructor. Es accesible via `@property` de solo lectura, almacenado pero aún no usado directamente en `evaluate()` — los módulos de [identidad](identity.md) lo procesan en pasos posteriores.

---

## Planner — `planner.py` (118 líneas)

**Stateless**: Constructor sin argumentos. `governance_enabled` se pasa como parámetro.

### `build(decision, governance_enabled=True)` → `Plan`

| Componente | Tipo | Descripción |
|-----------|------|-------------|
| `PlanStep` | frozen dataclass | step_id, step_type, agent, description |
| `Plan` | frozen dataclass | strategy, steps, requires_multi_step, selected_agent, decision + `has_step()` helper |

El Plan incluye steps condicionales de gate:
- `identity_review` step → se agrega si `decision.requires_identity_review`
- `governance_review` step → se agrega si `decision.requires_governance_review` AND `governance_enabled`

---

## Categories — `categories.py` (81 líneas)

`TaskCategory` enum compartido entre orchestrator y cognition. Extraído para evitar dependencia circular.

Categorías: CONVERSATION, BUSINESS, TECHNICAL, CREATIVE, RESEARCH, ANALYSIS, COMMUNICATION, GOVERNANCE, TRAINING, META (y más).

---

## Grafo de Dependencias

```
categories.py ← decision_engine.py ← planner.py
     ↑                                    │
     └─── orchestrator.py ←───────────────┘
                (via DI en constructor)
```

No hay dependencia inversa cognition → orchestrator.

---

## Integración en el Pipeline

| Paso | Módulo | Función |
|------|--------|---------|
| 1 | Orchestrator | `classify()` → `TaskCategory` (heurísticas de keywords) |
| 2 | **DecisionEngine** | `evaluate()` → `DecisionResult` |
| 3 | **Planner** | `build()` → `Plan` con gate steps |

El resultado del DecisionEngine también alimenta:
- [Identity Decision Modulation](identity.md) (paso 3c)
- [Identity Confidence](identity.md) (paso 3d)
- [Identity Autonomy Modulation](identity.md) (paso 3e)

---

## Temas Relacionados

- [Pipeline](pipeline.md) — El pipeline completo donde operan
- [Agentes](agents.md) — Quiénes ejecutan las decisiones
- [Identidad](identity.md) — Los módulos que evalúan la decisión
- [Teleología](teleology.md) — Las metas que informan el contexto
