# Desarrollo

> [← Integraciones](../integrations/README.md) · [Índice ↑](../README.md)

---

## Sobre este documento

Este documento es la **guía práctica para desarrolladores** (y para Copilot) — define las convenciones de código, las reglas operacionales críticas, el catálogo completo de tests, y el roadmap de funcionalidades pendientes. Si los documentos de arquitectura explican *qué hace* cada pieza del sistema, este documento explica *cómo trabajar* con el sistema de forma segura y consistente.

### ¿Qué cubre este documento?

Documenta las **convenciones de código** para Python (backend) y TypeScript (dashboard), el **tema visual CSS** con todas sus variables, las **3 reglas operacionales críticas** (servicios solo via Tasks, terminales se cierran después de usar, Discord bot nunca duplicar), el **catálogo completo de tests** (50 archivos, ~17,100 líneas con pytest), el **roadmap por módulo** (qué está pendiente y su prioridad), los **niveles de autonomía** (0-4) y las **integraciones externas planificadas**.

### ¿Cuál es su función en la arquitectura?

Este no es un módulo de la arquitectura sino una **meta-documentación operacional**. Define las reglas del juego para construir sobre el sistema existente. Es especialmente crítico porque ADLRA es un proyecto que se auto-modifica (con supervisión humana) — sin reglas claras, los cambios podrían romper la cadena de confianza entre Django, el código y Harold.

### ¿Cómo afecta al comportamiento de Django?

Indirectamente pero de forma importante:
- Las **convenciones de código** aseguran que nuevos módulos se integren correctamente (lazy imports, logger naming, Pydantic validation)
- Los **tests** validan que cada módulo de [identidad](../architecture/identity.md), [pipeline](../architecture/pipeline.md), [memoria](../architecture/memory.md), [evaluación](../architecture/evaluation.md), [cognición](../architecture/cognition.md), [seguridad](../architecture/security.md) y [teleología](../architecture/teleology.md) funcione correctamente antes de deployar
- Las **reglas operacionales** previenen problemas como servicios duplicados (que causan double responses en Discord) o terminales zombie que consumen recursos

### ¿Cómo interactúa con las demás piezas?

Este documento **referencia a todos los demás** porque cubre el proyecto completo:
- La tabla de tests enlaza a los módulos que testea: [Pipeline](../architecture/pipeline.md), [Identidad](../architecture/identity.md) (17 archivos de test para las 17 fases), [Memoria](../architecture/memory.md), [Evaluación](../architecture/evaluation.md), [Cognición](../architecture/cognition.md), [Eventos](../architecture/events.md), [Seguridad](../architecture/security.md), [Teleología](../architecture/teleology.md), [Database](../architecture/database.md)
- El roadmap referencia módulos pendientes en [Pipeline](../architecture/pipeline.md) (Human-in-the-Loop), [Dashboard](../dashboard/README.md) (múltiples mejoras), [Config](../config/README.md) (auth), [Integraciones](../integrations/README.md) (email, calendar, Slack)
- Las reglas operacionales conectan con [Config](../config/README.md) (tasks.json) y [Integraciones](../integrations/README.md) (Discord bot PID management)

---

## Convenciones de Código

### Python (Backend)

| Convención | Ejemplo |
|-----------|---------|
| Lazy imports en handlers | `from src.api.main import get_state` |
| Logger | `logging.getLogger("iame.module_name")` |
| Endpoints async | `async def handler():` |
| Validación | Pydantic models para request/response |
| Paths | Constantes relativas a `__file__` |
| DB | psycopg2 sync con autocommit |

### TypeScript (Dashboard)

| Convención | Ejemplo |
|-----------|---------|
| Client directive | `"use client"` en todas las páginas interactivas |
| Estado global | Zustand store (`lib/store.ts`) |
| Estado local | `useState` / `useReducer` |
| API | Centralizada en `lib/api.ts` (~111 métodos) |
| UI | shadcn/ui primitives + custom lab theme |
| i18n | `lib/i18n/` — `en.json` + `es.json` (631 keys) |

### Tema Visual

