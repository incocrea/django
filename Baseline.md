# ADLRA — Baseline Arquitectónico Completo

> **Autonomous Digital Learning Representative Agent (ADLRA)** — Sistema multi-agente que aprende a representar la identidad, personalidad y estilo de toma de decisiones de su principal a través de entrenamiento progresivo, memoria contextual y feedback humano.

**Principal**: Harold Vélez
**Nombre del delegado**: Django
**Fecha de línea base**: 21 de febrero de 2026

---

## Tabla de Contenidos

1. [Resumen Ejecutivo](#1-resumen-ejecutivo)
2. [Métricas del Proyecto](#2-métricas-del-proyecto)
3. [Stack Tecnológico](#3-stack-tecnológico)
4. [Estructura de Archivos](#4-estructura-de-archivos)
5. [Configuración del Proyecto](#5-configuración-del-proyecto)
6. [Arquitectura Backend](#6-arquitectura-backend)
7. [Sistema de Identidad](#7-sistema-de-identidad)
8. [Sistema de Teleología](#8-sistema-de-teleología)
9. [Pipeline del Orchestrator](#9-pipeline-del-orchestrator)
10. [API REST — 116 Endpoints](#10-api-rest--116-endpoints)
11. [Dashboard — 15 Páginas](#11-dashboard--15-páginas)
12. [Componentes del Dashboard](#12-componentes-del-dashboard)
13. [Bot de Discord](#13-bot-de-discord)
14. [Base de Datos](#14-base-de-datos)
15. [Sistema de Tests](#15-sistema-de-tests)
16. [Estado Actual vs Planificado](#16-estado-actual-vs-planificado)
17. [Hoja de Ruta](#17-hoja-de-ruta)
18. [Convenciones de Desarrollo](#18-convenciones-de-desarrollo)
19. [Glosario](#19-glosario)

---

## 1. Resumen Ejecutivo

ADLRA es un clon virtual autónomo diseñado para emular a Harold Vélez. El sistema consta de:

- **Backend**: FastAPI con pipeline de 25+ pasos, 5 agentes especializados, 22 módulos de identidad, 4 niveles de memoria, sistema teleológico de metas
- **Frontend**: Dashboard Next.js 15 con 15 páginas de control, monitoreo y configuración
- **Integraciones**: Bot de Discord para conversación natural, webhook para publicación
- **LLMs**: Cadena Gemini → Groq → Ollama con circuit breaker y fallback automático
- **Costo operativo**: $0/mes (todos servicios en tier gratuito)

El principal puede entrenar, ajustar, monitorear y gobernar al delegado desde el dashboard. Cada interacción pasa por clasificación, cognición determinística, memoria, generación LLM, revisión de identidad, gobernanza, evaluación y persistencia — todo trazable vía grafos React Flow.

---

## 2. Métricas del Proyecto

| Métrica | Valor |
|---------|-------|
| Backend Python (src/) | 25,217 líneas, 89 archivos |
| Tests (pytest) | 20,041 líneas, 47 archivos |
| Dashboard (app/ + components/ + lib/) | 12,490 líneas |
| **Total líneas de código** | **57,748** |
| Endpoints REST + WebSocket | 116 |
| Páginas del dashboard | 15 |
| Módulos de identidad | 22 archivos, 6,080 líneas |
| Tablas en Postgres | 15 |
| Agentes LLM | 5 |
| Proveedores LLM | 3 (Gemini, Groq, Ollama) |
| Fases completadas | 17 (Phase 1 → Phase 17) |
| i18n keys | 631 (EN + ES simétricos) |

---

## 3. Stack Tecnológico

| Capa | Tecnología | Detalles |
|------|-----------|----------|
| **Backend** | Python 3.11, FastAPI, uvicorn | Puerto 8000, `--reload` en dev |
| **Frontend** | Next.js 15 (App Router), React 19, TypeScript 5.7 | Puerto 3000, proxy `/api/*` → backend |
| **UI** | shadcn/ui, Tailwind CSS 3.4, lucide-react, CVA | Tema oscuro laboratorio, variables CSS custom |
| **Estado** | Zustand 5 | Persistencia localStorage para chat |
| **LLM** | Gemini 2.5 Flash, Groq Llama 3.3 70B, Ollama (qwen2.5:32b, llama3.1:8b) | Cadena de fallback automática |
| **Vector DB** | ChromaDB (local) | Embedding all-MiniLM-L6-v2 (384-dim) |
| **SQL DB** | Neon Postgres (remoto) | psycopg2 sync, autocommit |
| **Procedural DB** | SQLite | Correcciones, workflows |
| **Charts** | recharts, SVG inline | Visualizaciones analytics |
| **Flow Viz** | @xyflow/react v12 | Trazas cognitivas |
| **Discord** | discord.py, httpx | Bot + webhook publisher |
| **Auth** | Supabase (planificado) | API abierta actualmente |

---

## 4. Estructura de Archivos

```
iame.lol/
├── agent/                                    # Backend Python FastAPI
│   ├── src/                                  # 25,217 líneas, 89 archivos
│   │   ├── agents/          (878 ln, 8 files) # 5 agentes + crew + base
│   │   ├── api/           (4,110 ln, 3 files) # main.py (403) + routes.py (3,707)
│   │   ├── cognition/       (349 ln, 3 files) # DecisionEngine + Planner + categories
│   │   ├── db/            (1,970 ln, 3 files) # database.py (408) + persistence.py (1,507)
│   │   ├── evaluation/    (2,151 ln, 6 files) # 5 módulos heurísticos
│   │   ├── events/          (151 ln, 2 files) # EventBus pub/sub + WebSocket broadcast
│   │   ├── flows/         (3,069 ln, 5 files) # orchestrator (2,716) + middleware + parallel + categories
│   │   ├── governance/         (1 file)       # Stub (__init__.py)
│   │   ├── identity/      (6,080 ln, 22 files)# 22 módulos Phase 4-10C
│   │   ├── memory/        (1,636 ln, 4 files) # manager (956) + hybrid_search + compaction
│   │   ├── router/          (629 ln, 3 files) # model_router (476) + circuit_breaker (153)
│   │   ├── security/        (429 ln, 4 files) # input_sanitizer + content_wrapper + middleware
│   │   ├── skills/        (1,364 ln, 6 files) # registry + web_research + learn_topic + repo_explorer + tools
│   │   ├── teleology/    (2,294 ln, 11 files) # goals + plans + priorities + rewards + conflicts
│   │   ├── trace/           (~408 ln, 2 files) # TraceCollector + TraceStore
│   │   ├── training/        (521 ln, 2 files) # 3 modos de entrenamiento
│   │   ├── config.py                  (117 ln)# Pydantic BaseSettings
│   │   ├── service_logger.py          (263 ln)# Rotating file logger
│   │   └── watchdog.py               (136 ln)# Service health watchdog
│   ├── tests/                                 # 20,041 líneas, 47 test files + conftest.py
│   └── configs → ../configs                   # Symlink
├── dashboard/                                 # Frontend Next.js 15
│   ├── app/                                   # 15 rutas (App Router)
│   ├── components/                            # 31 archivos en 7 directorios
│   ├── lib/                                   # api.ts (1,105 ln), store.ts (159 ln), hooks/, i18n/
│   ├── next.config.ts                         # Proxy /api/* → localhost:8000
│   ├── tailwind.config.ts                     # Tema lab oscuro custom
│   └── package.json                           # 33 deps, 10 devDeps
├── configs/
│   ├── persona.yaml              (89 ln)      # Identidad del principal
│   ├── models.json              (138 ln)      # Proveedores LLM, fallback, perfiles
│   ├── skills.json               (71 ln)      # Registro de habilidades
│   └── governance.yaml          (283 ln)      # Autonomía, riesgos, prohibiciones
├── scripts/
│   ├── discord_bot.py           (572 ln)      # Bot Discord natural
│   └── discord_post.py          (188 ln)      # Webhook publisher
├── .vscode/tasks.json            (78 ln)      # 5 tasks de servicio
├── .env                                       # Variables de entorno (gitignored)
├── .env.example                               # Template de variables
├── .gitignore                    (82 ln)      # Python + Node + datos + logs
└── docs/
    └── Baseline.md                            # Este documento
```

---

## 5. Configuración del Proyecto

### 5.1 persona.yaml — Identidad del Principal

Define quién es Harold Vélez para el sistema. Leído por `IdentityManager`, `IdentityCoreAgent`, y los endpoints de persona.

| Sección | Contenido |
|---------|-----------|
| `principal_name` | "Harold Vélez" |
| `personality` (Big Five) | openness 0.81, conscientiousness 0.74, extraversion 0.84, agreeableness 0.75, neuroticism 0.23 |
| `values` | efficiency, quality, innovation, integrity, reliability, respect |
| `communication` | formality 0.35, directness 0.77, verbosity 0.3, emotional_expression 0.68, humor 0.91, empathy 0.82 |
| `decision_making` | risk_tolerance moderate, analysis_depth thorough, stakeholder_weight high, data_vs_intuition 0.7 |
| `boundaries` | 5 reglas duras (presupuesto/timeline, no conflictos, confidencialidad, limitaciones, criterios de éxito) |
| `expertise` | Primary: fullstack dev, AI/ML, project management. Secondary: 3 más |
| `autonomy` | current_level: 0 (Observer). Per-category: todas en 0 |

**Flujo**: Dashboard Identity Studio → `PUT /api/persona/*` → escribe YAML → `agent_crew.reload_persona()` → todos los agentes actualizan su system prompt.

### 5.2 models.json — Configuración de LLMs

Define proveedores, modelos, cadena de fallback y perfiles.

| Sección | Contenido |
|---------|-----------|
| `providers` | Gemini (gemini-2.5-flash, 1M ctx), Groq (llama-3.3-70b, 128K ctx), Ollama (qwen2.5:32b, llama3.1:8b, mxbai-embed-large) |
| `fallback_chain` | gemini → groq → ollama/llama3.1:8b |
| `agent_assignments` | identity_core→Gemini, business→Groq, communication→Gemini, technical→Groq, governance→Gemini |
| `profiles` | balanced (default), max_quality (all Gemini), privacy_mode (all Ollama), budget_mode (all Ollama) |
| `task_type_routing` | embeddings→mxbai, fast_classification→Groq, deep_reasoning→Gemini, code_generation→Groq |

**Flujo**: Dashboard Model Manager → `PUT /api/models/assignment` o `PUT /api/models/profile` → escribe JSON → `model_router.reload_config()`.

### 5.3 skills.json — Registro de Habilidades

| Skill ID | Tipo | Riesgo | Habilitado | Dependencias |
|----------|------|--------|------------|--------------|
| `web-research` | tool | low | sí | — |
| `email-draft` | tool | low | sí | identity-core |
| `document-gen` | tool | low | sí | identity-core |
| `learn-topic` | tool | low | sí | web-research, identity-core |
| `repo-explorer` | tool | low | sí | identity-core |

### 5.4 governance.yaml — Constitución del Agente

| Sección | Contenido |
|---------|-----------|
| `autonomy_levels` (0-4) | Observer → Assistant → Delegate → Autonomous → Trusted |
| `risk_levels` (4) | low (auto@L1), medium (auto@L2), high (approval, auto@L3), critical (2FA, auto@L4) |
| `forbidden` (7 reglas) | No bypass identity, no ocultar naturaleza AI, no auto-modificar governance, no deshabilitar audit |
| `self_modification` | File permission matrix, safety checks (test-driven, git isolation, auto-rollback), rate limits |
| `monitoring` | Drift detection (threshold 0.85), performance tracking (5% error, 2s target) |
| `privacy` | Procesamiento local requerido para muestras/correcciones; nunca enviar datos de identidad a cloud |

### 5.5 config.py — Variables de Entorno (Pydantic BaseSettings)

Carga desde `.env` vía `@lru_cache`. Agrupadas por función:

| Grupo | Variables | Defaults |
|-------|----------|----------|
| App | `app_name`, `app_version`, `environment`, `debug` | "Django", "2.0.0", "development", True |
| API | `api_host`, `api_port` | "0.0.0.0", 8000 |
| LLM | `gemini_api_key`, `groq_api_key`, `ollama_base_url` | None, None, localhost:11434 |
| DB | `database_url`, `chroma_persist_directory` | None, "./chroma_data" |
| Web Research | `tavily_api_key` | None |
| Governance | `default_autonomy_level`, `principal_name` | 0, "Principal" |

Properties booleanas: `has_gemini`, `has_groq`, `has_tavily`, `has_database`, `has_r2`, `has_supabase`.

### 5.6 .vscode/tasks.json — Gestión de Servicios

| Task | Comando | Notas |
|------|---------|-------|
| `Backend: FastAPI Server` | `uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --reload` | instanceLimit:1 |
| `Dashboard: Next.js Dev Server` | `Remove-Item .next; npx next dev --port 3000` | instanceLimit:1 |
| `Health Watchdog` | `python -m src.watchdog` | reveal:never |
| `Discord: Django Bot` | `venv/Scripts/python.exe scripts/discord_bot.py` | PID lock, instanceLimit:1 |
| `Start All Services` | dependsOn las 4 anteriores (parallel) | Build group default |

**Regla crítica**: Todos los servicios se inician via VS Code Tasks, NUNCA con `run_in_terminal isBackground:true`.

---

## 6. Arquitectura Backend

### 6.1 Composición — main.py (403 líneas)

`AppState` es el singleton global que contiene todos los componentes inicializados. La secuencia de `lifespan()`:

| Paso | Componente | Wiring |
|------|-----------|--------|
| 1 | `Settings` | `get_settings()` |
| 2 | `Database` | `.connect()` → `.initialize_schema()` → `event_bus.set_db_callback()` |
| 3 | `PersistenceRepository` | `PersistenceRepository(database)` |
| 4 | `ModelRouter` | wira callback de persistencia de tokens |
| 5 | `AgentCrew` | 5 agentes con persona + governance config |
| 6 | `MemoryManager` | `.set_model_router()` → `.set_database()` |
| 7 | `SkillRegistry` | skills.json |
| 8 | `TrainingManager` | `await load_corrections()` |
| 9 | `IdentityManager` | `.load_active_profile()` (DB first, fallback rebuild) |
| 10 | `DecisionEngine` | Inmutable, `autonomy_level=0`, `identity_profile=profile` |
| 11 | `Planner` | Stateless, sin args |
| 12 | Wire AlignmentEvaluator | `.set_baseline_embedding()` |
| 13 | `MiddlewareRegistry` | Pipeline hooks |
| 14 | `Orchestrator` | Requiere DecisionEngine + Planner (isinstance check) |
| 15 | Security Middleware | Registrado en PRE_CLASSIFY, prioridad 10 |
| 16 | Teleology | GoalManager, PriorityEngine, RewardModel, PlanExecutor, ConflictDetector, TeleologicalGovernance, BackgroundReevaluator |
| 17 | Teleology Middleware | 3 hooks vía `build_teleology_middleware()` |
| 18 | Memory rollback persistence | Callback wiring |
| 19 | Emit `system.startup` | Via EventBus |

**Shutdown**: service_log.shutdown → background_reevaluator.stop → memory_manager.close → database.close.

**Middleware HTTP**: `ServiceMonitorMiddleware` (logs >5s y errores 500), `CORSMiddleware` (localhost:3000).

### 6.2 Agentes — src/agents/ (878 líneas, 8 archivos)

| Agente | Clase | Extiende | Rol | Modelo Default |
|--------|-------|----------|-----|----------------|
| Identity Core | `IdentityCoreAgent` | Standalone | Guardián de persona, responde COMO el principal | Gemini |
| Business | `BusinessAgent` | `BaseAgent` | Estrategia, deals, pricing | Groq |
| Communication | `CommunicationAgent` | `BaseAgent` | Emails, propuestas en estilo del principal | Gemini |
| Technical | `TechnicalAgent` | `BaseAgent` | Código, arquitectura, debugging | Groq |
| Governance | `GovernanceAgent` | `BaseAgent` | Meta-agente revisor de compliance (JSON output, temp 0.2) | Gemini |

`AgentCrew` inicializa los 5 agentes con `ModelRouter` + paths de persona/governance.

### 6.3 Model Router — src/router/ (629 líneas)

**model_router.py (476 ln)**: Cadena de fallback Gemini → Groq → Ollama.
- `generate(prompt, role, system_prompt, temperature, max_tokens)` → `ModelResponse`
- Token counting real por proveedor (Gemini `usage_metadata`, Groq `usage.total_tokens`, Ollama `eval_count`)
- Estimación de costo: Gemini $0.15/1M, Groq $0.05/1M, Ollama $0
- Hot-reload via `reload_config()`
- 4 perfiles switcheables
- Callback de persistencia de tokens wired al startup

**circuit_breaker.py (153 ln)**: Circuit breaker por proveedor.
- Estados: CLOSED (normal) → OPEN (fallo) → HALF_OPEN (prueba)
- Threshold configurable de fallos consecutivos
- Timeout de recovery
- Integrado en Health Bar del dashboard (LEDs verde/amber/rojo)

### 6.4 Cognición — src/cognition/ (349 líneas, 3 archivos)

Capa 100% determinística — cero llamadas LLM, cero IO. El Orchestrator no puede iniciar sin ella (isinstance guards).

**DecisionEngine** (181 ln) — **Inmutable** (`__slots__` + `__setattr__` override):
- `Strategy` enum: DIRECT_RESPONSE, STRUCTURED_ANALYSIS, RESEARCH_REQUIRED, MULTI_AGENT
- `evaluate(category, message, governance_enabled)` → `DecisionResult` (frozen dataclass)
- Determina categoría, estrategia, agente, nivel de riesgo, si requiere revisión de identidad/gobernanza

**Planner** (144 ln) — **Stateless**:
- `build(decision, governance_enabled)` → `Plan` (frozen dataclass)
- Plan contiene PlanSteps con gate flags para identity_review y governance_review

**categories.py** (81 ln) — `TaskCategory` enum compartido (evita dependencia circular orchestrator↔cognition).

### 6.5 Memoria — src/memory/ (1,636 líneas, 4 archivos)

| Tier | Storage | Propósito | Persistencia |
|------|---------|-----------|-------------|
| **Working** | OrderedDict per conversation_id (max 20 msgs, max 50 sessions, LRU eviction, 1h TTL) | Buffer de conversación activa, aislado por conversation_id | Efímero |
| **Episodic** | ChromaDB `episodic_memory` | Logs de interacción, búsqueda semántica | Disco (`chroma_data/`) |
| **Semantic** | ChromaDB `semantic_memory` | Base de conocimiento, writing samples, RAG | Disco (`chroma_data/`) |
| **Procedural** | SQLite `procedural.db` | Workflows, patrones, correcciones | Disco (`data/`) |

**manager.py (956 ln)**: `MemoryManager` — `recall()` → `build_context()` → `store_interaction()`. Budget ~2000 tokens. Prioridad: working > semantic > episodic. `clear_working(conversation_id)` por conversación.

**hybrid_search.py**: Búsqueda híbrida RRF (Reciprocal Rank Fusion) + MMR (Maximal Marginal Relevance) + temporal decay. FTS via tabla `episodic_search` con tsvector + GIN index.

**compaction.py**: Motor de compactación — resume clusters episódicos en abstracciones semánticas.

### 6.6 Evaluación — src/evaluation/ (2,151 líneas, 6 archivos)

Todos heurísticos, sin llamadas LLM extra. Singletons en memoria (max 200-1000 records, perdidos en restart).

| Módulo | Líneas | Qué mide |
|--------|--------|----------|
| `quality_scorer.py` | 456 | 5 dimensiones: Relevance, Coherence, Completeness, Efficiency, Memory Utilization → grade A-F compuesto |
| `alignment_evaluator.py` | 506 | Alineación con persona: value keywords, communication style, boundary compliance, decision-making, identity_similarity |
| `legal_risk.py` | 359 | 15+ regex patterns en 6 categorías (contractual, financiero, confidencialidad, liability, regulatorio, AI disclosure) |
| `decision_registry.py` | 407 | Detecta decisiones de negocio, registra trigger/rationale/alternatives/trade-offs/stakeholders |
| `memory_rollback.py` | 408 | Trackea cada operación de memoria con before/after state, checkpoints, undo |

### 6.7 Eventos — src/events/event_bus.py (151 líneas)

`EventBus` singleton — broadcast simultáneo a: (1) WebSocket connections, (2) Postgres `audit_log`, (3) suscriptores in-memory. Últimos 100 eventos en memoria. `IAmeEvent` dataclass con `event_type`, `data`, `risk_level`, `timestamp`.

### 6.8 Trace Cognitivo — src/trace/ (~408 líneas)

**collector.py (407 ln)**: `TraceCollector` — per-interaction, crea grafo React Flow con auto-layout. `TraceNode` dataclass incluye `persist_id`, `model_used`, `risk_score`.

**32 tipos de nodo**: input, classify, decision_engine, planner, memory_recall, correction, context_weighting, prompt_build, llm_generate, identity_review, governance_review, memory_store, evaluation, output, branch, multi_agent, identity_decision_modulation, identity_confidence, identity_autonomy, identity_bias, identity_memory_bridge, identity_retrieval_weight, identity_context_weight, identity_prompt_inject, identity_consolidation, identity_drift, identity_policy, identity_feedback, identity_health_monitor, identity_health_regulation, identity_evolution, identity_shadow, identity_version_candidate.

`TraceStore` — singleton, últimas 100 trazas en memoria.

### 6.9 Security — src/security/ (429 líneas, 4 archivos)

| Archivo | Función |
|---------|---------|
| `input_sanitizer.py` | Sanitización de input: detecta XSS, SQL injection, prompt injection. Modo detect-only en Phase 11 |
| `content_wrapper.py` | Marca contenido externo con boundary markers de seguridad |
| `middleware.py` (58 ln) | `SecurityMiddleware` en posición PRE_CLASSIFY (priority 10) del pipeline de middleware |

### 6.10 Skills — src/skills/ (1,364 líneas, 6 archivos)

| Archivo | Función |
|---------|---------|
| `registry.py` | Carga skills.json, toggle enable/disable, tracking de uso |
| `web_research.py` | Wrapper Tavily API (search + deep research con sub-queries) |
| `learn_topic.py` (314 ln) | Web search → LLM summarize → chunk → ChromaDB semantic memory. Depth selector (1-3) |
| `repo_explorer.py` (1,000 ln) | Lee archivos locales, explora directorios, fetch repos GitHub, accede docs/ propios. Chat-trigger + API + optional memory storage |
| `tools.py` | `EmailDraftTool` + `DocumentGenTool` (ambos usan CommunicationAgent) |

### 6.11 Training — src/training/manager.py (521 líneas)

3 modos: correction, free_conversation, guided_interview.
- **Correction**: Principal da (original, corrected, feedback) → procedural memory SQLite → genera suggestion → inyectado en cada prompt
- **Free Conversation**: Intercambio natural → LLM extrae rasgos de personalidad → semantic memory ChromaDB
- **Guided Interview**: 15 preguntas predefinidas → análisis de respuestas → extracción de rasgos → semantic memory

Writing sample upload: chunking ~500 palabras → semantic memory como `writing_sample` category.

### 6.12 Service Logger — src/service_logger.py (263 líneas)

Rotating file logs (`logs/service.log`, 5×2MB) + structured JSONL (`logs/service_events.jsonl`). Tracks starts, stops, crashes, GPU events, slow requests con system snapshots (RSS, CPU%, GPU VRAM).

### 6.13 Watchdog — src/watchdog.py (136 líneas)

Monitor de salud del servicio — polling periódico de `/health` y reporta estado.

---

## 7. Sistema de Identidad — src/identity/ (6,080 líneas, 22 archivos)

Representación formal, versionada y embedding-backed de la identidad del principal. Inyectada en DecisionEngine (read-only) y AlignmentEvaluator (drift detection).

### 7.1 Tabla de Módulos

| Archivo | Líneas | Fase | Función |
|---------|--------|------|---------|
| `schema.py` | ~105 | 4 | `IdentityProfile` Pydantic model — version, Big Five, values, comm_style, baseline_embedding 384-dim, drift_threshold |
| `embedding.py` | ~215 | 4 | `build_identity_text()` + `compute_baseline_embedding()` (all-MiniLM-L6-v2) + `cosine_similarity()` |
| `versioning.py` | ~215 | 4 | `IdentityVersionManager` — semantic versioning, SHA-256 hashing, persistence via config_versions |
| `manager.py` | ~416 | 4 | `IdentityManager` singleton — load/save/rebuild profile. DB first, fallback YAML. Short-circuit si hash unchanged |
| `enforcement.py` | ~90 | 5A | `IdentityEnforcer` — evalúa similarity vs drift_threshold. Observational only |
| `policy.py` | ~165 | 5B | `IdentityPolicyEngine` — clasifica severidad de drift → acción (none/log/flag/rewrite_request/block) |
| `feedback.py` | ~110 | 5C | `IdentityFeedbackController` — genera hints de corrección cuando action==rewrite_request AND severity==high |
| `memory_bridge.py` | ~165 | 6A | `IdentityMemoryBridge` — cosine similarity per recalled memory vs baseline |
| `context_weighting.py` | ~200 | 6B | `IdentityContextWeighter` — tags `[IDENTITY_ALIGNED]` o `[LOW_IDENTITY_ALIGNMENT]` en líneas de contexto |
| `decision_modulation.py` | ~333 | 6C | `IdentityDecisionModulator` — evalúa decision–identity alignment across 4 factores |
| `confidence.py` | ~264 | 6D | `IdentityConfidenceEngine` — agrega señales en confidence score (0-1) + autonomy_modifier |
| `autonomy_modulation.py` | ~175 | 7A | `IdentityAutonomyModulator` — ajusta governance threshold según confidence |
| `retrieval_weighting.py` | ~210 | 7B | `IdentityRetrievalWeighter` — re-ranking: 0.8×semantic + 0.2×identity affinity |
| `consolidation_weighting.py` | ~210 | 7C | `IdentityConsolidationWeighter` — ajusta importance pre-storage usando confidence + alignment |
| `behavioral_bias.py` | ~500 | 8A | `IdentityBehavioralBias` — recommended_planner_mode + style_bias (tone, assertiveness, depth, creativity) |
| `prompt_integration.py` | ~195 | 8B | `IdentityPromptIntegrator` — inyecta `[IDENTITY STYLE PREFERENCES]` block en system prompt |
| `health_monitor.py` | ~320 | 9A | `IdentityHealthMonitor` — sliding window 50 interacciones, instability_index, health classification |
| `health_regulation.py` | ~260 | 9B | `IdentityHealthRegulator` — ajusta governance + identity weight según health status |
| `evolution.py` | ~600 | 10A | `IdentityEvolutionEngine` — análisis de evolución a largo plazo, propuestas de evolución |
| `shadow_simulation.py` | ~528 | 10B | `IdentityShadowSimulator` — simulación paralela no-mutante de candidato de identidad |
| `version_control.py` | ~710 | 10C | `IdentityVersionControl` — snapshots inmutables, apply/rollback controlados, SHA-256 verification |

### 7.2 Flujo de Identidad en el Pipeline

Los módulos de identidad se integran en el Orchestrator en pasos específicos:

| Pipeline Step | Módulo | Función |
|---------------|--------|---------|
| 2b | Memory Bridge | Calcula afinidad per-memory vs baseline embedding |
| 2c | Retrieval Weighting | Re-rankea memorias (0.8 semantic + 0.2 identity affinity) |
| 3b | Context Weighting | Anota líneas de contexto con tags de alineación |
| 3c | Decision Modulation | Evalúa alignment decisión-identidad (4 factores) |
| 3d | Confidence | Agrega señales → confidence score |
| 3e | Autonomy Modulation | Ajusta governance threshold |
| 3f | Behavioral Bias | Genera style_bias para planning |
| 3g | Prompt Integration | Inyecta preferences en system prompt |
| 6d | Consolidation Weighting | Ajusta importance pre-storage |
| 7c | Enforcement | Evalúa drift vs threshold |
| 7d | Policy | Clasificata severidad, determina acción |
| 7e | Feedback | Genera correction hints si rewrite_request+high |
| 9a | Health Monitor | Monitoreo longitudinal post-persistence |
| 9b | Health Regulation | Ajuste adaptativo post-health |
| 9c | Evolution | Análisis de evolución (propuesta only) |
| 9d | Shadow Simulation | Simulación paralela no-mutante |
| 9e | Version Control | Crea candidato de versión (proposal only) |

---

## 8. Sistema de Teleología — src/teleology/ (2,294 líneas, 11 archivos)

Sistema de metas y propósitos del agente.

### 8.1 Tabla de Archivos

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
| `tel_middleware.py` | 112 | Middleware hooks en el pipeline del orchestrator |

### 8.2 Jerarquía de Metas

| Tipo | Fuente | Duración | Cancelación |
|------|--------|----------|-------------|
| **MISSION** | Solo principal | Indefinida | Requiere confirmación humana |
| **STRATEGIC** | Principal o MISSION | Semanas-meses | Requiere confirmación humana |
| **OPERATIONAL** | Cualquier nivel superior | Días-semanas | Automática si padre se cancela |
| **TACTICAL** | Sistema o usuario | Horas-días | Automática al completarse o expirar |

### 8.3 Máquina de Estados

```
CREATED → ACTIVE → COMPLETED
                 → FAILED
                 → CANCELLED
          ACTIVE ↔ PAUSED
          ACTIVE ↔ BLOCKED (dependencia no resuelta)
```

GoalCondition types: `metric_threshold`, `event_occurred`, `time_elapsed`, `memory_contains`, `human_confirmed`.

### 8.4 Motor de Prioridades (7 factores)

| Factor | Peso | Descripción |
|--------|------|-------------|
| `base_priority` | 0.25 | Prioridad inherente del tipo |
| `urgency` | 0.20 | Proximidad a deadline |
| `identity_alignment` | 0.15 | Alineación con IdentityProfile |
| `dependency_pressure` | 0.15 | Cuántas metas dependen de esta |
| `progress_momentum` | 0.10 | Progreso reciente |
| `latency_penalty` | 0.10 | Tiempo sin progreso (MISSION=∞, STRATEGIC=30d, OPERATIONAL=14d, TACTICAL=3d) |
| `context_relevance` | 0.05 | Relevancia al contexto actual |

### 8.5 Modelo de Recompensas (7 componentes, -1.0 a 1.0)

`goal_progress`, `quality_score`, `identity_fidelity`, `governance_compliance`, `efficiency`, `novelty`, `user_satisfaction`. Stored en tabla `reward_signals`, rolling 500 interacciones.

### 8.6 Background Loop

`GoalReevaluationLoop` — cada 30min. Safeguards: max 1 concurrente, timeout 60s, max 3 acciones/ciclo, respeta emergency stop, requiere `autonomy_level ≥ 2` (actualmente Level 0 → solo observa).

### 8.7 Resolución de Conflictos

6 tipos (resource, value, temporal, dependency, priority, identity) × 5 estrategias en cascada: type dominance → priority dominance → identity dominance → temporal pause → escalation.

### 8.8 Gobernanza de Metas

4 checks: identity alignment (<0.40 reject, 0.40-0.60 flag, >0.60 approve), forbidden operations, value consistency, drift protection.

### 8.9 Integración con Orchestrator

| Step | Nombre | Posición |
|------|--------|----------|
| 0.3 | `GoalContextInjector` | PRE_CLASSIFY — inyecta contexto de metas activas |
| 10b | `RewardComputer` | POST_STORE — calcula señal de recompensa |
| 10c | `GoalProgressUpdater` | POST_STORE — actualiza progreso de metas |

---

## 9. Pipeline del Orchestrator — src/flows/orchestrator.py (2,716 líneas)

Pipeline central de 25+ pasos. 3 Modos Cognitivos: Full (1, sin restricciones), Memory+LLM (2, default, grounded en memoria), Memory Only (3, sin LLM).

| Step | Nombre | Descripción |
|------|--------|-------------|
| 0 | Emergency Check | Bloquea si `_emergency_stopped` |
| 0.3 | Goal Context Injector | Inyecta metas activas (teleología middleware) |
| 0.5 | Learn-topic Auto-detect | Regex "aprende sobre X" → web → LLM → ChromaDB (bypass steps 2-10) |
| 0.7 | Repo Explorer Auto-detect | Regex "lee/explora repo X" → local/GitHub/docs read → context injection (bypass steps 2-10). Modes 1-2 |
| 1 | Classify | Keyword heuristics → TaskCategory enum |
| 2 | Decision Engine | Deterministic strategy/risk/agent (no LLM) |
| 3c | Identity Decision Modulation | Evalúa alignment decision-identity (observational) |
| 3d | Identity Confidence | Agrega señales → confidence 0-1 + autonomy_modifier |
| 3e | Autonomy Modulation | Ajusta governance threshold |
| 3f | Behavioral Bias | Genera style_bias + planner_mode recommendation |
| 3g | Prompt Integration | Inyecta identity style preferences |
| 3 | Planner | Builds step-by-step Plan con gate flags (no LLM) |
| 4 | Memory Recall + Mode Filter | ChromaDB search + working memory. Mode 2-3: filtra semantic a `learned_knowledge` |
| 2b | Identity Memory Bridge | Cosine similarity per memory vs baseline |
| 2c | Identity Retrieval Weighting | Re-rank 0.8×semantic + 0.2×identity |
| 5 | Correction Injection | Behavioral suggestions from training |
| 3b | Identity Context Weighting | Tags [ALIGNED]/[LOW_ALIGNMENT] en contexto |
| 6 | Prompt Build + Mode Logic | Mode 3: return memory-only. Mode 2: knowledge headers |
| 7 | LLM Generate | Route to agent via ModelRouter |
| 8 | Identity Review | Per plan: non-conversation reviewed by Identity Core |
| 9 | Governance Review | Per plan: reviewed by GovernanceAgent (JSON) |
| 6d | Consolidation Weighting | Ajusta importance pre-storage |
| 10 | Memory Store | Save working + episodic + 5 evaluaciones + Postgres persist |
| 10b | Reward Computer | Calcula señal de recompensa (teleología middleware) |
| 10c | Goal Progress Updater | Actualiza progreso de metas |
| 7c | Identity Enforcement | Evalúa drift vs threshold |
| 7d | Identity Policy | Clasifica severidad, determina acción |
| 7e | Identity Feedback | Genera correction hints |
| 9a | Health Monitor | Monitoreo longitudinal (sliding window 50) |
| 9b | Health Regulation | Ajuste adaptativo |
| 9c | Evolution Analysis | Propuesta de evolución (200 interacciones, OLS regression) |
| 9d | Shadow Simulation | Simulación paralela no-mutante |
| 9e | Version Control | Candidato de versión |

Cada paso emite eventos via EventBus y crea nodos via TraceCollector.

### Middleware Pipeline — src/flows/middleware.py (196 líneas)

`MiddlewareRegistry` con `MiddlewarePosition` enum (9 hooks). Permite inyectar lógica en puntos del pipeline sin modificar orchestrator.

### Ejecución Paralela — src/flows/parallel_executor.py (204 líneas)

`ParallelExecutor` — 2+ agentes concurrentes vía `asyncio.gather`. `ResponseAggregator` fusiona resultados. Integrado cuando `Strategy.MULTI_AGENT`.

---

## 10. API REST — 116 Endpoints

### 10.1 Health & Status (4 endpoints)

| Método | Path | Descripción | DB |
|--------|------|-------------|-----|
| `GET` | `/health` | Pulse: versión, proveedores, uptime, DB status | — |
| `GET` | `/router/status` | Status del Model Router: providers, profiles, circuit breakers, latency | — |
| `GET` | `/events/recent` | Eventos recientes de EventBus/audit_log | Read: Postgres `audit_log` |
| `GET` | `/system-status` | Status compuesto pesado para dashboard Health Bar | Read: Postgres (6 queries), ChromaDB, in-memory |

### 10.2 Service Log (2 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/service-log/events` | Eventos del JSONL log file |
| `GET` | `/service-log/report` | Crash report con detección de patrones |

### 10.3 Chat (2 endpoints)

| Método | Path | Descripción | DB | Eventos |
|--------|------|-------------|-----|---------|
| `POST` | `/chat` | Chat principal — pipeline completo del orchestrator. Input sanitization, conversation management, 10+ pasos | Write: conversations, messages. Read: messages history. ChromaDB via orchestrator | `chat.message_received`, `agent_state.*`, `chat.response_generated`, `security.threat_detected` |
| `WS` | `/ws` | WebSocket para updates real-time. Ping/pong + chat directo (bypass orchestrator → identity_core) | — | Recibe todos los eventos del bus |

### 10.4 Configuration (2 endpoints)

| Método | Path | Descripción | Eventos |
|--------|------|-------------|---------|
| `POST` | `/router/reload` | Hot-reload models.json | `config.router_reloaded` |
| `POST` | `/persona/reload` | Hot-reload persona.yaml across all agents | `config.persona_reloaded` |

### 10.5 Persona / Identity Studio (5 endpoints)

| Método | Path | Descripción | Efecto | Eventos |
|--------|------|-------------|--------|---------|
| `GET` | `/persona/info` | Persona actual: name, Big Five, values, communication, boundaries | — | — |
| `PUT` | `/persona/traits` | Actualizar Big Five (0.0-1.0) | Escribe `persona.yaml`, reload all agents | `config.traits_updated` |
| `PUT` | `/persona/values` | Reemplazar lista de valores | Escribe YAML, reload | `config.values_updated` |
| `PUT` | `/persona/communication` | Actualizar sliders de comunicación | Escribe YAML, reload | `config.communication_updated` |
| `PUT` | `/persona/boundaries` | Reemplazar boundaries | Escribe YAML, reload | `config.boundaries_updated` |

### 10.6 Agent Crew (1 endpoint)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/crew/agents` | Lista 5 agentes con roles y descripciones |

### 10.7 Cognitive Trace (6 endpoints)

| Método | Path | Descripción | DB |
|--------|------|-------------|-----|
| `GET` | `/trace/list` | Lista trazas recientes (Postgres → fallback memory) | Read: interactions + trace_nodes |
| `GET` | `/trace/{id}` | Grafo completo de traza | Read: interactions + trace_nodes + evaluations |
| `GET` | `/trace/latest/graph` | Traza más reciente | Read: Postgres |
| `POST` | `/trace/replay` | Re-procesa mensaje por pipeline completo, retorna traza | Write: all orchestrator ops |
| `DELETE` | `/trace/{id}` | Elimina una traza | Write: DELETE interactions, trace_nodes, evaluations |
| `DELETE` | `/trace` | Elimina TODAS las trazas | Write: DELETE all |

### 10.8 Persisted Interactions (4 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/interactions` | Lista interacciones con filtro de categoría y paginación |
| `GET` | `/interactions/{id}/trace` | Traza persisted de una interacción |
| `GET` | `/interactions/{id}/evaluations` | Evaluaciones de una interacción |
| `GET` | `/token-usage/persisted` | Token usage agregado desde Postgres |

### 10.9 Memory System (8 endpoints)

| Método | Path | Descripción | DB |
|--------|------|-------------|-----|
| `GET` | `/memory/stats` | Stats de los 4 tiers | Read: ChromaDB + SQLite |
| `POST` | `/memory/search` | Búsqueda cross-tier con filtro | Read: ChromaDB + working |
| `POST` | `/memory/semantic/store` | Almacenar nuevo conocimiento semántico | Write: ChromaDB |
| `POST` | `/memory/working/clear` | Limpiar working memory (per-conversation o all) | — (ephemeral) |
| `POST` | `/memory/bulk-delete` | Eliminar múltiples memorias across tiers | Write: ChromaDB. Persist: memory_operations |
| `PUT` | `/memory/semantic/{id}` | Actualizar contenido/categoría de memoria | Write: ChromaDB. Persist: memory_operations |
| `DELETE` | `/memory/semantic/{id}` | Eliminar memoria semántica | Write: ChromaDB. Persist: memory_operations |
| `DELETE` | `/memory/episodic/{id}` | Eliminar memoria episódica | Write: ChromaDB. Persist: memory_operations |

### 10.10 Skills (5 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/skills` | Lista skills registradas |
| `POST` | `/skills/{id}/toggle` | Enable/disable skill |
| `POST` | `/skills/web-research` | Ejecuta web research via Tavily |
| `POST` | `/skills/learn-topic` | Research → summarize → chunk → ChromaDB |
| `POST` | `/skills/explore-repo` | Lee archivos locales/GitHub/docs, context injection |

### 10.11 Training System (14 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/training/status` | Status del sistema de training |
| `POST` | `/training/session/start` | Iniciar sesión (mode: correction/free/guided) |
| `POST` | `/training/session/end` | Terminar sesión activa |
| `POST` | `/training/correction` | Enviar corrección (original→corrected) |
| `GET` | `/training/corrections` | Listar correcciones de procedural memory |
| `PUT` | `/training/corrections/{id}` | Actualizar corrección |
| `DELETE` | `/training/corrections/{id}` | Eliminar corrección |
| `DELETE` | `/training/suggestions/{idx}` | Eliminar suggestion individual |
| `DELETE` | `/training/suggestions` | Limpiar todas las suggestions |
| `GET` | `/training/history` | Historial de sesiones + suggestions |
| `POST` | `/training/exchange` | Free conversation → LLM + trait extraction |
| `GET` | `/training/interview/questions` | 15 preguntas de entrevista guiada |
| `POST` | `/training/interview/answer` | Respuesta de entrevista → trait extraction |
| `POST` | `/training/upload-samples` | Upload .txt/.md → chunk → ChromaDB semantic |

### 10.12 Models (4 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/models/config` | Config completa + estado online de providers |
| `PUT` | `/models/assignment` | Asignar modelo a agente específico |
| `PUT` | `/models/profile` | Cambiar perfil activo (balanced/max_quality/privacy/budget) |
| `POST` | `/models/test` | Test health check de un provider |

### 10.13 Evaluation System (21 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/evaluation/overview` | Overview combinado de 5 sistemas |
| `GET` | `/evaluation/quality/stats` | Estadísticas de calidad |
| `GET` | `/evaluation/quality/reports` | Reportes de calidad recientes |
| `GET` | `/evaluation/quality/{id}` | Reporte de calidad por interacción |
| `GET` | `/evaluation/alignment/stats` | Estadísticas de alignment |
| `GET` | `/evaluation/alignment/reports` | Reportes de alignment recientes |
| `GET` | `/evaluation/legal/stats` | Estadísticas de riesgo legal |
| `GET` | `/evaluation/legal/reports` | Reportes de riesgo legal |
| `GET` | `/evaluation/legal/flagged` | Assessments por encima de threshold |
| `GET` | `/evaluation/decisions/stats` | Stats del registro de decisiones |
| `GET` | `/evaluation/decisions` | Lista decisiones (filtro status/category) |
| `GET` | `/evaluation/decisions/pending` | Decisiones pendientes de aprobación |
| `PUT` | `/evaluation/decisions/{id}/status` | Actualizar estado (approve/reject/defer) |
| `POST` | `/evaluation/decisions/register` | Registrar decisión manualmente |
| `GET` | `/evaluation/rollback/stats` | Stats de rollback |
| `GET` | `/evaluation/rollback/log` | Log de operaciones de memoria |
| `POST` | `/evaluation/rollback/undo/{id}` | Deshacer operación de memoria |
| `GET` | `/evaluation/rollback/checkpoints` | Checkpoints de memoria |
| `POST` | `/evaluation/rollback/checkpoint` | Crear checkpoint |
| `POST` | `/evaluation/rollback/checkpoint/{id}/rollback` | Rollback a checkpoint |
| `DELETE` | `/evaluation/data` | Eliminar TODOS los datos de evaluación |

### 10.14 Governance (6 endpoints)

| Método | Path | Descripción | Eventos |
|--------|------|-------------|---------|
| `GET` | `/governance/config` | Configuración YAML | — |
| `GET` | `/governance/audit-log` | Log de auditoría reciente | — |
| `GET` | `/governance/approvals` | Aprobaciones pendientes (risk high/critical) | — |
| `POST` | `/governance/emergency-stop` | **EMERGENCY STOP** — detiene todo | `governance.emergency_stop_activated` (critical) |
| `POST` | `/governance/emergency-resume` | Reanudar operaciones | `governance.emergency_stop_deactivated` |
| `GET` | `/governance/emergency-status` | Estado del emergency stop | — |

### 10.15 Analytics (6 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/analytics/overview` | Eventos por tipo/agente/riesgo |
| `GET` | `/analytics/identity-fidelity` | Score de fidelidad (heurístico basado en correcciones) |
| `GET` | `/analytics/autonomy` | Métricas de autonomía (auto vs escalated) |
| `GET` | `/analytics/token-usage` | Token usage últimos 30 días |
| `DELETE` | `/analytics/events` | Eliminar audit_log |
| `DELETE` | `/analytics/tokens` | Eliminar token_usage |

### 10.16 Identity Governance (11 endpoints)

| Método | Path | Descripción | Eventos |
|--------|------|-------------|---------|
| `GET` | `/identity/versions` | Listar snapshots de versión | — |
| `GET` | `/identity/versions/{id}` | Detalle de un snapshot | — |
| `POST` | `/identity/snapshot` | Crear snapshot manual | — |
| `POST` | `/identity/activate/{id}` | Activar versión (pointer swap, no-destructivo). Idempotency guard | `identity.version_activated` |
| `POST` | `/identity/rollback/{id}` | **ROLLBACK ESTRUCTURAL** — invalida snapshots posteriores | `identity.version_rolled_back` (high) |
| `DELETE` | `/identity/versions/{id}` | Eliminar snapshot (no si es activo → 409) | — |
| `GET` | `/identity/evolution` | Candidatos de evolución recientes | — |
| `POST` | `/identity/evolution/{id}/approve` | Aprobar evolución → crear snapshot | `identity.evolution_approved` |
| `POST` | `/identity/evolution/{id}/reject` | Rechazar evolución → audit log | `identity.evolution_rejected` |
| `GET` | `/identity/shadow` | Resultados de shadow simulation | — |
| `GET` | `/identity/health` | Señales de salud de identidad + latest health monitor | — |

### 10.17 Middleware (2 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/middleware/list` | Lista middleware registrado |
| `POST` | `/middleware/{name}/toggle` | Toggle enable/disable |

### 10.18 Teleology / Goals (11 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/goals` | Listar metas (filtro status/type) |
| `GET` | `/goals/{id}` | Detalle de una meta |
| `POST` | `/goals` | Crear meta (title, description, type) |
| `PUT` | `/goals/{id}` | Actualizar meta |
| `DELETE` | `/goals/{id}` | Eliminar meta |
| `POST` | `/goals/{id}/activate` | CREATED → ACTIVE |
| `POST` | `/goals/{id}/complete` | Marcar completada |
| `POST` | `/goals/{id}/cancel` | Cancelar meta |
| `GET` | `/goals/conflicts/detect` | Detectar conflictos entre metas activas |
| `GET` | `/goals/rewards/recent` | Señales de recompensa recientes |
| `POST` | `/goals/{id}/governance` | Evaluar governance de una meta |

---

## 11. Dashboard — 15 Páginas

### 11.1 Command Center — `/`

**Archivo**: `app/page.tsx` (207 ln)
**Propósito**: Panel principal con KPIs, estado del sistema y acciones rápidas.

**Información mostrada**:

| Sección | Fuente de datos | Componente |
|---------|----------------|------------|
| 8 KPI Cards (Conversations, Fidelity, Autonomy, Training, Skills, Cost, Memory, Uptime) | `systemStatus` del Zustand store (polling cada 5s via `useHealth()`) | `KPICard` |
| Agent Status Ring (rotación animada, 7 estados: Ready/Thinking/Acting/Awaiting/Training/Error/Offline) | `agentState` del store (actualizado via WebSocket) | `AgentStatusRing` |
| Health Bar (7 LEDs: Database, LLM, WebSocket, Router, Governance, Storage, Security) | `health`, `wsConnected`, `routerStatus`, `systemStatus` del store | `HealthBar` |
| Activity Feed (timeline de eventos con detalle expandible) | `events` del store (WebSocket real-time) | `ActivityFeed` |
| Quick Actions (7 botones de acción directa) | — | `QuickActions` |
| Persona Card (Big Five mini-bars + valores) | `persona` del store | `PersonaCard` |
| Router Card (fallback chain + circuit breakers + agent assignments) | `routerStatus`, `health` del store | `RouterCard` |

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Start Chat** (link) | Navega a `/chat` |
| **Start Training** (botón) | `api.startTrainingSession("correction")` → `POST /training/session/start` → navega a `/training` |
| **Reload Persona** (botón) | `api.reloadPersona()` → `POST /persona/reload` → `api.personaInfo()` → actualiza store |
| **Reload Router** (botón) | `api.reloadRouter()` → `POST /router/reload` → `api.routerStatus()` → actualiza store |
| **Identity Governance** (link) | Navega a `/identity-governance` |
| **Pending Approvals** (badge condicional) | Navega a `/governance` (solo visible si hay approvals pendientes) |
| **Emergency Stop** (botón rojo, confirm 2-click) | `api.emergencyStop()` → `POST /governance/emergency-stop` → emite `governance.emergency_stop_activated` → orchestrator bloquea todo procesamiento |
| **Emergency Resume** (botón verde, confirm 2-click) | `api.emergencyResume()` → `POST /governance/emergency-resume` → orchestrator resume |

### 11.2 Chat — `/chat`

**Archivos**: `app/chat/page.tsx` (14 ln) + `components/chat/chat-panel.tsx` (199 ln) + `components/chat/message-bubble.tsx` (100 ln)
**Propósito**: Interfaz de conversación con el delegado.

**Información mostrada**:

| Sección | Descripción |
|---------|-------------|
| Header | Título, badge de `conversation_id`, badge del último modelo usado |
| Message bubbles | avatar (User/Bot), contenido (whitespace-preserving), footer con: multi-agent indicator, modelo, cognitive mode icon, knowledge source indicators (🧠 Memory / 🌐 General) |
| Streaming indicator | Loader spinner durante procesamiento |
| Cognitive Mode badge | 🟢 Full / 🟡 Memory+LLM / 🔴 Memory Only |

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Input de texto + Enter/Send** | `api.chat(text, conversationId, cognitiveMode)` → `POST /api/chat` → orchestrator pipeline completo → respuesta con `content`, `provider`, `model`, `knowledge_sources`, `multi_agent` → se agrega al store `messages` → se guarda `conversationId` |
| **Cognitive Mode toggle** (botón cíclico) | `setCognitiveMode()` cicla 1→2→3→1 en Zustand store. Mode 1: Full (web + LLM + all memory). Mode 2: Memory+LLM (default, grounded en memoria recalled). Mode 3: Memory Only (NO LLM, solo recall) |
| **Clear** (botón) | `clearChat()` en store — limpia `messages[]`, `conversationId` |

**Qué pasa cuando envías un mensaje (end-to-end)**:

1. UI → `api.chat(text, conversationId, cognitiveMode)` → `POST /api/chat`
2. Backend: `InputSanitizer().sanitize()` — detecta threats (XSS, SQL injection, prompt injection)
3. Backend: `database.create_conversation()` (si nuevo) → Postgres `conversations`
4. Backend: `database.save_message()` (user) → Postgres `messages`
5. Backend: emite `agent_state.thinking` → WebSocket → Dashboard Agent Ring se pone azul
6. Backend: `database.get_conversation_messages()` → últimos 20 mensajes de historial
7. Backend: emite `agent_state.acting` → WebSocket → Dashboard Agent Ring se pone amber
8. Backend: `orchestrator.process(message, conversation_id, history, cognitive_mode)` → pipeline de 25+ pasos
9. En el pipeline: classify → decision engine → identity modules → planner → memory recall → corrections → prompt build → LLM generate → identity review → governance review → memory store → evaluations → persistence → health monitor → evolution → shadow simulation → version control
10. Backend: `database.save_message()` (assistant) → Postgres `messages`
11. Backend: emite `chat.response_generated` → WebSocket
12. Backend: emite `agent_state.idle` → WebSocket → Dashboard Agent Ring vuelve a verde
13. Backend: retorna `ChatResponse` JSON → Dashboard
14. Dashboard: agrega message bubble, actualiza `conversationId`, muestra modelo/knowledge sources

### 11.3 Identity Studio — `/identity`

**Archivo**: `app/identity/page.tsx` (447 ln)
**Propósito**: Editor visual de la personalidad, valores y estilo de comunicación del principal.

**Carga inicial**: `api.personaInfo()` → `GET /persona/info` → Big Five, comm values, values list, boundaries.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **5 Big Five sliders** (0.0-1.0) | Estado local `traits`. Mueve valores en tiempo real con radar chart SVG |
| **Save Personality** (botón) | `api.updateTraits(traits)` → `PUT /persona/traits` → escribe `persona.yaml` → `agent_crew.reload_persona()` → todos los agentes actualizan system prompt → emite `config.traits_updated` |
| **6 Communication sliders** (formality, directness, verbosity, emotional_expression, humor, empathy) | Estado local `comm` |
| **Save Communication** (botón) | `api.updateCommunication(comm)` → `PUT /persona/communication` → escribe YAML → reload → emite `config.communication_updated` |
| **Values list** (drag-and-drop para reordenar, input para agregar, botón X para eliminar) | Estado local `values`. Drag reordena prioridades |
| **Save Values** (botón) | `api.updateValues({values})` → `PUT /persona/values` → escribe YAML → reload → emite `config.values_updated` |
| **Boundaries list** (input + agregar, X para eliminar) | Estado local `boundaries` |
| **Save Boundaries** (botón) | `api.updateBoundaries({boundaries})` → `PUT /persona/boundaries` → escribe YAML → reload → emite `config.boundaries_updated` |
| **Reload from file** (botón) | `api.reloadPersona()` → `POST /persona/reload` → relee persona.yaml → reload agents → re-fetch info |
| **View Versions →** (link) | Navega a `/identity-governance` |

**Efecto en Django**: Cada save actualiza `persona.yaml` y recarga TODOS los agentes. Esto significa que al cambiar un slider de personalidad y guardar, la siguiente respuesta de Django reflejará la nueva personalidad. El `IdentityManager` detectará el cambio de SHA-256 hash y actualizará el `IdentityProfile` y baseline embedding.

### 11.4 Training Center — `/training`

**Archivo**: `app/training/page.tsx` (1,125 ln)
**Propósito**: Entrenar al delegado con 3 modos de aprendizaje.

**Carga inicial**: `api.trainingStatus()`, `api.trainingHistory()`, `api.listCorrections()`.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Mode selector** (3 botones) | Correction / Free Conversation / Guided Interview — estado local |
| **Start Session** | `api.startTrainingSession(mode)` → `POST /training/session/start` |
| **End Session** | `api.endTrainingSession()` → `POST /training/session/end` |
| **Correction form** (original + corrected + feedback + submit) | `api.submitCorrection(data)` → `POST /training/correction` → guarda en SQLite procedural → genera suggestion → se inyecta en cada prompt futuro |
| **Free conversation chat** (input + send) | `api.trainingExchange(msg)` → `POST /training/exchange` → orchestrator pipeline completo → LLM extrae rasgos de personalidad → guarda en ChromaDB semantic memory |
| **Guided interview** (answer textarea + submit) | `api.submitInterviewAnswer(qId, answer)` → `POST /training/interview/answer` → analiza respuesta para rasgos → guarda en ChromaDB |
| **Interview navigation** (prev/next/skip, dot nav) | Estado local, 15 preguntas |
| **File upload drag area** | `api.uploadWritingSamples(files)` → `POST /training/upload-samples` (FormData) → chunk ~500 palabras → ChromaDB semantic como `writing_sample` |
| **Edit correction** (inline) | `api.updateCorrection(id, data)` → `PUT /training/corrections/{id}` → SQLite update → reload suggestions |
| **Delete correction** | `api.deleteCorrection(id)` → `DELETE /training/corrections/{id}` → SQLite delete → reload |
| **Delete suggestion** | `api.deleteSuggestion(idx)` → `DELETE /training/suggestions/{idx}` → remove in-memory |
| **Clear all suggestions** | `api.clearAllSuggestions()` → `DELETE /training/suggestions` → clear in-memory list |

**Efecto en Django**: Las correcciones se inyectan como "behavioral suggestions" en el prompt de cada interacción futura (paso 5 del pipeline). Los writing samples alimentan el RAG semántico. La free conversation extrae rasgos del habla natural del principal.

### 11.5 Testing Playground — `/testing`

**Archivo**: `app/testing/page.tsx` (391 ln)
**Propósito**: Probar al delegado con escenarios controlados.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Chat Simulator** (input + send) | `api.chat(msg)` → `POST /chat` — mismo pipeline que chat normal |
| **6 Built-in Scenarios** (botón Run per scenario) | `api.chat(scenario.prompt)` → `POST /chat` — evalúa respuesta del delegado en situaciones específicas |
| **Run All** (botón) | Ejecuta los 6 escenarios secuencialmente |
| **A/B Compare** (textarea + Compare) | `api.chat(prompt)` × 2 en paralelo → muestra respuestas side-by-side para comparar consistencia |

### 11.6 Model Manager — `/models`

**Archivo**: `app/models/page.tsx` (403 ln)
**Propósito**: Gestionar proveedores LLM, perfiles y asignaciones.

**Carga inicial**: `api.modelsConfig()` + `api.routerStatus()`.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Refresh** | Re-fetch config + status |
| **Test model** (per provider) | `api.testModel(modelId)` → `POST /models/test` → envía "Respond with only 'OK'" → muestra latency + resultado |
| **Profile selector** (4 botones) | `api.switchProfile(profile)` → `PUT /models/profile` → escribe models.json → `model_router.reload_config()` → muestra effective assignments. balanced: asignaciones default. max_quality: todo Gemini. privacy_mode: todo Ollama. budget_mode: todo Ollama ligero |
| **Agent assignment dropdowns** | `api.updateModelAssignment(role, modelId)` → `PUT /models/assignment` → escribe models.json → reload_config |
| **Reload from file** | `api.reloadRouter()` → `POST /router/reload` |

**Datos mostrados**: Provider status cards (online/offline, circuit breaker LEDs), fallback chain arrows, agent→model table, test results.

### 11.7 Skill Manager — `/skills`

**Archivo**: `app/skills/page.tsx` (~300 ln)
**Propósito**: Gestionar habilidades del delegado.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Search/filter** | Filtro client-side |
| **Toggle skill** (per skill) | `api.toggleSkill(id, enabled)` → `POST /skills/{id}/toggle` → actualiza skills.json → emite `config.skill_toggled` |
| **Learn Topic** (topic + depth 1-3 + Learn) | `api.learnTopic(topic, depth)` → `POST /skills/learn-topic` → Tavily search → LLM summarize → chunk → ChromaDB semantic memory como `learned_knowledge` |
| **Explore Repo** (target + type + store toggle) | `api.exploreRepo(target, type, store)` → `POST /skills/explore-repo` → lee archivos locales/GitHub/docs → context injection o storage en ChromaDB |

### 11.8 Memory Lab — `/memory`

**Archivo**: `app/memory/page.tsx` (692 ln)
**Propósito**: Explorar, buscar, crear, editar y eliminar memorias del delegado.

**Carga inicial**: `api.memoryStats()`.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Refresh** | `api.memoryStats()` → `GET /memory/stats` → counts de 4 tiers |
| **Search** (query + tier filters + limit) | `api.memorySearch(query, limit, tiers)` → `POST /memory/search` → ChromaDB cosine similarity → resultados agrupados por tier |
| **Tier filter** (working/episodic/semantic toggles) | Filtra búsqueda a tiers específicos |
| **Delete semantic** | `api.deleteSemanticMemory(id)` → `DELETE /memory/semantic/{id}` → ChromaDB delete + rollback audit record |
| **Delete episodic** | `api.deleteEpisodicMemory(id)` → `DELETE /memory/episodic/{id}` → ChromaDB delete + rollback audit |
| **Bulk delete** | `api.bulkDeleteMemories(items)` → `POST /memory/bulk-delete` → batch delete + audit per item |
| **Edit semantic** (inline content + category) | `api.updateSemanticMemory(id, content, category)` → `PUT /memory/semantic/{id}` → ChromaDB update + rollback audit |
| **Store new memory** (content + category + Store) | `api.storeSemanticMemory(content, category, "dashboard")` → `POST /memory/semantic/store` → ChromaDB add + rollback audit |

**Efecto en Django**: Agregar/editar/eliminar memorias afecta directamente lo que Django "recuerda". Una nueva memoria semántica aparecerá en futuros recalls. Una eliminación significa que Django olvidará esa información.

### 11.9 Governance Console — `/governance`

**Archivo**: `app/governance/page.tsx` (572 ln)
**Propósito**: Visualizar y controlar las reglas de gobernanza del delegado.

**Carga inicial**: `api.emergencyStatus()`, `api.governanceConfig()`, `api.governanceAuditLog(100)`, `api.governanceApprovals()`, `api.listMiddleware()`.

**5 tabs**: Overview / Audit Log / Approvals / Boundaries / Middleware.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Emergency Stop** (confirm dialog) | `api.emergencyStop()` → `POST /governance/emergency-stop` → `orchestrator.emergency_stop()` → bloquea TODA interacción. Dashboard muestra banner rojo |
| **Emergency Resume** (confirm dialog) | `api.emergencyResume()` → `POST /governance/emergency-resume` → resume operaciones |
| **Approval action buttons** | **NOTA**: Actualmente solo dismisses del estado local React. No hacen API call al backend — gap funcional |
| **Middleware toggle** | `api.toggleMiddleware(name)` → `POST /middleware/{name}/toggle` → habilita/deshabilita middleware específico |
| **Section toggles** (autonomy, risk) | Expand/collapse de secciones de config (local) |

**Datos mostrados**: Autonomy levels table (0-4), risk classification matrix, forbidden operations list, data privacy rules, audit log timeline, pending approvals, middleware pipeline list.

### 11.10 Analytics Dashboard — `/analytics`

**Archivo**: `app/analytics/page.tsx` (330 ln)
**Propósito**: Métricas de rendimiento y uso del delegado.

**Carga inicial**: `api.analyticsOverview()`, `api.analyticsIdentityFidelity()`, `api.analyticsAutonomy()`, `api.analyticsTokenUsage()`.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Refresh** | Re-fetch 4 APIs de analytics |
| **Clear All Data** (confirm) | `api.analyticsDeleteEvents()` → `DELETE /analytics/events` (borra audit_log) + `api.analyticsDeleteTokens()` → `DELETE /analytics/tokens` (borra token_usage + in-memory log) |

**Datos mostrados**: 4 KPI cards (total events, unique types, interactions, errors), identity fidelity bar chart (heurístico basado en correcciones: baseline 85% + 1.5% per correction, cap 98%), events by type/agent/risk mini charts, token usage.

### 11.11 Cognitive Trace — `/trace`

**Archivo**: `app/trace/page.tsx` (703 ln)
**Propósito**: Visualizar el pipeline de procesamiento como grafos React Flow.

**Carga inicial**: `api.traceList(50)` + `api.traceLatest()`.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Replay** (input + Send) | `api.traceReplay(message)` → `POST /trace/replay` → ejecuta pipeline completo → retorna grafo con nodos y edges |
| **Trace list item click** | `api.traceDetail(id)` → `GET /trace/{id}` → carga grafo de traza específico en canvas React Flow |
| **Delete trace** | `api.traceDelete(id)` → `DELETE /trace/{id}` → elimina de Postgres + in-memory store |
| **Delete all** (confirm) | `api.traceDeleteAll()` → `DELETE /trace` → elimina TODO de Postgres |
| **Load latest** | `api.traceLatest()` → `GET /trace/latest/graph` |
| **Prompt viewer modal** | Muestra user prompt y system prompt de la traza. Tabs user/system. Copy to clipboard |
| **React Flow canvas** | Zoom, pan, minimap, controles. Nodos expandibles con input/output/metrics/errors |

**32 tipos de nodo**: Cada paso del pipeline se visualiza como un nodo coloreado (input=blue, classify=purple, decision_engine=orange, planner=sky, memory_recall=cyan, correction=amber, prompt_build=indigo, llm_generate=emerald, identity_review=rose, governance_review=yellow, memory_store=teal, evaluation=violet, output=green, branch=red, multi_agent=fuchsia, identity_decision_modulation=pink, identity_confidence=pink, identity_autonomy=pink, identity_bias=fuchsia, identity_memory_bridge=rose, identity_retrieval_weight=rose, identity_context_weight=pink, identity_prompt_inject=fuchsia, identity_consolidation=pink, identity_drift=red, identity_policy=red, identity_feedback=orange, identity_health_monitor=lime, identity_health_regulation=lime, identity_evolution=amber, identity_shadow=slate, identity_version_candidate=emerald). Nodos expandibles muestran processing summary, input, output, metrics, errors.

### 11.12 Evaluation Dashboard — `/evaluation`

**Archivo**: `app/evaluation/page.tsx` (874 ln)
**Propósito**: Dashboard detallado de los 5 módulos de evaluación.

**Carga inicial**: `api.evaluationOverview()`. Tabs cargan lazy.

**6 tabs**: Overview / Quality / Alignment / Legal Risk / Decisions / Rollback.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Tab switch** | Lazy load de datos por tab: Quality→`api.qualityReports()`, Alignment→`api.alignmentReports()`, Legal→`api.legalReports()`, Decisions→`api.decisionsList()`, Rollback→`api.rollbackLog()` + `api.rollbackCheckpoints()` |
| **Delete all evaluation data** (confirm) | `api.evaluationDeleteAll()` → `DELETE /evaluation/data` → elimina de Postgres + limpia 5 stores in-memory |
| **Row expand/collapse** | Detalle expandible por reporte |
| **Decision: Approve/Reject/Defer** | `api.decisionsUpdateStatus(id, status, "principal")` → `PUT /evaluation/decisions/{id}/status` → actualiza in-memory status |
| **Undo rollback** | `api.rollbackUndo(id)` → `POST /evaluation/rollback/undo/{id}` → restaura memoria desde before-state |
| **Create checkpoint** | `api.rollbackCreateCheckpoint(name, desc)` → `POST /evaluation/rollback/checkpoint` → snapshot in-memory |

**Datos mostrados**: Quality dimension score bars (5 dimensiones × A-F), alignment breakdown (values, communication, boundaries, decisions, identity_similarity con drift severity + policy action badges computados client-side), legal risk flags (6 categorías), decision registry (status, risk, category, rationale), rollback operations (action type, tier, rolled_back status), checkpoints.

### 11.13 Identity Governance — `/identity-governance`

**Archivo**: `app/identity-governance/page.tsx` (68 ln shell) + 4 sub-componentes + modal = 1,836 ln total
**Propósito**: Gestionar versiones de identidad, evolución, shadow simulation y salud.

**4 tabs**:

**Versions Tab** (543 ln):

| Control | Acción Interna |
|---------|---------------|
| **Create Snapshot** (modal con version, label, tags, notes) | `api.identityCreateSnapshot(data)` → `POST /identity/snapshot` → `IdentityVersionControl.create_snapshot()` → Postgres `config_versions` |
| **Activate version** (confirm warning) | `api.identityActivateVersion(id)` → `POST /identity/activate/{id}` → pointer swap del IdentityProfile activo. Recrea DecisionEngine. Actualiza AlignmentEvaluator baseline. Non-destructivo. Guard de idempotencia |
| **Rollback version** (confirm danger) | `api.identityRollbackVersion(id)` → `POST /identity/rollback/{id}` → invalida TODOS los snapshots posteriores, crea rollback_marker, swap de profile. **Destructivo** |
| **Delete snapshot** (confirm, solo no-activo) | `api.identityDeleteVersion(id)` → `DELETE /identity/versions/{id}` → guard: no delete active (409) |

**Evolution Tab** (287 ln):

| Control | Acción Interna |
|---------|---------------|
| **Approve evolution** | `api.identityEvolutionApprove(id)` → `POST /identity/evolution/{id}/approve` → crea snapshot en Postgres con metadata de evolución + emite `identity.evolution_approved` |
| **Reject evolution** | `api.identityEvolutionReject(id)` → `POST /identity/evolution/{id}/reject` → emite `identity.evolution_rejected` vía EventBus singleton |

**Shadow Tab** (305 ln): Solo visualización (refresh button). Muestra risk grades (safe/cautious/risky/destabilizing), comparison tables, structural diffs.

**Health Tab** (328 ln): Solo visualización (refresh button). Muestra instability_index, avg similarity/confidence, drift rate, CSS bar charts de timeline de señales, tabla de 10 señales recientes.

### 11.14 Goals / Teleology — `/goals`

**Archivo**: `app/goals/page.tsx` (362 ln)
**Propósito**: Gestionar metas y propósitos del delegado.

**Carga inicial**: `api.listGoals()`, `api.detectConflicts()`, `api.recentRewards(20)`.

**3 tabs**: Goals / Conflicts / Rewards.

**Controles y acciones**:

| Control | Acción Interna |
|---------|---------------|
| **Create Goal** (title + description + type selector + Create) | `api.createGoal(data)` → `POST /goals` → `goal_manager.create_goal()` |
| **Activate** (per goal, solo draft) | `api.activateGoal(id)` → `POST /goals/{id}/activate` → CREATED → ACTIVE |
| **Complete** (per goal, solo active) | `api.completeGoal(id)` → `POST /goals/{id}/complete` → ACTIVE → COMPLETED |
| **Cancel** (per goal) | `api.cancelGoal(id)` → `POST /goals/{id}/cancel` → → CANCELLED |
| **Delete** (per goal) | `api.deleteGoal(id)` → `DELETE /goals/{id}` |

---

## 12. Componentes del Dashboard

### 12.1 Layout (3 componentes)

| Componente | Archivo | Función |
|-----------|---------|---------|
| `ClientShell` | `components/layout/client-shell.tsx` (23 ln) | Root shell: invoca `useWebSocket()`, `useHealth(5000)`, `useInitialData()`. Wraps en `I18nProvider` + `TooltipProvider` |
| `Header` | `components/layout/header.tsx` (82 ln) | Barra superior: nombre principal, language switcher (ES/EN), DB status LED, provider count, WS status, agent state badge |
| `Sidebar` | `components/layout/sidebar.tsx` (161 ln) | 240px sidebar: logo, status LED, 4 grupos nav (Core, Identity & Training, Infrastructure, Observability), 14 nav items, version footer |

### 12.2 Command Center (7 componentes)

| Componente | Archivo | Líneas | Función |
|-----------|---------|--------|---------|
| `KPICard` | `kpi-card.tsx` | 75 | Card con valor grande, trend indicator, icono |
| `AgentStatusRing` | `agent-status-ring.tsx` | 87 | Ring animado 112×112px con "IA" central + estado |
| `HealthBar` | `health-bar.tsx` | 153 | 7 LEDs de estado del sistema con tooltips |
| `ActivityFeed` | `activity-feed.tsx` | 349 | Timeline de eventos expandibles + DecisionCard sub-component |
| `PersonaCard` | `persona-card.tsx` | 92 | Big Five mini-bars + valores como badges |
| `QuickActions` | `quick-actions.tsx` | 165 | 7 action buttons con API calls directos |
| `RouterCard` | `router-card.tsx` | 117 | Fallback chain + circuit breaker LEDs + agent assignments |

### 12.3 Chat (2 componentes)

| Componente | Archivo | Función |
|-----------|---------|---------|
| `ChatPanel` | `chat-panel.tsx` (199 ln) | Chat completo: input, send, mode toggle, clear, auto-scroll |
| `MessageBubble` | `message-bubble.tsx` (100 ln) | Bubble con avatar, content, footer (model, mode, sources) |

### 12.4 Trace (1 componente)

| Componente | Archivo | Función |
|-----------|---------|---------|
| `TraceNodeComponent` | `trace-node.tsx` (408 ln) | Nodo React Flow custom: 32 tipos con color/icon mapping, expandible con input/output/metrics |

### 12.5 Identity Governance (5 componentes)

| Componente | Archivo | Líneas | Función |
|-----------|---------|--------|---------|
| `VersionsTab` | `versions-tab.tsx` | 543 | CRUD de versiones con activate/rollback/delete + confirm dialogs |
| `EvolutionTab` | `evolution-tab.tsx` | 287 | Approve/reject evolution candidates con risk badges |
| `ShadowTab` | `shadow-tab.tsx` | 305 | Read-only: risk grades, comparison tables, structural diffs |
| `HealthTab` | `health-tab.tsx` | 328 | Read-only: CSS bar charts, signals timeline, health metrics |
| `CreateSnapshotModal` | `create-snapshot-modal.tsx` | 305 | Modal: version input, tags, label, notes, validation |

### 12.6 UI Primitives (10 archivos)

| Componente | Tipo | Notas |
|-----------|------|-------|
| `Badge` | Custom (CVA) | 6 variantes: default, success, warning, destructive, info, outline |
| `Button` | shadcn/ui | 6 variantes × 4 sizes, Radix Slot support |
| `Card` | shadcn/ui | Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter |
| `ConfirmDialog` | Custom | 3 variantes (danger/warning/info), backdrop blur, focus trap |
| `Input` | shadcn/ui | Lab-themed borders, focus ring |
| `Progress` | shadcn/ui (Radix) | Horizontal bar with translateX animation |
| `ScrollArea` | shadcn/ui (Radix) | Custom scrollbar thumb |
| `Separator` | shadcn/ui (Radix) | Horizontal/vertical |
| `Tabs` | shadcn/ui (Radix) | Tabs, TabsList, TabsTrigger, TabsContent |
| `Tooltip` | shadcn/ui (Radix) | Tooltip, TooltipTrigger, TooltipContent, TooltipProvider |

### 12.7 Estado Global — Zustand Store (159 líneas)

| Grupo | Estado | Persistido |
|-------|--------|------------|
| Health | `health: HealthResponse` | No |
| System | `systemStatus: SystemStatus` | No |
| Agent | `agentState` (7 estados) | No |
| Chat | `messages[]`, `conversationId`, `isStreaming`, `cognitiveMode` | **Sí** (localStorage `iame-chat`) |
| Events | `events[]` (max 100) | No |
| Persona | `persona: PersonaInfo` | No |
| Router | `routerStatus: RouterStatus` | No |
| WebSocket | `wsConnected: boolean` | No |

### 12.8 Hooks

| Hook | Função |
|------|--------|
| `useHealth(5000)` | Polling: `api.health()` + `api.systemStatus()` cada 5s → hydrate store |
| `useInitialData()` | One-shot: carga `events`, `persona`, `routerStatus` en paralelo |
| `useWebSocket()` | WS a `ws://localhost:8000/api/ws` — auto-reconnect (3s), keepalive (30s), parsea eventos, actualiza `agentState` |

### 12.9 i18n

631 keys en 17 secciones, EN + ES fully symmetric. Provider: `I18nProvider` en `client-shell.tsx`. Hook: `useTranslation()` → `{ locale, setLocale, t(key) }`. Persiste locale en localStorage.

---

## 13. Bot de Discord — scripts/discord_bot.py (572 líneas)

### Arquitectura

- **Dependencies**: discord.py, httpx (ambos en agent/venv)
- **Token**: `DISCORD_BOT_TOKEN` en `.env`
- **Task**: `Discord: Django Bot` en tasks.json
- **PID file**: `agent/discord_bot.pid` (atexit cleanup)
- **API**: Todas las interacciones via `POST /api/chat` al backend

### Flujo de Interacción

1. User escribe en Discord → `on_message()`
2. Track user en `known_users` (display_name, username, message_count)
3. Almacena mensaje en `channel_buffers[channel_id]` (deque maxlen=30)
4. Si es comando (!reset, !who, !full, !memory) → handle directamente
5. Si debe responder (mention, reply, name trigger, o LLM decide):
   - Build `GROUP_CONTEXT_PROMPT` con: servidor, canal, miembros, últimos 30 mensajes
   - `POST /api/chat` con: message tagged como "[user_name]", `conversation_id` per-channel, `cognitive_mode=2`, context
   - Si respuesta contiene `[SILENT]` → no responde
   - Si no → simula typing → split en chunks ≤2000 chars → envía
6. Cooldown: `MIN_RESPONSE_GAP = 8s` entre respuestas por canal

### Características Clave

- **Per-channel conversations**: Cada canal Discord tiene su propio `conversation_id` y working memory aislada
- **Member awareness**: `guild.chunk()` al startup + `build_member_list()` inyectado en cada prompt
- **Real-time context**: GROUP_CONTEXT_PROMPT overrides Knowledge Status restrictions
- **Cognitive mode 2**: Memory+LLM para todas las interacciones Discord
- **Comandos**: `!full <msg>` (mode 1), `!memory <msg>` (mode 3), `!reset`/`!nueva`/`!new`, `!who`/`!quien`

### Discord Webhook — scripts/discord_post.py (188 líneas)

Publisher a canal `#updates-django` via webhook URL en `.env`.
- Texto plano: auto-split a 2000 chars
- `--embed`: Rich embed con título y color
- `--file`: Leer contenido de archivo

---

## 14. Base de Datos

### 14.1 Postgres (Neon) — 15 tablas en 4 grupos

| Grupo | Tablas | Propósito |
|-------|--------|-----------|
| **Core** (5) | `conversations`, `messages`, `audit_log`, `kpi_snapshots`, `config_versions` | Datos de conversación, auditoría, configuración |
| **Persistence** (5) | `interactions`, `trace_nodes`, `evaluations`, `token_usage`, `memory_operations` | Artefactos del orchestrator |
| **Hybrid Search** (1) | `episodic_search` (FTS mirror con tsvector + GIN index) | Full-text search |
| **Teleology** (4) | `goals`, `plans`, `reward_signals`, `goal_events` | Sistema de metas |

**Driver**: psycopg2 (sync, autocommit). **Totalmente opcional** — app degrada gracefully sin DB.

### 14.2 ChromaDB (local)

| Colección | Propósito |
|-----------|-----------|
| `episodic_memory` | Logs de interacción, búsqueda semántica |
| `semantic_memory` | Base de conocimiento, writing samples, learned_knowledge |

Embedding: all-MiniLM-L6-v2 (384 dimensiones). Persiste en disco (`chroma_data/`).

### 14.3 SQLite

Archivo `data/procedural.db`. Almacena correcciones de training y patrones procedurales.

### 14.4 PersistenceRepository (1,507 líneas)

Fire-and-forget — un fallo NUNCA afecta la respuesta al usuario. 25+ methods:
- **Write**: `save_interaction()`, `save_trace_nodes()`, `save_all_evaluations()`, `save_token_usage()`, `save_memory_operation()`, `store_identity_snapshot()`
- **Read**: evaluaciones, trazas, analytics, identidad
- **Delete**: evaluaciones, eventos, tokens
- Callbacks wired en ModelRouter y MemoryRollbackManager at startup

---

## 15. Sistema de Tests — 20,041 líneas, 47 archivos

| Archivo | Líneas | Módulo testeado |
|---------|--------|----------------|
| `test_api.py` | 135 | API routes básicos |
| `test_basics.py` | 59 | Smoke tests |
| `test_config.py` | 86 | Pydantic settings |
| `test_crew.py` | 104 | AgentCrew |
| `test_event_bus.py` | 115 | EventBus pub/sub |
| `test_cognition.py` | 235 | DecisionEngine + Planner |
| `test_orchestrator.py` | 161 | Pipeline orchestrator |
| `test_model_router.py` | 112 | ModelRouter fallback |
| `test_memory.py` | 279 | MemoryManager 4-tier |
| `test_skills.py` | 103 | SkillRegistry |
| `test_training.py` | 148 | TrainingManager |
| `test_security.py` | 267 | Input sanitizer + content wrapper |
| `test_circuit_breaker.py` | 219 | Circuit breaker states |
| `test_hybrid_search.py` | 325 | RRF + MMR + temporal decay |
| `test_compaction.py` | 228 | Memory compaction engine |
| `test_middleware.py` | 152 | MiddlewareRegistry pipeline |
| `test_parallel_executor.py` | 134 | ParallelExecutor + Aggregator |
| `test_identity_module.py` | 391 | IdentityProfile schema + embedding |
| `test_identity_core.py` | 182 | IdentityCoreAgent |
| `test_identity_enforcement.py` | 229 | Phase 5A enforcement |
| `test_identity_policy.py` | 280 | Phase 5B policy |
| `test_identity_feedback.py` | 336 | Phase 5C feedback |
| `test_identity_memory_bridge.py` | 610 | Phase 6A memory bridge |
| `test_identity_context_weighting.py` | 530 | Phase 6B context weighting |
| `test_identity_decision_modulation.py` | 607 | Phase 6C decision modulation |
| `test_identity_confidence.py` | 474 | Phase 6D confidence engine |
| `test_identity_autonomy_modulation.py` | 451 | Phase 7A autonomy modulation |
| `test_identity_retrieval_weighting.py` | 457 | Phase 7B retrieval weighting |
| `test_identity_consolidation_weighting.py` | 589 | Phase 7C consolidation weighting |
| `test_identity_behavioral_bias.py` | 607 | Phase 8A behavioral bias |
| `test_identity_prompt_integration.py` | 443 | Phase 8B prompt integration |
| `test_identity_health_monitor.py` | 696 | Phase 9A health monitor |
| `test_identity_health_regulation.py` | 544 | Phase 9B health regulation |
| `test_identity_evolution.py` | 670 | Phase 10A evolution engine |
| `test_identity_shadow_simulation.py` | 896 | Phase 10B shadow simulation |
| `test_identity_versioning.py` | 938 | Phase 10C version control |
| `test_identity_rollback.py` | 568 | Memory rollback + identity |
| `test_identity_governance.py` | 715 | Phase 10D governance UI |
| `test_identity_governance_hardening.py` | 420 | Phase 10D.1 hardening |
| `test_goals.py` | 336 | Goal CRUD + state machine |
| `test_plans.py` | 271 | Plans lifecycle |
| `test_priorities_rewards.py` | 213 | Priority engine + reward model |
| `test_background_conflicts.py` | 235 | Background loop + conflict detection |
| `test_teleology_api.py` | 320 | Goals API endpoints |
| `test_teleology_governance.py` | 147 | Teleology governance checks |
| `test_teleology_middleware.py` | 144 | Teleology middleware integration |

**Framework**: pytest. **conftest.py** (100 ln): fixtures compartidos.

---

## 16. Estado Actual vs Planificado

### ✅ Completado

| Phase | Descripción | Tests |
|-------|-------------|-------|
| 1-2 | 5-agent crew, 4-tier memory, Model Router, 14 pages dashboard, WebSocket, Event Bus | ~300 |
| 3 | Architectural Hardening — cognición obligatoria, inmutable, sin legacy | 49 |
| 3.5 | Learn-topic skill, 3 cognitive modes | ~30 |
| 4 | Identity Core — IdentityProfile, embedding 384-dim, versioning, SHA-256 | 47 |
| 5A-C | Enforcement, Policy, Feedback | 126 |
| 6A-D | Memory Bridge, Context Weighting, Decision Modulation, Confidence | 246 |
| 7A-C | Autonomy Modulation, Retrieval Weighting, Consolidation Weighting | 201 |
| 8A-B | Behavioral Bias, Prompt Integration | 168 |
| 9A-B | Health Monitor, Health Regulation | 212 |
| 10A-C | Evolution Engine, Shadow Simulation, Version Control | 337 |
| 10D-D.1 | Identity Governance UI + Hardening | 75 |
| 11 | Security (input sanitizer, content wrapper, middleware) | ~20 |
| 12 | Circuit Breaker (per-provider, 3 states) | ~19 |
| 13 | Memory Compaction Engine | ~17 |
| 14 | Hybrid Search (RRF + MMR + temporal decay) | 16 |
| 15 | Middleware Pipeline | 16 |
| 16 | Parallel Executor + Response Aggregator | 10 |
| 17 | Teleology (goals, plans, priorities, rewards, conflicts, governance, background) | ~130 |

### 🔲 Planificado

| Item | Prioridad | Descripción |
|------|-----------|-------------|
| **Human-in-the-Loop** | ALTA | Mecanismo de pausa para approval queue |
| **Supabase Auth** | ALTA | JWT + login/logout + rutas protegidas |
| **Memory Consolidation Background** | ALTA | Job periódico episodic → semantic summarization |
| **Real Identity Fidelity** | MEDIA | Ponderar identity_similarity en overall_score + dashboard gauge |
| **LLM-Based Classification** | MEDIA | Reemplazar keyword matching con clasificación semántica |
| **External Integrations** | MEDIA | Email send/receive, calendar, Slack, CRM |
| **Tool Policy Cascade** | MEDIA | Políticas por skill integradas con gobernanza |
| **Sandboxed Code Execution** | MEDIA | Ejecución de código en sandbox aislado |
| **File Operations** | MEDIA | Lectura/escritura de archivos con control de acceso |
| **Self-Healing Pipeline** | MEDIA | Detección → diagnóstico → desactivación → fallback → restauración |
| **QLoRA Fine-Tuning** | BAJA | PEFT + Unsloth para modelo privado |
| **Self-Modification** | BAJA | Acceso al codebase con governance |
| **Smart Model Selection** | BAJA | Routing task-aware (complejidad → modelo óptimo) |
| **Async DB Driver** | BAJA | Migrar psycopg2 sync → asyncpg |

---

## 17. Hoja de Ruta

**Objetivo final**: Un clon virtual autónomo capaz de emular a Harold Vélez de forma casi indistinguible en pensamiento, decisiones y uso de herramientas.

### Modelo de Autonomía Operativa

| Nivel | Capacidades | Estado |
|-------|-------------|--------|
| **Level 0** | Texto→texto, responde preguntas | ✅ Actual |
| **Level 1** | + Memoria persistente, evaluaciones, búsqueda semántica | ✅ Completado (Phases 11-14) |
| **Level 2** | + Skills activos, multi-agente, middleware pipeline | ✅ Completado (Phases 15-16) |
| **Level 3** | + Metas, planes, background loop, recompensas | ✅ Completado (Phase 17) — loop inactivo (autonomy_level=0) |
| **Level 4** | + Acciones proactivas background, self-healing, integrations | 🔲 Futuro |

### Integraciones Externas (Roadmap)

| Prioridad | Integración | Skill | Governance |
|-----------|-------------|-------|------------|
| **P1** | Email (Gmail/SMTP) | `email_send`, `email_read` | Approval para envío, audit trail |
| **P1** | Calendar (Google Calendar) | `calendar_read`, `calendar_write` | Confirmación para crear eventos |
| **P2** | Slack | `slack_message`, `slack_read` | Channel whitelist + approval |
| **P3** | CRM | `crm_read`, `crm_update` | Read libre, write con approval |
| **P3** | Document Storage (R2/S3) | `file_store`, `file_retrieve` | Size limits, type whitelist |

### Métricas de Éxito

| Métrica | Objetivo |
|---------|----------|
| Indistinguibilidad | >85% en test ciego |
| Fidelidad de identidad | >90% embedding similarity |
| Autonomía operativa | Nivel 3+ estable |
| Herramientas activas | ≥10 integraciones |
| Tiempo de respuesta | <5 segundos promedio |
| Costo mensual | <$50/mes |

### Riesgos Residuales

| ID | Riesgo | Mitigación |
|----|--------|------------|
| R1 | ChromaDB no escala a millones de docs | Migración a Qdrant/Pinecone si >100K docs |
| R2 | psycopg2 sync en handlers async | Migración a asyncpg planificada |
| R3 | Emergency stop es in-memory (se pierde en restart) | Persistir flag en DB |
| R4 | Evaluaciones heurísticas (no LLM) | Aceptable por costo $0, LLM eval como mejora futura |
| R5 | Multiplicación de tokens con multi-agent + teleología | Monitor analytics + circuit breaker |
| R6 | Reward hacking | Human-in-the-loop + auditoría periódica |
| R7 | Identity drift acumulativo | Parcialmente resuelto (22 módulos). Pending: hard enforcement |

---

## 18. Convenciones de Desarrollo

### Python (Backend)
- Lazy imports en handlers: `from src.api.main import get_state`
- Logger: `logging.getLogger("iame.module_name")`
- Endpoints async excepto donde sync es requerido
- Pydantic models para request/response
- Path constants relativos a `__file__`

### TypeScript (Dashboard)
- `"use client"` en todas las páginas interactivas
- Zustand para estado global, useState para local
- `api.ts` centraliza todas las llamadas al backend
- shadcn/ui para primitivos, custom para lab-themed
- Directiva de tema oscuro con variables CSS

### Tema Visual

Paleta de colores:
- **Lab**: bg `#0a0e1a`, surface `#111827`, card `#1a2035`, border `#1e293b`, text `#e2e8f0`, text-dim `#94a3b8`
- **Status**: green `#22c55e`, amber `#f59e0b`, red `#ef4444`, blue `#3b82f6`, purple `#a855f7`
- **Accent**: primary `#6366f1`, secondary `#8b5cf6`, glow `#818cf8`

Fonts: Inter (sans) + JetBrains Mono (mono). Animaciones: `pulse-led`, `slide-in`, `fade-up`.

### Reglas Operativas
- **Servicios SOLO via VS Code Tasks** — nunca terminales background
- **Terminales** solo para one-off commands (curl, pytest, pip install)
- **Kill terminals after use** — no acumular
- **Documentación**: actualizar este documento con cada cambio arquitectural

---

## 19. Glosario

| Término | Significado |
|---------|-------------|
| **ADLRA** | Autonomous Digital Learning Representative Agent |
| **Baseline Embedding** | Vector 384-dim que representa la identidad del principal |
| **Big Five** | Modelo psicológico OCEAN: Openness, Conscientiousness, Extraversion, Agreeableness, Neuroticism |
| **ChromaDB** | Base de datos vectorial local para búsqueda semántica |
| **Circuit Breaker** | Patrón que protege contra fallos cascada en proveedores LLM |
| **Cognitive Mode** | Nivel de inteligencia: Full (1), Memory+LLM (2), Memory Only (3) |
| **Cognitive Trace** | Grafo React Flow que registra cada paso del procesamiento |
| **Cosine Similarity** | Medida de similitud entre vectores (0=nada, 1=idénticos) |
| **Drift** | Desviación de personalidad respecto al baseline |
| **EventBus** | Sistema pub/sub para broadcast de eventos a WebSocket, DB y suscriptores |
| **Fire-and-forget** | Operación que se ejecuta sin esperar resultado ni bloquear |
| **Governance** | Marco de reglas que controla qué puede hacer el delegado |
| **Identity Profile** | Estructura Pydantic con toda la identidad formalizada |
| **Kill Switch** | Botón de emergencia que detiene todo procesamiento |
| **LLM** | Large Language Model (Gemini, Llama, etc.) |
| **MMR** | Maximal Marginal Relevance — diversificación de resultados |
| **Orchestrator** | Pipeline central que coordina todo el procesamiento |
| **Principal** | La persona que el delegado emula (Harold Vélez) |
| **QLoRA** | Técnica eficiente de fine-tuning con bajo uso de memoria |
| **RAG** | Retrieval-Augmented Generation — combinar búsqueda con generación |
| **RRF** | Reciprocal Rank Fusion — fusión de rankings |
| **Shadow Simulation** | Simulación paralela no-mutante de candidato de identidad |
| **Teleology** | Sistema de metas y propósitos del agente |

---

> **Fuentes**: Auditoría exhaustiva del repositorio completo (89 archivos backend, 15 páginas dashboard, 31 componentes, 47 test files, 4 configs, 2 scripts). Verificado contra codebase real.
> **Fecha de generación**: 21 de febrero de 2026
