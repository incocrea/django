# Eventos & Trace Cognitivo

> [← Evaluación](evaluation.md) · [Seguridad →](security.md)

---

## Sobre este documento

Este documento describe el **sistema nervioso** de Django — los dos mecanismos que hacen visible y auditable todo lo que pasa internamente. El `EventBus` transmite lo que ocurre en tiempo real (como un sistema de notificaciones), y el `TraceCollector` graba cada paso del procesamiento como un grafo visual (como una cámara que filma el pensamiento de Django).

### ¿Qué cubre este documento?

Documenta el `EventBus` (151 líneas), su mecanismo de broadcast simultáneo a 3 destinos, el esquema `IAmeEvent` con sus 7 campos, los 16+ tipos de eventos, el `TraceCollector` (407 líneas) que genera grafos compatibles con React Flow, el `TraceStore` (100 trazas en memoria), y los 32 tipos de nodos de traza que representan cada paso del pipeline.

### ¿Cuál es su función en la arquitectura?

Los eventos y trazas son el **sistema de observabilidad**. Sin ellos, lo que hace Django internamente sería una caja negra. Gracias a events y trace, Harold puede:
- **Ver en tiempo real** qué está haciendo Django (via WebSocket al Dashboard)
- **Auditar** qué hizo en el pasado (via audit_log en Postgres)
- **Visualizar** exactamente cómo pensó Django cada respuesta (via el grafo de traza cognitiva en React Flow)

### ¿Cómo afecta al comportamiento de Django?

Estos sistemas no afectan *qué* responde Django — son puramente observacionales. Pero afectan profundamente la **experiencia de Harold**:
- El Command Center del dashboard se actualiza en tiempo real gracias a los eventos WebSocket
- Los LEDs de estado de agentes cambian de color según los eventos `agent_state.*`
- El feed de actividad muestra cada evento importante
- La página de Cognitive Trace renderiza el grafo completo de procesamiento de cualquier interacción

### ¿Cómo interactúa con las demás piezas?

Events y Trace son **consumidos y producidos por casi todo el sistema**:
- **[Pipeline](pipeline.md)**: cada uno de los 25+ pasos emite al menos un evento y crea al menos un nodo de traza
- **[Identidad](identity.md)**: los 17 pasos de identidad emiten eventos propios (`identity.drift_checked`, `identity.confidence_computed`, `identity.evolution_analyzed`, etc.) y generan 17 tipos de nodo de traza específicos
- **[Evaluación](evaluation.md)**: emite nodos de tipo `evaluation` después de cada scoring
- **[Teleología](teleology.md)**: emite eventos de ciclo de vida de metas
- **[Gobernanza](../dashboard/governance-console.md)**: emite eventos críticos como `governance.emergency_stop_activated`
- **[Base de Datos](database.md)**: los eventos se persisten en la tabla `audit_log` de Postgres; los nodos de traza en la tabla `trace_nodes`
- **[Dashboard](../dashboard/README.md)**: el Command Center, Chat, y otros componentes consumen eventos via WebSocket en tiempo real
- **[Discord Bot](../integrations/README.md)**: los eventos de chat se emiten igual que para interacciones de dashboard
- **[Startup](startup.md)**: el primer evento del sistema es `system.startup`; el EventBus se inicializa temprano en la secuencia
- **[API](../api/README.md)**: endpoints de trace (6) y events/recent exponen los datos al frontend

---

## EventBus — `src/events/event_bus.py` (151 líneas)

### Broadcast Simultáneo

Cada `IAmeEvent` se emite a 3 destinos en paralelo:

| Destino | Descripción |
|---------|-------------|
| **WebSocket** | Todas las conexiones WS activas → Dashboard real-time updates |
| **Postgres** | Tabla `audit_log` → [Base de Datos](database.md) |
| **In-memory subscribers** | Callbacks registrados vía `subscribe()` |

### IAmeEvent

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `event_type` | str | E.g., `chat.response_generated`, `identity.drift_checked` |
| `data` | dict | Payload del evento |
| `risk_level` | str | low / medium / high / critical |
| `timestamp` | datetime | Momento de emisión |

### Buffer

Últimos **100 eventos** en memoria. Accesible via `GET /events/recent` → ver [API](../api/README.md).

### Eventos Principales

