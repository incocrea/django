# Eventos & Trace Cognitivo

> [← Evaluación](evaluation.md) · [Seguridad →](security.md)

---

## Sobre este documento

Este documento describe el **sistema nervioso** de Doe — los dos mecanismos que hacen visible y auditable todo lo que pasa internamente. El `EventBus` transmite lo que ocurre en tiempo real (como un sistema de notificaciones), y el `TraceCollector` graba cada paso del procesamiento como un grafo visual (como una cámara que filma el pensamiento de Doe).

### ¿Qué cubre este documento?

Documenta el `EventBus` (151 líneas), su mecanismo de broadcast simultáneo a 3 destinos, el esquema `IAmeEvent` con sus 7 campos, los 16+ tipos de eventos, el `TraceCollector` (562 líneas) que genera grafos con 8 sub-flow groups, el `TraceStore` (100 trazas en memoria), y los 33 tipos de nodos de traza organizados en bloques funcionales.

### ¿Cuál es su función en la arquitectura?

Los eventos y trazas son el **sistema de observabilidad**. Sin ellos, lo que hace Doe internamente sería una caja negra. Gracias a events y trace, Harold puede:
- **Ver en tiempo real** qué está haciendo Doe (via WebSocket al Dashboard)
- **Auditar** qué hizo en el pasado (via audit_log en Postgres)
- **Visualizar** exactamente cómo pensó Doe cada respuesta (via el pipeline horizontal CSS Grid en la página `/trace`)

### ¿Cómo afecta al comportamiento de Doe?

Estos sistemas no afectan *qué* responde Doe — son puramente observacionales. Pero afectan profundamente la **experiencia de Harold**:
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

## Trace Cognitivo — `src/trace/` (~562 líneas)

### TraceCollector — `collector.py` (562 líneas)

Cada interacción genera un grafo dirigido con sub-flow groups:

- **Per-interaction** — un TraceCollector nuevo por cada `orchestrator.process()` call
- **Layout horizontal** — 8 sub-flow groups renderizados como columnas CSS Grid en el frontend
- `TraceNode` dataclass incluye `persist_id`, `model_used`, `risk_score`, `uses_llm`, `uses_embedding`, `is_skill_step`, `parent_node_id`
- **Sub-flow groups**: `_NODE_TYPE_GROUP` (33 mappings) + `_NODE_ID_GROUP` (overrides) + `_GROUP_META` (8 grupos con label/color/order)

### 8 Sub-Flow Groups

| # | Grupo | Color | Nodos |
|---|-------|-------|-------|
| 0 | Pre-Pipeline | #6366f1 (indigo) | input, branch |
| 1 | Classification | #06b6d4 (cyan) | classify, decision_engine |
| 2 | Identity Pre-Gen | #a855f7 (purple) | identity_decision_modulation, identity_confidence, identity_autonomy, identity_bias |
| 3 | Planning & Skills | #f59e0b (amber) | planner, skill_execution |
| 4 | Context Assembly | #10b981 (emerald) | memory_recall, identity_memory_bridge, identity_retrieval_weight, correction, identity_context_weight, identity_prompt_inject, prompt_build, compaction |
| 5 | LLM Generation | #ef4444 (red) | llm_generate, multi_agent, identity_review, governance_review |
| 6 | Persistence & Eval | #3b82f6 (blue) | identity_consolidation, memory_store, evaluation, identity_drift, identity_policy, identity_feedback, output |
| 7 | Post-Pipeline | #ec4899 (pink) | identity_health_monitor, identity_health_regulation, identity_evolution, identity_shadow, identity_version_candidate |

### 33 Tipos de Nodo

| Tipo | Color | Paso del Pipeline |
|------|-------|-------------------|
| `input` | blue | Mensaje del usuario |
| `classify` | purple | Paso 1 — clasificación |
| `decision_engine` | orange | Paso 2 — DecisionEngine |
| `planner` | sky | Paso 3 — Planner |
| `skill_execution` | amber | Paso 1d — ejecución de skills (learn-topic, repo-explorer) |
| `memory_recall` | cyan | Paso 4 — recall de memoria |
| `correction` | amber | Paso 5 — injection de correcciones |
| `prompt_build` | indigo | Paso 6 — construcción del prompt |
| `llm_generate` | emerald | Paso 7 — generación LLM |
| `identity_review` | rose | Paso 8 — revisión de identidad |
| `governance_review` | yellow | Paso 9 — revisión de gobernanza |
| `memory_store` | teal | Paso 10 — almacenamiento |
| `evaluation` | violet | Paso 10 — evaluaciones |
| `output` | green | Respuesta final |
| `branch` | red | Bifurcación en pipeline |
| `multi_agent` | fuchsia | Ejecución multi-agente |
| `compaction` | slate | Compactación de working memory |
| `identity_decision_modulation` | pink | Paso 3c (Phase 6C) |
| `identity_confidence` | pink | Paso 3d (Phase 6D) |
| `identity_autonomy` | pink | Paso 3e (Phase 7A) |
| `identity_bias` | fuchsia | Paso 3f (Phase 8A) |
| `identity_memory_bridge` | rose | Paso 2b (Phase 6A) |
| `identity_retrieval_weight` | rose | Paso 2c (Phase 7B) |
| `identity_context_weight` | pink | Paso 3b (Phase 6B) |
| `identity_prompt_inject` | fuchsia | Paso 3g (Phase 8B) |
| `identity_consolidation` | pink | Paso 6d (Phase 7C) |
| `identity_drift` | red | Paso 7c (Phase 5A) |
| `identity_policy` | red | Paso 7d (Phase 5B) |
| `identity_feedback` | orange | Paso 7e (Phase 5C) |
| `identity_health_monitor` | lime | Paso 9a (Phase 9A) |
| `identity_health_regulation` | lime | Paso 9b (Phase 9B) |
| `identity_evolution` | amber | Paso 9c (Phase 10A) |
| `identity_shadow` | slate | Paso 9d (Phase 10B) |
| `identity_version_candidate` | emerald | Paso 9e (Phase 10C) |

### TraceStore

Singleton — últimas **100 trazas** en memoria. Respaldado en [Postgres](database.md) tablas `interactions` + `trace_nodes`.

---

## Visualización en Dashboard

Página: [Cognitive Trace](../dashboard/cognitive-trace.md) (`/trace`)

- Pipeline horizontal CSS Grid con 8 columnas (accordion nodes)
- **🧠 LLM badge** — nodos que usan LLM marcados con badge violeta
- Nodos expandibles: input/output/metrics/errors por paso
- **LLM Details panel** — badges de provider (Gemini/Groq), tokens, latencia, costo estimado
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
- [Cognitive Trace](../dashboard/cognitive-trace.md) — Pipeline horizontal CSS Grid
- [Command Center](../dashboard/command-center.md) — ActivityFeed, AgentStatusRing, HealthBar
- [Gobernanza](../dashboard/governance-console.md) — Audit log viewer
- [Analytics](../dashboard/analytics.md) — Events by type/agent/risk
