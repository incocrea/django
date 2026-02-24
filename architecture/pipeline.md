# Pipeline del Orchestrator

> [← Arquitectura](README.md) · [Agentes →](agents.md)

**Archivo**: `src/flows/orchestrator.py` (2,616 líneas)

---

## Sobre este documento

Este documento describe el **corazón de Django** — el pipeline del Orchestrator. Cada vez que alguien le habla a Django (por chat, Discord, o API), el mensaje pasa por una secuencia de 25+ pasos antes de que Django responda. Este documento detalla exactamente qué hace cada paso, en qué orden se ejecutan, y qué decide cada uno.

### ¿Qué cubre este documento?

Documenta los **3 modos cognitivos** (Full, Memory+LLM, Memory Only), la **tabla completa de pasos del pipeline** (desde la clasificación inicial hasta la versión candidata de identidad), el **sistema de middleware** (9 hooks que módulos externos pueden usar para inyectar lógica), el **ejecutor paralelo** (que corre evaluaciones simultáneamente), y el **flujo end-to-end** de un mensaje desde que el endpoint lo recibe hasta que la respuesta se envía al usuario.

### ¿Cuál es su función en la arquitectura?

El Orchestrator es el **director de orquesta** — el módulo central que coordina a todos los demás. No genera respuestas por sí mismo, no almacena memorias, no evalúa calidad. Lo que hace es **invocar a cada subsistema en el orden correcto** y pasar los resultados de uno al siguiente. Es la columna vertebral que le da estructura al pensamiento de Django.

### ¿Cómo afecta al comportamiento de Django?

**Totalmente**. El pipeline determina:
- **Qué tan inteligente es Django**: en modo Full (1) usa web + LLM + toda la memoria; en modo Memory+LLM (2, default) se limita a conocimiento aprendido; en modo Memory Only (3) ni siquiera usa LLM
- **Qué tan seguro es**: los pasos de gobernanza y revisión de identidad pueden señalar respuestas riesgosas
- **Qué tan fiel a Harold es**: 10+ pasos de identidad (modulación de decisiones, confianza, sesgo conductual, integración de prompt, etc.) trabajan para que cada respuesta suene como Harold
- **Qué tan buena memoria tiene**: los pasos de recall, re-ranking y consolidación determinan qué recuerda y con qué prioridad

### ¿Cómo interactúa con las demás piezas?

El pipeline es donde **todo converge**. Cada paso invoca a un módulo diferente:
- **Paso 1** → [Cognición](cognition.md): clasifica el mensaje y el `DecisionEngine` decide estrategia/riesgo sin LLM
- **Pasos 3c-3g** → [Identidad](identity.md): 5 sub-pasos de modulación, confianza, autonomía, sesgo conductual y prompt injection
- **Paso 4** → [Memoria](memory.md): recall de los 4 niveles (working, episodic, semantic, procedural) con filtro por modo cognitivo
- **Pasos 2b-2c** → [Identidad](identity.md): memory bridge + retrieval re-ranking basado en afinidad de identidad
- **Paso 7** → [Agentes](agents.md): el agente seleccionado genera la respuesta via el [Model Router](../integrations/README.md)
- **Pasos 8-9** → [Agentes](agents.md): IdentityCore revisa y GovernanceAgent evalúa cumplimiento
- **Paso 10** → [Evaluación](evaluation.md): 5 módulos heurísticos califican la respuesta en paralelo
- **Paso 10** → [Base de Datos](database.md): persistencia de interacciones, trazas, evaluaciones y tokens
- **Pasos 9a-9e** → [Identidad](identity.md): post-pipeline — health monitor, regulation, evolution, shadow sim, version candidate
- Cada paso emite eventos via el [EventBus](events.md) y genera nodos para el [TraceCollector](events.md)
- Los hooks de [Teleología](teleology.md) se inyectan en posiciones PRE_CLASSIFY, POST_CLASSIFY y POST_STORE
- La [Seguridad](security.md) sanitiza inputs antes de que entren al pipeline

---

## Resumen

Pipeline central de 25+ pasos que procesa CADA mensaje. Soporta 3 Modos Cognitivos:

| Modo | Nombre | Comportamiento |
|------|--------|---------------|
| 1 | Full 🟢 | Web + LLM + toda la memoria, sin restricciones |
| 2 | Memory+LLM 🟡 | LLM genera pero grounded en memoria recalled. Filtro semántico a `learned_knowledge`. **Default** |
| 3 | Memory Only 🔴 | NO LLM. Solo recall de memoria, retorno directo |

---

## Pasos del Pipeline