| Tipo | Cuándo | Risk |
|------|--------|------|
| `system.startup` | Backend inicia | low |
| `chat.message_received` | Nuevo mensaje de usuario | low |
| `chat.response_generated` | Respuesta lista | low |
| `agent_state.thinking/acting/idle` | Cambio de estado del agente | low |
| `config.*_updated/reloaded` | Cambio de configuración | low |
| `identity.drift_checked` | Enforcement evalúa drift | low-high |
| `identity.policy_evaluated` | Policy clasifica severidad | low-high |
| `identity.health_evaluated` | Health monitor evalúa | low |
| `identity.evolution_analyzed` | Evolución propuesta | medium |
| `identity.shadow_simulated` | Shadow sim completada | medium |
| `identity.version_candidate_created` | Candidato de versión | medium |
| `identity.version_activated` | Versión activada | medium |
| `identity.version_rolled_back` | Rollback ejecutado | high |
| `governance.emergency_stop_activated` | **KILL SWITCH** | **critical** |
| `governance.emergency_stop_deactivated` | Resume operaciones | medium |
| `security.threat_detected` | Amenaza de seguridad detectada | high |

---

## Trace Cognitivo — `src/trace/` (401 líneas)

### TraceCollector — `collector.py` (342 líneas)

Cada interacción genera un grafo [React Flow](https://reactflow.dev/) compatible:

- **Per-interaction** — un TraceCollector nuevo por cada `orchestrator.process()` call
- **Auto-layout** — posiciona nodos automáticamente
- `TraceNode` dataclass incluye `persist_id`, `model_used`, `risk_score`

### 16 Tipos de Nodo

| Tipo | Color | Paso del Pipeline |
|------|-------|-------------------|
| `input` | blue | Mensaje del usuario |
| `classify` | purple | Paso 1 — clasificación |
| `decision_engine` | indigo | Paso 2 — DecisionEngine |
| `planner` | cyan | Paso 3 — Planner |
| `memory_recall` | teal | Paso 4 — recall de memoria |
| `correction` | yellow | Paso 5 — injection de correcciones |
| `context_weighting` | — | Paso 3b — identity context tags |
| `prompt_build` | orange | Paso 6 — construcción del prompt |
| `llm_generate` | green | Paso 7 — generación LLM |
| `identity_review` | pink | Paso 8 — revisión de identidad |
| `governance_review` | amber | Paso 9 — revisión de gobernanza |
| `memory_store` | emerald | Paso 10 — almacenamiento |
| `evaluation` | violet | Paso 10 — evaluaciones |
| `output` | sky | Respuesta final |
| `branch` | — | Bifurcación en pipeline |
| `multi_agent` | rose | Ejecución multi-agente |

### TraceStore

Singleton — últimas **100 trazas** en memoria. Respaldado en [Postgres](database.md) tablas `interactions` + `trace_nodes`.

---

## Visualización en Dashboard

Página: [Cognitive Trace](../dashboard/cognitive-trace.md) (`/trace`)

- Canvas React Flow con zoom, pan, minimap
- Nodos expandibles: input/output/metrics/errors por paso
- Prompt viewer modal (user + system prompts)
- Lista de trazas con click-to-load
- Replay: re-procesa un mensaje por pipeline completo

**API**: 6 endpoints en `/trace/*` → ver [API](../api/README.md)

---

## Integración con WebSocket

```
EventBus.emit(event)
  ├── WebSocket broadcast → Dashboard
  │     ├── ActivityFeed (Command Center) → muestra timeline
  │     ├── AgentStatusRing → actualiza estado visual
  │     └── HealthBar → refleja cambios
  ├── Postgres audit_log → persiste
  └── In-memory subscribers → callbacks
```

El Dashboard se conecta via `useWebSocket()` hook → `ws://localhost:8000/api/ws`. Auto-reconnect (3s), keepalive (30s).

---

## Temas Relacionados

- [Pipeline](pipeline.md) — Cada paso emite eventos y crea trace nodes
- [Base de Datos](database.md) — audit_log, trace_nodes, interactions
- [Cognitive Trace](../dashboard/cognitive-trace.md) — Visualización React Flow
- [Command Center](../dashboard/command-center.md) — ActivityFeed, AgentStatusRing, HealthBar
- [Gobernanza](../dashboard/governance-console.md) — Audit log viewer
- [Analytics](../dashboard/analytics.md) — Events by type/agent/risk