| Variable CSS | Valor | Uso |
|-------------|-------|-----|
| `lab-bg` | `#0a0a0f` | Fondo principal |
| `lab-surface` | `#12121a` | Superficies elevadas |
| `lab-card` | `#1a1a2e` | Cards |
| `lab-border` | `#2a2a3e` | Bordes |
| `lab-text` | `#e0e0e8` | Texto principal |
| `lab-text-dim` | `#8888a0` | Texto secundario |
| `accent-primary` | `#6366f1` | Indigo primario |
| `accent-glow` | `#818cf8` | Glow effects |
| `status-green` | `#22c55e` | OK/Online |
| `status-amber` | `#f59e0b` | Warning |
| `status-red` | `#ef4444` | Error/Critical |
| `status-blue` | `#3b82f6` | Info |

---

## Reglas Operacionales Críticas

### Servicios = SOLO Tasks

| ✅ Correcto | ❌ Prohibido |
|------------|-------------|
| `run_task("Start All Services")` | `run_in_terminal` con `isBackground: true` para servicios |
| `run_task("Backend: FastAPI Server")` | Ejecutar `uvicorn` en terminal |
| `run_task("Discord: Django Bot")` | Ejecutar `python discord_bot.py` manualmente |

Las Tasks tienen `instanceLimit: 1` — re-ejecutar mata la instancia anterior.

### Terminales = Se cierran después de usar

| Escenario | Acción |
|----------|--------|
| `curl`, `pip install` | Reuse foreground terminal |
| One-off test | Kill después de resultado |
| Background rápido | Kill inmediatamente |
| Task de servicio | Keep alive (VS Code las gestiona) |

### Bot Discord = NUNCA duplicar

- PID lock en `agent/discord_bot.pid`
- NUNCA WMIC scans ni `os.kill()`
- Si duplicación: `wmic process where "name='python.exe'" get processid,commandline`

---

## Testing

### Framework

- **pytest** desde `agent/tests/`
- **conftest.py** — fixtures compartidas (mocks, configs, event loops)
- **No LLM calls** — todos los tests usan mocks
- **Patrón**: `test_{module}.py` con clases `Test{Feature}`

### Archivos de Test (50 archivos, ~17,100 líneas)

| Archivo | Líneas | Módulo Testeado |
|---------|--------|-----------------|
| test_orchestrator.py | 1,735 | [Pipeline](../architecture/pipeline.md) completo |
| test_identity_behavioral_bias.py | 1,244 | [Identity](../architecture/identity.md) Phase 8A |
| test_identity_version_control.py | 1,213 | Identity Phase 10C |
| test_identity_shadow_simulation.py | 1,092 | Identity Phase 10B |
| test_identity_evolution.py | 1,084 | Identity Phase 10A |
| test_identity_health_regulation.py | 1,017 | Identity Phase 9B |
| test_identity_health_monitor.py | 823 | Identity Phase 9A |
| test_identity_consolidation_weighting.py | 815 | Identity Phase 7C |
| test_identity_confidence.py | 711 | Identity Phase 6D |
| test_identity_decision_modulation.py | 609 | Identity Phase 6C |
| test_identity_prompt_integration.py | 604 | Identity Phase 8B |
| test_identity_retrieval_weighting.py | 504 | Identity Phase 7B |
| test_identity_autonomy_modulation.py | 451 | Identity Phase 7A |
| test_identity_context_weighting.py | 430 | Identity Phase 6B |
| test_identity_policy.py | 425 | Identity Phase 5B |
| test_identity_memory_bridge.py | 425 | Identity Phase 6A |
| test_memory_manager.py | 400 | [Memoria](../architecture/memory.md) |
| test_memory_compaction.py | 395 | Memoria compactación |
| test_identity_feedback.py | 390 | Identity Phase 5C |
| test_identity_enforcement.py | 350 | Identity Phase 5A |
| test_goal_manager.py | 330 | [Teleología](../architecture/teleology.md) |
| test_memory_hybrid_search.py | 325 | Memoria hybrid search |
| test_evaluation_quality.py | 300 | [Evaluación](../architecture/evaluation.md) quality |
| test_evaluation_alignment.py | 280 | Evaluación alignment |
| test_evaluation_legal.py | 275 | Evaluación legal risk |
| test_evaluation_rollback.py | 270 | Evaluación rollback |
| test_evaluation_decisions.py | 265 | Evaluación decisions |
| test_training_manager.py | 255 | Training system |
| test_identity_manager.py | 250 | Identity manager |
| test_identity_core.py | 245 | Identity schema + embedding |
| test_skill_registry.py | 235 | [Skills](../architecture/pipeline.md) |
| test_model_router.py | 230 | [Model Router](../integrations/README.md) |
| test_circuit_breaker.py | 225 | Circuit Breaker |
| test_persistence.py | 220 | [Database](../architecture/database.md) |
| test_web_research.py | 210 | Web research skill |
| test_learn_topic.py | 205 | Learn-topic skill |
| test_events.py | 200 | [EventBus](../architecture/events.md) |
| test_trace.py | 195 | [TraceCollector](../architecture/events.md) |
| test_trace_subflows.py | 548 | Trace sub-flow grouping + LLM metadata |
| test_embedding_router.py | ~180 | [EmbeddingRouter](../integrations/README.md) singleton + Qwen3 embedding |
| test_skill_report.py | ~160 | [SkillReport](../architecture/pipeline.md) + SkillStep + StepTimer |
| test_cognition.py | 190 | [Cognición](../architecture/cognition.md) |
| test_semantic_classifier.py | 186 | [Semantic Classifier](../architecture/cognition.md) |
| test_governance.py | 185 | [Governance](../dashboard/governance-console.md) |
| test_security.py | 180 | [Security](../architecture/security.md) |
| test_identity_governance_hardening.py | 175 | Phase 10D.1 |
| test_input_sanitizer.py | 170 | InputSanitizer |
| test_content_wrapper.py | 165 | ContentWrapper |
| test_watchdog.py | 155 | Watchdog |
| conftest.py | 150 | Fixtures compartidas |

