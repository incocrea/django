# Teleología — src/teleology/

> [← Identidad](identity.md) · [Evaluación →](evaluation.md)

**Directorio**: `src/teleology/` (2,294 líneas, 11 archivos)

---

## Sobre este documento

Este documento describe el **sistema teleológico** de Django — la capa que le da **propósito**. Mientras la identidad define *quién es* Django y la cognición decide *cómo* responder, la teleología define *para qué* lo hace. Es el sistema de metas, prioridades, recompensas y resolución de conflictos que permite a Django operar con dirección estratégica en lugar de simplemente reaccionar a mensajes.

### ¿Qué cubre este documento?

Documenta la **jerarquía de 4 tipos de metas** (mission → strategic → operational → tactical), la **máquina de estados** (7 estados: CREATED → ACTIVE → IN_PROGRESS → COMPLETED…), el **motor de prioridades** (7 factores ponderados incluyendo urgencia, impacto, alineación con identidad), el **modelo de recompensa** (7 componentes que evalúan el progreso), el **sistema de resolución de conflictos** (6 tipos de conflicto × 5 estrategias de resolución), los **controles de gobernanza** (4 chequeos antes de ejecutar), el **loop de fondo** (cada 30 minutos evalua progreso), y los **3 hooks de middleware** que se inyectan en el pipeline.

### ¿Cuál es su función en la arquitectura?

La teleología es el **navegante estratégico**. Sin este sistema, Django reacciona a mensajes individuales sin visión de conjunto — responde bien a cada pregunta pero no trabaja hacia ningún objetivo mayor. Con la teleología, Django puede tener metas como "aprender sobre el negocio de Harold", priorizar tareas que contribuyen a esas metas, detectar conflictos entre objetivos contradictorios, y reportar progreso.

### ¿Cómo afecta al comportamiento de Django?

- **Metas activas enriquecen el contexto**: si Django tiene una meta activa relevante al mensaje actual, esa meta se inyecta en el contexto del prompt — el LLM sabe qué está intentando lograr Django
- **Priorización inteligente**: cuando hay múltiples tareas, la teleología calcula cuál es más urgente/importante usando 7 factores ponderados
- **Resolución de conflictos**: si dos metas se contradicen, el sistema detecta el conflicto y aplica una de 5 estrategias (priorizar, comprometer, delegar, aplazar, descomponer)
- **Importante**: actualmente el nivel de autonomía es 0 (Observer), por lo que el background loop **solo observa** — no ejecuta acciones autónomas. El sistema está construido pero esperando que el nivel de autonomía suba para activarse completamente

### ¿Cómo interactúa con las demás piezas?

- **[Pipeline](pipeline.md)**: 3 hooks de middleware se inyectan en el Orchestrator:
  - `PRE_CLASSIFY`: puede enriquecer el contexto con metas relevantes antes de clasificar el mensaje
  - `POST_CLASSIFY`: puede ajustar la categoría del mensaje si está relacionado con una meta activa
  - `POST_STORE`: registra señales de recompensa y actualiza progreso de metas después de cada interacción
- **[Identidad](identity.md)**: el motor de prioridades incluye "alineación con identidad" como uno de sus 7 factores — las metas que se alinean con los valores de Harold tienen mayor prioridad
- **[Base de Datos](database.md)**: 4 tablas dedicadas en Postgres (`goals`, `goal_progress`, `goal_conflicts`, `reward_signals`) persisten todo el estado teleológico
- **[Eventos](events.md)**: emite eventos de ciclo de vida de metas, conflictos detectados, y recompensas
- **[Gobernanza](../dashboard/governance-console.md)**: 4 chequeos de gobernanza antes de ejecutar cualquier acción autónoma (nivel de autonomía, tipo de riesgo, límites por categoría, cooldown)
- **[Dashboard](../dashboard/goals.md)**: la página Goals permite crear, editar, activar, completar y cancelar metas manualmente
- **[API](../api/README.md)**: 11 endpoints para CRUD de metas, detección de conflictos, señales de recompensa y evaluación de gobernanza
- **[Startup](startup.md)**: el GoalManager se inicializa durante el lifespan y registra sus hooks en el middleware del Orchestrator

---

## Resumen

Sistema de metas y propósitos del agente. Define QUÉ quiere lograr Django, con qué prioridad, cómo medir progreso, y cómo resolver conflictos entre objetivos.

---

## Archivos

| Archivo | Líneas | Función |
|---------|--------|---------|
| `goals.py` | 297 | `Goal` dataclass, `GoalType` enum, `GoalState` enum, `GoalCondition` types |
| `goal_manager.py` | 354 | CRUD de metas, máquina de estados, persistencia |
| `plans.py` | 231 | `PersistentPlan` + `PersistentPlanStep` con DAG dependencies |
| `plan_executor.py` | 319 | Ejecución de planes paso a paso con retry policies |
| `priorities.py` | 198 | Priority Engine — 7 factores ponderados |
| `rewards.py` | 240 | Reward Model — 7 componentes de señal |
| `conflicts.py` | 197 | Detección y resolución de conflictos entre metas |
| `background.py` | 106 | `GoalReevaluationLoop` — ciclo background 30min |
| `tel_governance.py` | 206 | Governance checks antes de activar/ejecutar metas |
| `tel_middleware.py` | 112 | Middleware hooks en el [pipeline](pipeline.md) |