| Step | Nombre | Módulo | Descripción |
|------|--------|--------|-------------|
| 0 | Emergency Check | orchestrator | Bloquea si `_emergency_stopped` → ver [governance](../dashboard/governance-console.md) |
| 0.3 | Goal Context Injector | [teleology middleware](teleology.md) | Inyecta metas activas (PRE_CLASSIFY hook) |
| 1 | Semantic Classify | [semantic_classifier](cognition.md) | Clasificación semántica por centroides — embeddings Qwen3-Embedding-8B via EmbeddingRouter, 9 categorías, 360 training phrases. Fallback a CONVERSATION si confidence < 0.30 |
| 1d | Skill Execution | [skills](../dashboard/skill-manager.md) | Si Planner incluye `skill_execution`: learn-topic (mode 1) o repo-explorer (modes 1-2). Contexto inyectado en pipeline, no bypass |
| 2 | Decision Engine | [cognition](cognition.md) | Deterministic strategy/risk/agent (no LLM) |
| 3c | Identity Decision Modulation | [identity](identity.md) | Evalúa alignment decision-identity (observational) |
| 3d | Identity Confidence | [identity](identity.md) | Agrega señales → confidence 0-1 + autonomy_modifier |
| 3e | Autonomy Modulation | [identity](identity.md) | Ajusta governance threshold según confidence |
| 3f | Behavioral Bias | [identity](identity.md) | Genera style_bias + planner_mode recommendation |
| 3g | Prompt Integration | [identity](identity.md) | Inyecta `[IDENTITY STYLE PREFERENCES]` en system prompt |
| 3 | Planner | [cognition](cognition.md) | Builds step-by-step Plan con gate flags (no LLM) |
| 4 | Memory Recall + Mode Filter | [memory](memory.md) | ChromaDB search + working memory. Mode 2-3: filtra semantic a `learned_knowledge` |
| 2b | Identity Memory Bridge | [identity](identity.md) | Cosine similarity per memory vs baseline |
| 2c | Identity Retrieval Weighting | [identity](identity.md) | Re-rank 0.8×semantic + 0.2×identity |
| 5 | Correction Injection | [training](../dashboard/training-center.md) | Behavioral suggestions from procedural memory |
| 3b | Identity Context Weighting | [identity](identity.md) | Tags `[ALIGNED]`/`[LOW_ALIGNMENT]` en contexto |
| 6 | Prompt Build + Mode Logic | orchestrator | Mode 3: return memory-only. Mode 2: knowledge headers |
| 7 | LLM Generate | [agents](agents.md) via [Model Router](../integrations/README.md) | Route to agent → provider API |
| 8 | Identity Review | [agents](agents.md) | Per plan: non-conversation reviewed by Identity Core |
| 9 | Governance Review | [agents](agents.md) | Per plan: reviewed by GovernanceAgent (JSON) |
| 6d | Consolidation Weighting | [identity](identity.md) | Ajusta importance pre-storage |
| 10 | Memory Store | [memory](memory.md) + [evaluation](evaluation.md) | Save working + episodic + 5 evaluaciones + [Postgres persist](database.md) |
| 10b | Reward Computer | [teleology middleware](teleology.md) | Calcula señal de recompensa (POST_STORE hook) |
| 10c | Goal Progress Updater | [teleology middleware](teleology.md) | Actualiza progreso de metas |
| 7c | Identity Enforcement | [identity](identity.md) | Evalúa drift vs threshold |
| 7d | Identity Policy | [identity](identity.md) | Clasifica severidad, determina acción |
| 7e | Identity Feedback | [identity](identity.md) | Genera correction hints si rewrite_request+high |
| 9a | Health Monitor | [identity](identity.md) | Monitoreo longitudinal (sliding window 50) |
| 9b | Health Regulation | [identity](identity.md) | Ajuste adaptativo post-health |
| 9c | Evolution Analysis | [identity](identity.md) | Propuesta de evolución (200 interacciones, OLS regression) |
| 9d | Shadow Simulation | [identity](identity.md) | Simulación paralela no-mutante |
| 9e | Version Control | [identity](identity.md) | Candidato de versión (proposal only) |

Cada paso emite eventos via [EventBus](events.md) y crea nodos via [TraceCollector](events.md).

---

## Middleware Pipeline

**Archivo**: `src/flows/middleware.py` (196 líneas)

`MiddlewareRegistry` con `MiddlewarePosition` enum (9 hooks). Permite inyectar lógica en puntos del pipeline sin modificar el orchestrator.

Hooks disponibles: PRE_CLASSIFY, POST_CLASSIFY, PRE_MEMORY, POST_MEMORY, PRE_GENERATE, POST_GENERATE, PRE_STORE, POST_STORE, POST_RESPONSE.

Dashboard: [Governance Console → Middleware tab](../dashboard/governance-console.md)
API: `GET /middleware/list`, `POST /middleware/{name}/toggle` → ver [API](../api/README.md)

---

## Ejecución Paralela

**Archivo**: `src/flows/parallel_executor.py` (204 líneas)

`ParallelExecutor` — 2+ agentes concurrentes vía `asyncio.gather`. `ResponseAggregator` fusiona resultados. Se activa cuando `Strategy.MULTI_AGENT` (ver [cognition](cognition.md)).

---

## Flujo End-to-End de un Mensaje de Chat

```
1. UI → api.chat() → POST /api/chat
2. InputSanitizer().sanitize()                    → security.md
3. database.create_conversation()                 → database.md (conversations)
4. database.save_message(user)                    → database.md (messages)
5. Emit agent_state.thinking → WebSocket          → events.md
6. database.get_conversation_messages()            → últimos 20
7. Emit agent_state.acting → WebSocket
8. orchestrator.process(message, conv_id, history, mode)
   └── 25+ pasos del pipeline (tabla arriba)
9. database.save_message(assistant)
10. Emit chat.response_generated → WebSocket
11. Emit agent_state.idle → WebSocket
12. Return ChatResponse JSON → Dashboard
```

Ver: [Chat page](../dashboard/chat.md) para la experiencia de usuario completa.

---

## Temas Relacionados

- [Cognición](cognition.md) — DecisionEngine y Planner que alimentan el pipeline
- [Memoria](memory.md) — Los 4 tiers que se consultan en paso 4
- [Identidad](identity.md) — Los 17 pasos de identidad integrados
- [Evaluación](evaluation.md) — Los 5 módulos que corren en paso 10
- [Teleología](teleology.md) — Los 3 middleware hooks del sistema de metas
- [Chat](../dashboard/chat.md) — La interfaz de usuario que dispara el pipeline
- [Cognitive Trace](../dashboard/cognitive-trace.md) — Visualización del pipeline como grafo