### Ejecutar Tests

```bash
cd agent
.\venv\Scripts\Activate.ps1
pytest                      # todos
pytest tests/test_orchestrator.py  # uno
pytest -k "test_confidence" # por keyword
pytest --tb=short -q        # salida compacta
```

---

## Roadmap

### Niveles de Autonomía

| Nivel | Nombre | Estado | Capacidades |
|-------|--------|--------|-------------|
| 0 | Observer | ✅ Completado | Solo responder, no iniciar acciones |
| 1 | Suggestor | ✅ Completado | Sugerencias proactivas (training suggestions) |
| 2 | Contributor | ✅ Completado | Ejecutar skills, almacenar conocimiento |
| 3 | Collaborator | ✅ Estructura | Goals + procedural memory (loop inactivo, autonomy ≥ 2) |
| 4 | Trusted | 🔲 Planificado | Acciones externas, auto-mejora |

### Pendiente por Módulo

| Módulo | Ítem | Prioridad |
|--------|------|-----------|
| [Pipeline](../architecture/pipeline.md) | Human-in-the-Loop (pause + approval queue) | ALTA |
| Pipeline | Multi-Agent Collaboration (parallel agent calls) | MEDIA |
| Pipeline | LLM-Based Classification (beyond keywords) | BAJA |
| [Identity Studio](../dashboard/identity-studio.md) | Version Control UI (diff viewer, rollback) | ALTA |
| Identity Studio | Writing Sample Gallery (browse, tag, word cloud) | MEDIA |
| Identity Studio | Drift Gauge visualization | MEDIA |
| [Memory Lab](../dashboard/memory-lab.md) | Consolidation (episodic → semantic summarization) | ALTA |
| Memory Lab | Memory Map (2D/3D embeddings) | MEDIA |
| Memory Lab | Timeline (chronological, zoom, filters) | MEDIA |
| [Analytics](../dashboard/analytics.md) | Real Identity Fidelity (embedding-based, not heuristic) | CRÍTICA |
| Analytics | Decision Quality Heatmap | MEDIA |
| Analytics | System Reliability metrics | MEDIA |
| [Governance](../dashboard/governance-console.md) | Visual Boundary Editor | ALTA |
| Governance | Delegation Token System | MEDIA |
| Skills | Autonomous Skill Acquisition | MEDIA |
| Auth | Supabase Auth + JWT (API currently open) | ALTA |
| RAG | Document Chunking Pipeline (proper overlap) | ALTA |
| RAG | ~~Local Embeddings (Ollama mxbai-embed-large)~~ | ~~BAJA~~ | ✅ DONE — Migrated to Qwen3-Embedding-8B (4096-dim) via EmbeddingRouter + Ollama |
| Fine-tuning | QLoRA + Unsloth pipeline | BAJA |

### Integraciones Externas Planificadas

| Integración | Tipo | Estado |
|-------------|------|--------|
| Discord Bot | Conversación natural | ✅ Operativo |
| Discord Webhook | Publicación | ✅ Operativo |
| Email (send/receive) | Comunicación | 🔲 |
| Calendar (Google/Outlook) | Scheduling | 🔲 |
| Slack | Messaging | 🔲 |
| CRM (optional) | Business | 🔲 |