---

## Jerarquía de Metas

| Tipo | Fuente | Duración | Cancelación |
|------|--------|----------|-------------|
| **MISSION** | Solo principal | Indefinida | Requiere confirmación humana |
| **STRATEGIC** | Principal o MISSION | Semanas-meses | Requiere confirmación humana |
| **OPERATIONAL** | Cualquier nivel superior | Días-semanas | Automática si padre se cancela |
| **TACTICAL** | Sistema o usuario | Horas-días | Automática al completarse o expirar |

---

## Máquina de Estados

```
CREATED → ACTIVE → COMPLETED
                 → FAILED
                 → CANCELLED
          ACTIVE ↔ PAUSED
          ACTIVE ↔ BLOCKED (dependencia no resuelta)
```

`GoalCondition` types: `metric_threshold`, `event_occurred`, `time_elapsed`, `memory_contains`, `human_confirmed`.

---

## Motor de Prioridades (7 factores)

| Factor | Peso | Descripción |
|--------|------|-------------|
| `base_priority` | 0.25 | Prioridad inherente del tipo (MISSION > TACTICAL) |
| `urgency` | 0.20 | Proximidad a deadline |
| `identity_alignment` | 0.15 | Alineación con [IdentityProfile](identity.md) |
| `dependency_pressure` | 0.15 | Cuántas metas dependen de esta |
| `progress_momentum` | 0.10 | Progreso reciente |
| `latency_penalty` | 0.10 | Tiempo sin progreso (MISSION=∞, STRATEGIC=30d, OPERATIONAL=14d, TACTICAL=3d) |
| `context_relevance` | 0.05 | Relevancia al contexto actual |

---

## Modelo de Recompensas (7 componentes)

Señal de recompensa: valor entre **-1.0** y **1.0** por interacción.

| Componente | Descripción |
|-----------|-------------|
| `goal_progress` | ¿Avanzó hacia alguna meta? |
| `quality_score` | Score de [Quality Scorer](evaluation.md) |
| `identity_fidelity` | Fidelidad al perfil de identidad |
| `governance_compliance` | Cumplimiento de reglas de gobernanza |
| `efficiency` | Eficiencia en uso de recursos/tokens |
| `novelty` | ¿Aprendió algo nuevo? |
| `user_satisfaction` | Señales de satisfacción del usuario |

Almacenado en tabla [Postgres](database.md) `reward_signals`, rolling 500 interacciones.

---

## Resolución de Conflictos

6 tipos de conflicto × 5 estrategias en cascada:

**Tipos**: resource, value, temporal, dependency, priority, identity.

**Estrategias** (en orden):
1. Type dominance (MISSION > STRATEGIC > OPERATIONAL > TACTICAL)
2. Priority dominance (score mayor gana)
3. Identity dominance (más alineado con identidad gana)
4. Temporal pause (pausa el menos urgente)
5. Escalation (envía a principal para decisión)

---

## Gobernanza de Metas

4 checks antes de activar o ejecutar:

| Check | Criterio |
|-------|----------|
| Identity Alignment | < 0.40 → reject, 0.40-0.60 → flag, > 0.60 → approve |
| Forbidden Operations | Verificación contra [governance.yaml](../config/README.md) forbidden list |
| Value Consistency | Consistencia con valores del principal |
| Drift Protection | Protección contra drift de identidad |

---

## Background Loop

`GoalReevaluationLoop` — cada **30 minutos**:

- Max 1 ejecución concurrente
- Timeout 60s
- Max 3 acciones por ciclo
- Respeta emergency stop
- Requiere `autonomy_level ≥ 2` (actualmente Level 0 → **solo observa**)

---

## Integración con Pipeline

| Step | Nombre | Posición de Middleware |
|------|--------|----------------------|
| 0.3 | `GoalContextInjector` | PRE_CLASSIFY — inyecta contexto de metas activas |
| 10b | `RewardComputer` | POST_STORE — calcula señal de recompensa |
| 10c | `GoalProgressUpdater` | POST_STORE — actualiza progreso de metas |

Ver: [Pipeline](pipeline.md), [Middleware](pipeline.md#middleware-pipeline)

---

## Gestión desde Dashboard

**Página**: [Goals](../dashboard/goals.md) (`/goals`)

3 tabs: Goals / Conflicts / Rewards.

Acciones: Create, Activate, Complete, Cancel, Delete goals.

**API**: 11 endpoints en `/goals/*` → ver [API](../api/README.md)

---

## Temas Relacionados

- [Pipeline](pipeline.md) — Middleware hooks (pasos 0.3, 10b, 10c)
- [Identidad](identity.md) — identity_alignment en prioridades, identidad en conflictos
- [Evaluación](evaluation.md) — Quality score alimenta rewards
- [Gobernanza](../dashboard/governance-console.md) — Emergency stop respetado
- [Goals page](../dashboard/goals.md) — Interfaz de gestión
- [Base de Datos](database.md) — Tablas goals, plans, reward_signals, goal_events