---

## Riesgos Conocidos

| ID | Riesgo | Severidad | Mitigación |
|----|--------|-----------|------------|
| R1 | **API abierta** — sin autenticación | ALTA | Planificado: Supabase Auth |
| R2 | **psycopg2 sync** en async handlers | MEDIA | Funcional pero bloquea event loop |
| R3 | **Free tier limits** — Gemini/Groq rate limits | MEDIA | Circuit breaker + fallback chain |
| R4 | **Evaluaciones in-memory** — se pierden en restart | MEDIA | Postgres persistence mitiga parcialmente |
| R5 | **Governance auto-approve** — no human-in-the-loop | MEDIA | Planificado: approval queue |
| R6 | **Single server** — no HA/scaling | BAJA | Aceptable para MVP |
| R7 | **ChromaDB local** — no backup automático | BAJA | `chroma_data/` en disco local |

---

## Glosario

| Término | Definición |
|---------|-----------|
| **ADLRA** | Autonomous Digital Delegate for Learning, Representing, and Acting — nombre completo del sistema |
| **Baseline Embedding** | Vector 4096-dim (Qwen3-Embedding-8B via EmbeddingRouter) del perfil de identidad, usado como referencia para medir drift |
| **Big Five** | Modelo OCEAN de personalidad: Openness, Conscientiousness, Extraversion, Agreeableness, Neuroticism |
| **ChromaDB** | Base de datos vectorial local para [memoria](../architecture/memory.md) episódica y semántica |
| **Circuit Breaker** | Patrón de resiliencia: CLOSED → OPEN → HALF_OPEN para proveedores LLM |
| **Cognitive Mode** | 3 niveles de inteligencia: Full (1), Memory+LLM (2), Memory Only (3) |
| **Content Hash** | SHA-256 del perfil de identidad para detectar cambios |
| **DecisionEngine** | Motor determinístico [inmutable](../architecture/cognition.md) que evalúa estrategia y riesgo sin LLM |
| **Drift** | Desviación del output respecto al baseline de [identidad](../architecture/identity.md) |
| **Enforcement** | Verificación soft de drift post-generación (Phase 5A) |
| **EventBus** | [Sistema pub/sub](../architecture/events.md) que broadcast a WebSocket + Postgres + subscribers |
| **Governance** | Marco de [control](../dashboard/governance-console.md): autonomía, reglas, aprobaciones, emergency stop |
| **Identity Profile** | Modelo Pydantic con Big Five, valores, comunicación, límites, baseline embedding |
| **LRU** | Least Recently Used — política de eviction de working memory |
| **Model Router** | [Enrutador](../integrations/README.md) con fallback chain (Gemini → Groq, 2-level) |
| **OLS** | Ordinary Least Squares — regresión lineal para trends en [evolución](../architecture/identity.md) |
| **Orchestrator** | [Cerebro central](../architecture/pipeline.md): pipeline de 25+ pasos por cada mensaje |
| **Planner** | Componente [stateless](../architecture/cognition.md) que construye Plan con steps y gates |
| **Principal** | La persona humana a quien Django representa (Harold) |
| **RRF** | Reciprocal Rank Fusion — combinación de rankings en [hybrid search](../architecture/memory.md) |
| **Shadow Simulation** | Simulación no-mutante de candidato de [identidad](../architecture/identity.md) (Phase 10B) |
| **Task** | VS Code Task definida en [tasks.json](../config/README.md) para gestionar servicios |
| **Teleology** | Sistema de [goals](../architecture/teleology.md) con jerarquía, priorización y recompensas |
| **Trace** | Grafo dirigido de [nodos](../architecture/events.md) por interacción, visualizable como pipeline horizontal CSS Grid |
| **Working Memory** | Buffer efímero per-conversation (max 20 msgs, 1h TTL, [LRU](../architecture/memory.md)) |

---

## Temas Relacionados

- [Arquitectura](../architecture/README.md) — Vista completa del sistema
- [Pipeline](../architecture/pipeline.md) — Flujo de procesamiento
- [Integraciones](../integrations/README.md) — Discord, Model Router
- [Configuración](../config/README.md) — Archivos de config
- [Baseline.md](../Baseline.md) — Documento monolítico original
