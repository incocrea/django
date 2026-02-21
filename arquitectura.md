# Django — Arquitectura Completa del Sistema

> **Documento de referencia técnica para auditoría de arquitectura de conciencia virtual.**
> Este documento describe con total precisión el estado actual del sistema "Django" (no como el framework de phyton), un sistema multi-agente diseñado para aprender a representar la identidad, personalidad y estilo de toma de decisiones de su principal (humano) a través de entrenamiento progresivo, memoria contextual y retroalimentación humana directa.

> **🚨 REGLA CRÍTICA: Este documento DEBE actualizarse cada vez que se modifique cualquier aspecto de la arquitectura del proyecto.**

---

## TABLA DE CONTENIDOS

1. [Visión General del Sistema](#1-visión-general-del-sistema)
2. [Diagrama de Alto Nivel](#2-diagrama-de-alto-nivel)
3. [Stack Tecnológico](#3-stack-tecnológico)
4. [Capa de Configuración](#4-capa-de-configuración)
5. [Capa de Entrada — API REST + WebSocket](#5-capa-de-entrada--api-rest--websocket)
6. [Capa Cognitiva — DecisionEngine + Planner](#6-capa-cognitiva--decisionengine--planner)
7. [Capa de Orquestación — Pipeline de 10 Pasos](#7-capa-de-orquestación--pipeline-de-10-pasos)
8. [Capa de Agentes — Crew de 5 Agentes Especializados](#8-capa-de-agentes--crew-de-5-agentes-especializados)
9. [Sistema de Memoria — 4 Niveles](#9-sistema-de-memoria--4-niveles)
10. [Model Router — Cadena de Fallback LLM](#10-model-router--cadena-de-fallback-llm)
11. [Sistema de Evaluación — 5 Módulos Heurísticos](#11-sistema-de-evaluación--5-módulos-heurísticos)
12. [Cognitive Trace — Observabilidad del Pipeline](#12-cognitive-trace--observabilidad-del-pipeline)
13. [Event Bus — Pub/Sub + WebSocket Broadcast](#13-event-bus--pubsub--websocket-broadcast)
14. [Governance — Marco de Control y Autonomía](#14-governance--marco-de-control-y-autonomía)
15. [Training — Sistema de Entrenamiento Progresivo](#15-training--sistema-de-entrenamiento-progresivo)
16. [Skills — Registro de Habilidades](#16-skills--registro-de-habilidades)
17. [Persistencia — Postgres + ChromaDB + SQLite](#17-persistencia--postgres--chromadb--sqlite)
18. [Dashboard — Interfaz de Control (Next.js)](#18-dashboard--interfaz-de-control-nextjs)
19. [Flujo Completo End-to-End](#19-flujo-completo-end-to-end)
20. [Garantías Arquitecturales (Phase 3)](#20-garantías-arquitecturales-phase-3)
21. [Identity Core — Módulo de Identidad Formal (Phase 4)](#21-identity-core--módulo-de-identidad-formal-phase-4)
22. [Qué Sobrevive un Reinicio](#22-qué-sobrevive-un-reinicio)
23. [Observaciones y Problemas Conocidos](#23-observaciones-y-problemas-conocidos)
24. [Estado Actual vs Planificado](#24-estado-actual-vs-planificado)
25. [Estructura de Archivos Completa](#25-estructura-de-archivos-completa)

---

## 1. VISIÓN GENERAL DEL SISTEMA

Django es un **delegado digital autónomo**: un sistema de IA multi-agente que aprende progresivamente a actuar como su "principal" (el humano que lo entrena). No es un chatbot genérico — es una **identidad virtual personalizada** que:

- **Piensa** con la misma estructura de decisión que su principal (valores, prioridades, tolerancia al riesgo).
- **Habla** con el mismo estilo de comunicación (formalidad, humor, empatía, verbosidad calibrada).
- **Decide** respetando los mismos límites (boundaries hardcodeados, análisis costo-beneficio, prioridades de stakeholders).
- **Aprende** de correcciones directas del humano, acumulando reglas conductuales en memoria procedimental.
- **Se evalúa a sí mismo** con 5 módulos heurísticos que miden calidad, alineación con la persona, riesgo legal, decisiones de negocio y operaciones de memoria — todo sin llamadas LLM adicionales.

**Filosofía de diseño**: El sistema opera en **$0/mes** usando exclusivamente tiers gratuitos (Gemini Free, Groq Free, Neon Free, ChromaDB local, Ollama local). La privacidad es prioridad: datos de identidad nunca salen de la máquina local; solo los outputs de agentes van a LLMs cloud.

**Estado actual**: Phase 10D.1 completada (Governance Hardening). El módulo de identidad formal (`src/identity/`, 22 archivos, ~6080 líneas) provee un `IdentityProfile` versionado, con baseline embedding de 384 dimensiones (all-MiniLM-L6-v2), inyectado en DecisionEngine y AlignmentEvaluator para drift detection basado en similitud coseno. Phase 5A agrega `IdentityEnforcer` como señal de observabilidad activa. Phase 5B agrega `IdentityPolicyEngine` como capa reactiva configurable (none/log/flag/rewrite_request/block). Phase 5C agrega `IdentityFeedbackController` como capa de feedback controlado. Phase 6A agrega `IdentityMemoryBridge` para análisis de afinidad memoria–identidad. Phase 6B agrega `IdentityContextWeighter` para anotación soft de contexto. Phase 6C agrega `IdentityDecisionModulator` para evaluación de alignment decisión–identidad. Phase 6D agrega `IdentityConfidenceEngine` para agregación de señales de identidad en confidence score. Phase 7A agrega `IdentityAutonomyModulator` para ajuste de governance threshold basado en confianza. Phase 7B agrega `IdentityRetrievalWeighter` para re-ranking de memoria ponderado por identidad. Phase 7C agrega `IdentityConsolidationWeighter` para ajuste de importancia de memoria pre-storage. Phase 8A agrega `IdentityBehavioralBias` para bias de planificación guiado por identidad. Phase 8B agrega `IdentityPromptIntegrator` para inyección de preferencias de estilo en system prompt. Phase 9A agrega `IdentityHealthMonitor` para monitoreo longitudinal de salud de identidad. Phase 9B agrega `IdentityHealthRegulator` para regulación adaptativa basada en salud. Phase 10A agrega `IdentityEvolutionEngine` para análisis y propuesta de evolución de identidad (proposal-only, requires_human_approval). Phase 10B agrega `IdentityShadowSimulator` para simulación de evolución en clon in-memory sin mutar la identidad real. Phase 10C agrega `IdentityVersionControl` para versionado inmutable de identidad con apply controlado y rollback seguro (710 líneas, 120 unit tests). Phase 10D agrega interfaz de governance de identidad (dashboard `/identity-governance`, 11 endpoints, 4 tabs: Versions/Evolution/Shadow/Health, 55 unit tests). Phase 10D.1 aplica governance hardening (5 defectos D1–D5: reject event fix con EventBus singleton, approve audit event, activate symmetry/idempotency, delete active guard — 20 unit tests).

---

## 2. DIAGRAMA DE ALTO NIVEL

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                 DASHBOARD (Next.js 15 / React 19 / Tailwind / shadcn/ui)     │
│                 Puerto 3000 — 13 rutas + WebSocket client                    │
├──────────────────────────┬───────────────────────────────────────────────────┤
│  Zustand Store (165 ln)  │  API Client — lib/api.ts (1117 ln, ~96 métodos)  │
│  Estado global del UI    │  Comunicación tipada con el backend               │
├──────────────────────────┴───────────────────────────────────────────────────┤
│                                     ▼ HTTP / WebSocket                       │
├──────────────────────────────────────────────────────────────────────────────┤
│         FastAPI Backend — Puerto 8000 — Prefijo /api — 100 endpoints          │
│                           routes.py (3264 ln) + main.py (324 ln)             │
├──────────┬───────────┬──────────┬───────────┬──────────┬─────────────────────┤
│ COGNICIÓN│ ORQUESTA- │ AGENT    │ MEMORIA   │ EVALUA-  │ TRACE              │
│ Decision │ DOR       │ CREW (5) │ 4 niveles │ CIÓN     │ Collector          │
│ Engine   │ Pipeline  │          │ Manager   │ 5 módulos│ 13 tipos nodo      │
│ (138 ln) │ 10+ pasos │ Identity │ (~810 ln) │ heuríst. │ (342 ln)           │
│ Planner  │ (2135 ln) │ Business │           │ (1796 ln │                    │
│ (118 ln) │ + 3 Modos │ Comms    │           │  total)  │                    │
│ Categ.   │           │ Tech     │           │          │                    │
│ (18 ln)  │           │ Govern.  │           │          │                    │
├──────────┴───────────┴──────────┴───────────┴──────────┴─────────────────────┤
│ Model Router (360 ln) │ Skill Registry (116 ln) │ Training Mgr (434 ln)      │
│ Gemini → Groq → Ollama│ 4 skills + LearnTopic  │ 3 modos + correcciones     │
│ Fallback automático   │ (106 ln) + Tools(110 ln)│ + upload writing samples   │
├───────────────────────┴─────────────────────────┴────────────────────────────┤
│ Event Bus (128 ln)     │ Service Logger (218 ln)  │ Governance (stub)         │
│ Pub/Sub + WS + Audit   │ Logs rotativos + JSONL   │ Config en YAML            │
├────────────────────────┴──────────────────────────┴──────────────────────────┤
│           ALMACENAMIENTO                                                      │
│  ChromaDB (local)          │  SQLite (local)         │  Neon Postgres (cloud) │
│  Episodic + Semantic       │  Procedural Memory      │  10 tablas             │
│  Vectores + cosine search  │  Correcciones + workflows│  Audit + Persistence  │
└────────────────────────────┴─────────────────────────┴────────────────────────┘
```

---

## 3. STACK TECNOLÓGICO

| Capa | Tecnología | Versión | Propósito | Notas |
|------|-----------|---------|-----------|-------|
| **Backend** | Python + FastAPI + uvicorn | 3.11+ | API REST + WebSocket, orquestación multi-agente | Puerto 8000, `--reload` en dev |
| **Frontend** | Next.js + React + TypeScript | 15 / 19 / 5.7 | Dashboard de control con 13 rutas | Puerto 3000, App Router |
| **UI** | shadcn/ui + Tailwind CSS + lucide-react | 3.4 | Componentes primitivos + tema lab oscuro | Variables CSS: lab-text, lab-card, accent-glow |
| **Estado UI** | Zustand | 5 | Store global del cliente | 165 líneas |
| **LLM primario** | Google Gemini 2.5 Flash | — | Proveedor principal (free tier) | Via google-generativeai SDK |
| **LLM secundario** | Groq (Llama 3.3 70B) | — | Fallback #1 (free tier) | Via groq SDK |
| **LLM local** | Ollama (llama3.1:8b) | — | Fallback #2 + modo privacidad | Siempre disponible localmente |
| **Vector DB** | ChromaDB | local | Memoria episódica + semántica | Embeddings `all-MiniLM-L6-v2` (default) |
| **SQL relacional** | Neon Postgres | remoto | Audit log, mensajes, persistencia Phase 1 | psycopg2 síncrono, autocommit |
| **SQL local** | SQLite | local | Memoria procedimental (correcciones, workflows) | Archivo `procedural.db` |
| **Visualización** | recharts | — | Gráficos del dashboard de analytics | |
| **Flow visualization** | @xyflow/react (React Flow v12) | — | Visualización de cognitive traces | Nodos personalizados |
| **Web research** | Tavily API | — | Búsquedas web para skill de investigación | Free tier |
| **Auth** | Supabase Auth | planificado | JWT + login/logout | No conectado aún — API abierta |
| **Storage** | Cloudflare R2 | planificado | S3-compatible para archivos | No conectado aún |

**Costo operacional: $0/mes** — todos los servicios en free tier.

---

## 4. CAPA DE CONFIGURACIÓN

### 4.1 Settings — `agent/src/config.py` (98 líneas)

Singleton Pydantic `BaseSettings` que carga desde `.env`. Define todas las variables de entorno con valores por defecto seguros:

```python
class Settings(BaseSettings):
    # LLM Providers
    gemini_api_key: Optional[str]     # Si None → se desactiva Gemini
    groq_api_key: Optional[str]       # Si None → se desactiva Groq
    ollama_base_url: str = "http://localhost:11434"

    # Database (opcional)
    database_url: Optional[str]       # Neon Postgres connection string

    # Feature flags (properties calculados)
    @property
    def has_gemini(self) -> bool      # Chequea que la key no sea None ni placeholder
    @property
    def has_database(self) -> bool    # Chequea que database_url exista
    @property
    def has_tavily(self) -> bool      # Chequea tavily_api_key
```

**Diseño clave**: Cada servicio externo tiene un flag booleano. El sistema degrada gracefully cuando un servicio no está disponible — nunca crashea por falta de configuración.

### 4.2 Archivos de Configuración YAML/JSON

| Archivo | Líneas | Propósito | Formato |
|---------|--------|-----------|---------|
| `configs/persona.yaml` | 87 | **Identidad completa del principal**: Big Five traits (5 dimensiones, 0.0-1.0), valores ordenados por prioridad (7 valores), estilo de comunicación (6 parámetros calibrados), decision_making (risk_tolerance, analysis_depth, stakeholder_weight, data_vs_intuition, trade_off_priorities), boundaries (6 reglas conductuales hardcodeadas), expertise (primary/secondary), preferencias de trabajo, writing_style | YAML |
| `configs/models.json` | ~172 | 3 proveedores LLM con modelos específicos, cadena de fallback (gemini → groq → ollama), asignaciones por agente, 6 task-type routings, 4 perfiles (balanced, max_quality, privacy_mode, budget_mode), version 2.0 | JSON |
| `configs/skills.json` | ~55 | Registro de 4 skills (web-research, email-draft, document-gen, learn-topic) con toggle enable/disable, nivel de riesgo por skill, estadísticas de uso | JSON |
| `configs/governance.yaml` | 254 | **Framework de gobernanza completo**: 5 niveles de autonomía (Observer→Trusted), clasificación de riesgo por acción (low/medium/high/critical con ejemplos), operaciones prohibidas, reglas de privacidad, escalamiento, controles de emergencia | YAML |

**Nota para auditores**: `persona.yaml` es el archivo fundamental de identidad. Contiene la "personalidad transferible" del principal — cada campo afecta directamente cómo los agentes generan respuestas. El campo `boundaries` contiene reglas que **nunca** pueden ser violadas por ningún agente, sin importar el contexto.

---

## 5. CAPA DE ENTRADA — API REST + WebSocket

### 5.1 Composition Root — `agent/src/api/main.py` (324 líneas)

Punto de entrada de la aplicación. El `AppState` global singleton mantiene todos los componentes inicializados:

```python
class AppState:
    model_router: ModelRouter          # Router LLM con fallback chain
    identity_agent: IdentityCoreAgent  # Agente principal (backward compat)
    agent_crew: AgentCrew              # Crew de 5 agentes
    orchestrator: Orchestrator         # Pipeline de 10 pasos
    decision_engine: DecisionEngine    # Motor cognitivo determinístico (inmutable)
    planner: Planner                   # Planificador de ejecución (stateless)
    memory_manager: MemoryManager      # Sistema de memoria 4 niveles
    skill_registry: SkillRegistry      # Registro de habilidades
    training_manager: TrainingManager  # Gestor de entrenamiento
    database: Database                 # Neon Postgres (opcional)
    persistence: PersistenceRepository # Repositorio Phase 1 (opcional)
    settings: Settings                 # Configuración global
```

**Orden de inicialización** (secuencial, determinístico):
1. `Settings` — cargar `.env`
2. `Database` + `PersistenceRepository` — Postgres (si `has_database`)
3. `ModelRouter` — inicializar proveedores LLM + wiring callback de token persistence
4. `AgentCrew` — 5 agentes con persona.yaml + governance.yaml
5. `MemoryManager` — 4 tiers (ChromaDB + SQLite)
6. `SkillRegistry` — cargar skills.json
7. `TrainingManager` — cargar correcciones desde ProceduralMemory
8. `DecisionEngine(autonomy_level=0)` + `Planner()` — capa cognitiva
9. `Orchestrator` — recibe crew, engine, planner, memory, skills, training (validación `isinstance`)
10. Wiring de callback de memory rollback persistence
11. Emisión de evento `system.startup` vía EventBus

**Middleware stack**:
- `ServiceMonitorMiddleware` — logea requests >5s como slow, registra errores 500
- `CORSMiddleware` — permite `localhost:3000` (dashboard)

### 5.2 API Endpoints — `agent/src/api/routes.py` (3264 líneas, 100 endpoints)

Todas las rutas bajo prefijo `/api`. Patrón uniforme:

```python
@router.get("/endpoint")
async def handler():
    from src.api.main import get_state  # Lazy import para evitar circular
    state = get_state()
    result = state.component.method()
    return {"status": "ok", **result}
```

**Catálogo completo de endpoints por grupo funcional:**

| Grupo | Endpoints | Métodos | Descripción |
|-------|-----------|---------|-------------|
| **Health & System** | `/health`, `/router/status`, `/events/recent`, `/system-status` | GET×4 | Estado del sistema, proveedores disponibles, eventos recientes, diagnóstico completo |
| **Service Log** | `/service-log/events`, `/service-log/report` | GET×2 | Logs de servicio estructurados |
| **Chat** | `/chat`, `/ws` | POST, WebSocket | Mensaje de chat principal (orquestado), WebSocket bidireccional |
| **Config Reload** | `/router/reload`, `/persona/reload` | POST×2 | Hot-reload de models.json y persona.yaml sin reinicio |
| **Persona** | `/persona/info`, `/persona/traits`, `/persona/values`, `/persona/communication`, `/persona/boundaries` | GET, PUT×4 | Lectura y escritura de todos los aspectos de la personalidad |
| **Crew** | `/crew/agents` | GET | Lista de agentes con roles y descripciones |
| **Trace** | `/trace/list`, `/trace/{id}`, `/trace/latest/graph`, `/trace/replay`, `/trace/{id}` (DELETE), `/trace` (DELETE) | GET×3, POST, DELETE×2 | Trazas cognitivas, grafos React Flow, replay paso a paso, eliminación individual y masiva |
| **Persistence** | `/interactions`, `/interactions/{id}/trace`, `/interactions/{id}/evaluations`, `/token-usage/persisted` | GET×4 | Datos persistidos en Postgres |
| **Memory** | `/memory/stats`, `/memory/search`, `/memory/semantic/store`, `/memory/semantic/{id}` (PUT, DELETE), `/memory/episodic/{id}` (DELETE), `/memory/bulk-delete`, `/memory/working/clear` | GET, POST×4, PUT, DELETE×2 | CRUD completo del sistema de memoria + eliminación masiva + limpieza de working memory por conversación |
| **Skills** | `/skills`, `/skills/{id}/toggle`, `/skills/web-research`, `/skills/learn-topic` | GET, POST×3 | Gestión de habilidades + herramienta de aprendizaje + learn-topic pipeline |
| **Training** | `/training/status`, `/training/session/start`, `/training/session/end`, `/training/correction`, `/training/history`, `/training/upload-samples`, `/training/exchange`, `/training/interview/questions`, `/training/interview/answer`, `/training/corrections`, `/training/corrections/{id}`, `/training/suggestions` | GET×3, POST×7, PUT, DELETE×2 | Sesiones de entrenamiento, correcciones CRUD, upload de muestras de escritura, free conversation exchanges, guided interview Q&A |
| **Models** | `/models/config`, `/models/assignment`, `/models/profile`, `/models/test` | GET, PUT×2, POST | Gestión de proveedores LLM, perfiles, asignaciones por agente |
| **Evaluation** | `/evaluation/overview`, quality/×3, alignment/×2, legal/×3, decisions/×5, rollback/×5, `/evaluation/data` (DELETE) | GET×14, POST×3, PUT×1, DELETE | 20 endpoints para los 5 módulos de evaluación + eliminación masiva |
| **Governance** | `/governance/config`, `/governance/audit-log`, `/governance/approvals`, `/governance/emergency-stop`, `/governance/emergency-resume`, `/governance/emergency-status` | GET×3, POST×2, GET×1 | Configuración, auditoría, aprobaciones, parada de emergencia |
| **Analytics** | `/analytics/overview`, `/analytics/identity-fidelity`, `/analytics/autonomy`, `/analytics/token-usage`, `/analytics/events` (DELETE), `/analytics/tokens` (DELETE) | GET×4, DELETE×2 | KPIs, fidelidad de identidad, métricas de autonomía, uso de tokens, eliminación de datos |
| **Identity Governance** | `/identity/versions`, `/identity/versions/{id}` (GET+DELETE), `/identity/snapshot`, `/identity/activate/{id}`, `/identity/rollback/{id}`, `/identity/evolution`, `/identity/evolution/{id}/approve`, `/identity/evolution/{id}/reject`, `/identity/shadow`, `/identity/health` | GET×5, POST×4, DELETE×1 | 11 endpoints. Versiones de identidad (list, get, delete con guard contra activa), snapshots con metadata (version, tags, label, notes), activación idempotente, rollback, evolución (approve/reject con audit events), shadow simulation, health signals. Phase 10D + 10D.1 hardening (D1–D5) |

### 5.3 WebSocket — `/api/ws`

Conexión bidireccional que sirve para:
1. **Broadcast de eventos** — el EventBus envía todos los eventos del sistema (orchestrator, decision, agent_state, chat, system) a todos los WebSocket conectados.
2. **Chat directo** — el WebSocket puede recibir mensajes de chat y responder directamente a través de `identity_core` (bypass del orchestrator — solo para conversación simple).

---

## 6. CAPA COGNITIVA — DecisionEngine + Planner

> **MÓDULO OBLIGATORIO** — El Orchestrator no puede inicializarse sin la capa cognitiva. La validación es por `isinstance()` en el constructor, y lanza `TypeError` si falta cualquiera de los dos componentes.

La capa cognitiva es un módulo **puro determinístico** — cero llamadas LLM, cero IO, cero acceso a repositorios. Su propósito es extraer la lógica de decisión que antes estaba implícita y dispersa en el orchestrator, haciéndola:
- **Explícita** — cada decisión produce un `DecisionResult` estructurado y trazable
- **Testeable** — 49 tests puros sin mocks en `tests/test_cognition.py`
- **Inmutable** — el DecisionEngine está congelado después de la construcción

### 6.1 TaskCategory — `agent/src/flows/categories.py` (18 líneas)

Enum compartido que define las 6 categorías de tarea posibles. Vive en `flows/` (no en `cognition/`) para romper la dependencia circular:

```python
class TaskCategory(str, Enum):
    CONVERSATION = "conversation"   # Chat general → IdentityCore responde directamente
    BUSINESS = "business"           # Estrategia, deals, pricing → BusinessAgent
    COMMUNICATION = "communication" # Email, mensajes, propuestas → CommunicationAgent
    TECHNICAL = "technical"         # Código, arquitectura, bugs → TechnicalAgent
    RESEARCH = "research"           # Investigación web → TechnicalAgent + WebResearch
    MULTI_AGENT = "multi_agent"     # Tareas complejas que requieren colaboración (stub)
```

**Grafo de dependencias**: `categories.py` ← `{decision_engine.py, orchestrator.py}`. No hay dependencia inversa de cognition → orchestrator.

### 6.2 DecisionEngine — `agent/src/cognition/decision_engine.py` (138 líneas)

Motor de decisión determinístico e **inmutable**. Evalúa cómo manejar cada tarea clasificada.

**Inmutabilidad garantizada por diseño**:
```python
class DecisionEngine:
    __slots__ = ("_autonomy_level", "_identity_profile")  # Solo dos atributos permitidos
    def __init__(self, autonomy_level: int = 0, identity_profile=None):
        object.__setattr__(self, "_autonomy_level", autonomy_level)  # Bypass del override
        object.__setattr__(self, "_identity_profile", identity_profile)
    def __setattr__(self, _name, _value):
        raise AttributeError("DecisionEngine is immutable")  # Bloquea toda mutación
    @property
    def autonomy_level(self) -> int: return self._autonomy_level  # Solo lectura
    @property
    def identity_profile(self): return self._identity_profile     # Solo lectura (Phase 4)
```

**Tablas de decisión (mapeo determinístico)**:

| TaskCategory | Strategy | Agent Seleccionado | Riesgo Preliminar | Identity Review | Governance Review |
|---|---|---|---|---|---|
| CONVERSATION | DIRECT_RESPONSE | identity_core | low | No | No |
| BUSINESS | STRUCTURED_ANALYSIS | business | medium | Sí | Sí |
| COMMUNICATION | STRUCTURED_ANALYSIS | communication | low | Sí | Sí |
| TECHNICAL | STRUCTURED_ANALYSIS | technical | low | Sí | Sí |
| RESEARCH | RESEARCH_REQUIRED | technical | low | Sí | Sí |
| MULTI_AGENT | MULTI_AGENT | identity_core | medium | Sí | Sí |

**Regla de gates**: Toda categoría que NO sea `CONVERSATION` pasa por identity review y governance review. Esto garantiza que cualquier output especializado sea revisado por el guardián de personalidad y el agente de cumplimiento.

**Output**: `DecisionResult` (frozen dataclass) con: category, strategy, selected_agent, autonomy_level, preliminary_risk, requires_identity_review, requires_governance_review, reasoning.

### 6.3 Planner — `agent/src/cognition/planner.py` (118 líneas)

Planificador de ejecución **stateless**. Convierte un `DecisionResult` en un `Plan` con pasos ordenados.

**Diseño stateless**: El constructor no acepta argumentos. `governance_enabled` se pasa como parámetro a `build()` desde el Orchestrator, que es el **single source of truth** para esa configuración.

```python
class Planner:
    def __init__(self): ...  # Sin argumentos — stateless
    def build(self, decision: DecisionResult, governance_enabled: bool = True, skip_governance: bool = False) -> Plan:
```

**Pasos generados por el Planner**:
1. `route` — **siempre presente** — enviar al agente seleccionado para generación LLM
2. `identity_review` — **condicional** — solo si `decision.requires_identity_review == True`
3. `governance_review` — **condicional** — solo si `decision.requires_governance_review AND governance_enabled AND NOT skip_governance`

**Output**: `Plan` (frozen dataclass) con: strategy, steps (List[PlanStep]), requires_multi_step, selected_agent, decision.

### 6.4 Tests de Cognición — `agent/tests/test_cognition.py` (235 líneas)

49 tests unitarios puros, sin mocks, sin IO:
- `TestDecisionEngine` (28 tests): strategy mapping, agent selection, risk assessment, review gates, autonomy level, inmutabilidad, frozen result, completitud de output
- `TestPlanner` (17 tests): identity review gate, governance review gate, estructura del plan, orden determinístico, metadatos del plan, frozen assertions
- `TestOrchestratorCognitionRequired` (3 tests): rechaza engine None, rechaza planner None, rechaza tipos incorrectos

---

## 7. CAPA DE ORQUESTACIÓN — Pipeline de 10 Pasos

### `agent/src/flows/orchestrator.py` (2135 líneas)

El Orchestrator es el **cerebro central** del sistema. Recibe un mensaje de usuario y lo procesa a través de un pipeline determinístico de 10+ pasos, donde cada paso emite eventos vía EventBus y crea nodos de trace vía TraceCollector.

**Constructor con validación obligatoria**:
```python
class Orchestrator:
    def __init__(self, crew, decision_engine: DecisionEngine, planner: Planner, ...):
        if not isinstance(decision_engine, DecisionEngine):
            raise TypeError("Orchestrator requires a DecisionEngine instance")
        if not isinstance(planner, Planner):
            raise TypeError("Orchestrator requires a Planner instance")
```

### Pipeline detallado:

| Paso | Nombre | Qué hace | LLM | Latencia típica |
|------|--------|----------|-----|----------------|
| 0 | **Emergency Check** | Si `_emergency_stopped == True`, bloquea todo. Retorna mensaje de parada. | No | <1ms |
| 1 | **Classify** | Clasifica el mensaje por keywords heurísticos. Requiere ≥2 matches para overridear CONVERSATION. 24 keywords por categoría. | No | <1ms |
| 1a | **Learning Detection** | Si el mensaje matchea patrones de aprendizaje ("aprende sobre X", "hazte experto en Y"), dispara el pipeline learn-topic: web search → LLM summarize → chunk → store en ChromaDB. Si la skill está habilitada y el match es positivo, la respuesta se genera con el contexto de lo aprendido y se retorna directamente (bypass pasos 2-10). | Sí | 10-60s |
| 2 | **Decision Engine** | `DecisionEngine.evaluate()` — determina strategy, agent, risk, review gates. Resultado: `DecisionResult` (frozen). | No | <1ms |
| 3c | **Identity Decision Modulation** | Evalúa alignment entre `DecisionResult` e `IdentityProfile` en 4 factores: risk tolerance, category–values, autonomía, decision style. Composite score → label (aligned/tension/misaligned). Estrictamente observacional — nunca modifica decisiones. Phase 6C. | No | <1ms |
| 3d | **Identity Confidence** | Agrega señales de identidad (enforcement similarity, policy severity, decision alignment, memory affinity) en confidence score ponderado (0.0–1.0) + autonomy_modifier (+1/0/-1). Primer componente no-observacional — metadata advisory. Degradación graceful con inputs faltantes. Phase 6D. | No | <1ms |
| 3e | **Autonomy Modulation** | Ajusta governance threshold basado en confidence level: low → -0.10 (más estricto), medium → 0.0, high → +0.05 (relajado). Threshold clamped a [0.0, 1.0]. Solo emite evento `identity.autonomy_adjusted` cuando hay ajuste. Metadata advisory inyectada en governance_review trace. Phase 7A. | No | <1ms |
| 3f | **Behavioral Bias** | Deriva soft planning bias desde señales de identidad: `recommended_planner_mode` (conservative/deep/none) + `style_bias` (tone_weight, assertiveness, depth_bias, creativity_bias). Basado en confidence, alignment, communication_style, values, decision_making. No modifica DecisionEngine ni governance. Advisory-only. Emite `identity.behavioral_bias_applied`. Phase 8A. | No | <1ms |
| 3g | **Prompt Integration** | Renderiza metadata de Phase 8A (`style_bias` + `recommended_planner_mode`) como bloque de texto determinístico `[IDENTITY STYLE PREFERENCES]` y lo inyecta (prepend) en el system prompt antes de las instrucciones principales. Guards: None/not dict/observational/bias_not_applied/no style_bias → skip. Emite `identity.prompt_injected`. Phase 8B. | No | <1ms |
| 6d | **Memory Consolidation Weighting** | Ajusta importancia de memoria pre-storage usando identity confidence (Phase 6D) y decision alignment (Phase 6C). Factor = 1.0 ± 0.10 (confidence) ± 0.05 (alignment), clamped [0.75, 1.25]. Non-blocking, no previene storage, no elimina ni muta contenido. Emite `identity.memory_consolidation_adjusted`. Phase 7C. | No | <1ms |
| 3 | **Planner** | `Planner.build()` — construye Plan con pasos ordenados. Respeta `governance_enabled` del Orchestrator. | No | <1ms |
| 4 | **Memory Recall + Mode Filter** | Consulta working memory **per-conversation** (RAM, aislado por `conversation_id`) + episodic (ChromaDB cosine search) + semantic (ChromaDB cosine search). Máximo 3 resultados por tier. Budget de ~2000 tokens. **Si cognitive_mode ≥ 2**: filtra `memories["semantic"]` para solo incluir chunks con `category=="learned_knowledge"`. | No | 50-200ms |
| 2b | **Identity Memory Bridge** | Analiza afinidad entre cada memoria recuperada y baseline embedding del principal. Cosine similarity por memoria + agregados avg/max/min. Estrictamente observacional (no modifica, filtra ni re-ordena). Phase 6A. | No | <5ms |
| 2c | **Identity-Weighted Retrieval** | Re-rankea memorias usando `weighted_score = semantic_similarity * 0.8 + identity_affinity * 0.2`. Sort estable descendente. Si affinity no disponible, preserva orden original. Estrictamente non-destructive (mismos items, sin filtrado). Phase 7B. | No | <1ms |
| 5 | **Correction Injection** | Extrae correcciones conductuales de ProceduralMemory (SQLite) para el agente target. Se inyectan como reglas en el prompt. | No | <5ms |
| 3b | **Identity Context Weighting** | Anota cada línea de memoria en el contexto con `[IDENTITY_ALIGNED]` (similarity ≥ avg) o `[LOW_IDENTITY_ALIGNMENT]` (similarity < avg) basado en scores de afinidad de Phase 6A. Matching por `memory_id` via `context_line_ids` (refactor Phase 6C). Adjunta bloque "Identity Context Analysis" con conteos. Estrictamente soft: no elimina, no reordena, no modifica contenido. | No | <1ms |
| 6 | **Prompt Build + Mode Logic** | Fusiona: memory context + correction context + extra context + conversation history. **Modo 3**: retorno directo de memoria sin LLM — ensambla líneas `[tier/category] text` de memorias recuperadas. **Modo 2**: inyecta Knowledge Status Header (con/sin datos aprendidos). **Modo 1**: sin restricciones. | No | <1ms |
| 7 | **LLM Generate** | Ruta al agente asignado → `ModelRouter.generate()` → proveedor LLM en la cadena de fallback. | Sí | 500-5000ms |
| 8 | **Identity Review** | **Per plan**: Si el Plan incluye paso `identity_review`, el IdentityCoreAgent revisa el output para alineación con la personalidad del principal. Puede reescribir la respuesta. | Sí | 500-3000ms |
| 9 | **Governance Review** | **Per plan**: Si el Plan incluye paso `governance_review`, el GovernanceAgent revisa el output. Produce JSON con `approved`, `risk_level`, `flags`, `revised_content`. Auto-approves en parse errors. | Sí | 500-3000ms |
| 10 | **Memory Store + Evaluation + Persist** | (a) Almacena en working + episodic memory. (b) Ejecuta 5 módulos de evaluación heurística (sin LLM). (c) Persiste a Postgres (fire-and-forget): interaction, trace_nodes, evaluations, token_usage. | No | 10-100ms |
| 9a | **Identity Health Monitor** | Post-persistence, fuera del pipeline principal. Agrega señales de identidad longitudinales (últimas 50 interacciones): avg_similarity, avg_confidence, drift_rate, high_severity_policy_rate, sustained_low_confidence, instability_index (0-1). Clasifica: stable (<0.25) / monitor (<0.50) / unstable (<0.70) / critical (≥0.70). Emite `identity.health_evaluated`. Estrictamente observacional — no afecta la interacción actual. Phase 9A. | No | 5-50ms |
| 9b | **Identity Health Regulation** | Post-health monitor, antes del return. Capa de meta-control adaptativa que reacciona a señales de salud de Phase 9A. Ajusta governance threshold (stable: 0, monitor: -0.05, unstable: -0.10, critical: -0.15) e identity weight (stable: 0, monitor: +0.05, unstable: +0.10, critical: +0.15). Clamp threshold [0.0, 1.0], identity_weight [0.0, 0.5]. Emite `identity.health_regulated` (solo cuando regulation_applied). Determinístico, stateless, metadata-only — nunca modifica identidad, decisiones, routing, LLM outputs ni interacción actual. Phase 9B. | No | 1-5ms |
| 9c | **Identity Evolution Analysis** | Post-health regulation, antes del return. Analiza trayectoria de identidad a largo plazo (últimas 200 interacciones): similarity_trend (regresión lineal), confidence_trend, sustained_high_confidence (últimas 10 > 0.75), sustained_similarity_shift (últimas 20 difieren > 0.08), drift_rate, avg_instability, high_severity_rate. Criterios de evolución: sustained_high_conf AND (trend > 0 OR sustained_shift) AND drift_rate < 0.25 AND avg_confidence > 0.70. Rechazo: instability > 0.60 OR high_severity > 0.20. Si candidato: computa centroid embedding (últimas 30 respuestas), calcula shift_magnitude = 1 - cosine_similarity(baseline, centroid), versión bump (minor 0.05-0.10, major > 0.10). Emite `identity.evolution_analyzed`. Persistido como `identity_evolution_analysis`. Proposal-only — `requires_human_approval: True` siempre, `observational: True`, nunca modifica IdentityProfile, baseline, decisiones, routing ni governance. Phase 10A. | No | 5-50ms |
| 9d | **Shadow Identity Simulation** | Post-evolution analysis, antes de persistencia final. Si hay evolution_candidate, construye clon in-memory (deep copy) del IdentityProfile y aplica cambios propuestos solo al shadow. Computa señales comparativas: hypothetical_similarity_shift, confidence_shift, drift_rate_change, instability_delta. Risk score ponderado (instability 0.30 + drift 0.25 + similarity 0.25 + confidence 0.20), clamped [0,1]. Risk grade: safe (<0.25) / cautious (<0.50) / risky (<0.70) / destabilizing (≥0.70). Produce structural diff (version, embedding, big_five, values, communication_style, boundaries). Emite `identity.shadow_simulated`. Persistido como `identity_shadow_simulation`. NUNCA muta el IdentityProfile real. `observational: True`, `requires_human_approval: True` siempre. Stateless, determinístico, sin LLM, sin IO, sin DB writes. Phase 10B. | No | 1-10ms |
| 9e | **Identity Version Candidate** | Post-shadow simulation, antes de persistencia. Si evolution_candidate == True AND shadow risk_grade in (safe, cautious) AND identity activa presente: crea snapshot inmutable del IdentityProfile actual con `IdentityVersionControl.build_version_candidate()`. Snapshot incluye: UUID version_id, ISO timestamp, SHA-256 content_hash, evolución metadata (proposed_version, shift_magnitude, shadow_risk_grade/score, evolution_risk_level). Resultado compacto via `build_result()` (strip snapshot/profile_data) almacenado en `evaluation_results["identity_version_candidate"]`. Emite `identity.version_candidate_created`. Persistido como `identity_version_candidate`. NUNCA auto-aplica cambios, NUNCA muta IdentityProfile, NUNCA modifica runtime. `observational: True`, `requires_human_approval: True` siempre. Proposal-only — el principal debe aprobar y ejecutar apply/rollback manualmente. Phase 10C. | No | <1ms |

**Routing por categoría** (`_route()` method):
```python
CONVERSATION → identity_core.respond(message, history, context)
BUSINESS     → business.generate(prompt, system_prompt)
COMMUNICATION→ communication.generate(prompt, system_prompt)
TECHNICAL    → technical.generate(prompt, system_prompt)
RESEARCH     → technical.generate(prompt, system_prompt)  # + web research skill si disponible
*default*    → identity_core.respond(message, history, context)
```

**Parada de emergencia**: `emergency_stop()` / `emergency_resume()` controlan un flag `_emergency_stopped` que bloquea todo procesamiento en el paso 0.

**Lo que NO contiene el Orchestrator** (eliminado en Phase 3):
- `_category_to_role()` — la selección de agente vive exclusivamente en DecisionEngine
- `if self.decision_engine is None` — no hay ruta legacy sin cognición
- Ternarios `plan else category !=` — no hay fallback sin Plan
- Variable `decision` reutilizada para evaluación — renombrada a `biz_decision` para evitar shadowing

---

## 8. CAPA DE AGENTES — Crew de 5 Agentes Especializados

### 8.1 Arquitectura de Agentes

```
                     ┌─────────────────────────────┐
                     │  AgentCrew (111 ln)          │
                     │  Inicializa y gestiona todos │
                     │  Método: reload_persona()    │
                     └────────┬────────────────────┘
                              │
           ┌──────────────────┼──────────────────────┐
           │                  │                      │
    ┌──────▼──────┐    ┌──────▼──────┐        ┌──────▼──────┐
    │  Identity   │    │  BaseAgent  │        │  BaseAgent  │
    │  CoreAgent  │    │  (ABC)      │        │  (ABC)      │
    │  (255 ln)   │    │  (91 ln)    │        │  (91 ln)    │
    │  STANDALONE │    └──────┬──────┘        └──────┬──────┘
    │  No extiende│         │                       │
    │  BaseAgent  │    ┌────┼────┬─────┐           │
    └─────────────┘    │    │    │     │           │
                 Business Comm Tech Governance
                 (50 ln) (56) (44)  (132 ln)
```

**Nota importante**: `IdentityCoreAgent` **NO extiende** `BaseAgent` — tiene su propia implementación con `respond()` y carga de persona directa desde YAML. Los otros 4 agentes sí extienden `BaseAgent`.

### 8.2 Detalle de cada Agente

| Agente | Clase | Archivo | Rol | System Prompt | Modelo Default | Temperatura |
|--------|-------|---------|-----|---------------|----------------|-------------|
| **Identity Core** | `IdentityCoreAgent` | `identity_core.py` (255 ln) | **Guardián de la personalidad**. Responde COMO el principal. Construye system prompt dinámico desde persona.yaml con Big Five traits mapeados a descripciones textuales + valores + estilo de comunicación + boundaries + writing style. Soporta Knowledge Boundary condicional (Modo 2). | Dinámico (~1500 tokens) con persona completa + Knowledge Boundary (si cognitive_mode==2) | Gemini | 0.7 |
| **Business** | `BusinessAgent` | `business_agent.py` (50 ln) | Estratega de negocio. Analiza deals, pricing, ROI, stakeholders. | `_persona_header()` + expertise + values + boundaries + contexto de negocio | Groq | 0.7 |
| **Communication** | `CommunicationAgent` | `communication_agent.py` (56 ln) | Especialista en comunicación. Redacta emails, propuestas, mensajes en el estilo del principal. | `_persona_header()` + communication style + writing style + values | Gemini | 0.7 |
| **Technical** | `TechnicalAgent` | `technical_agent.py` (44 ln) | Constructor técnico. Código, arquitectura, debugging. | `_persona_header()` + tech expertise + quality standards | Groq | 0.7 |
| **Governance** | `GovernanceAgent` | `governance_agent.py` (132 ln) | **Meta-agente de cumplimiento**. Revisa TODOS los outputs de los otros agentes. Produce JSON estructurado con `approved`, `risk_level`, `feedback`, `revised_content`, `flags`. | Boundaries + review criteria + formato JSON obligatorio | Gemini | **0.2** (baja — para consistencia) |

**BaseAgent** (`base_agent.py`, 91 líneas):
- Métodos compartidos: `_persona_header()`, `_values_block()`, `_boundaries_block()`
- `generate(prompt, system_prompt, temperature)` → dicta al ModelRouter con `role` del agente
- `review(content, context)` → por defecto auto-approves; GovernanceAgent override con revisión real

### 8.3 Cómo el IdentityCoreAgent construye la identidad

El system prompt del IdentityCoreAgent es el más crítico del sistema — es lo que hace que las respuestas "suenen como" el principal:

1. **Identidad base**: Nombre del principal + declaración de rol
2. **Personalidad (Big Five)**: Cada trait (openness, conscientiousness, etc.) se mapea a texto descriptivo. Ej: `openness: 0.87` → "Very creative, curious, and open to new ideas"
3. **Valores**: Lista priorizada de 7 valores inyectados como "Guiding Values"
4. **Estilo de comunicación**: Formality 0.35 (muy informal), Directness 0.8 (muy directo), Verbosity 0.24 (muy conciso), etc. — mapeados a instrucciones textuales
5. **Boundaries**: 6 reglas hardcodeadas que el agente NUNCA puede violar
6. **Writing style**: Preferencias de tono, estructuración, y elementos de personalidad
7. **AI disclosure**: "Solo revela tu naturaleza AI si te lo preguntan directamente — nunca voluntariamente"
8. **Knowledge Boundary**: Restricción de conocimiento closed-book — el agente SOLO responde con datos de su base de conocimiento local (semantic memory). Si no tiene información aprendida, dice "no sé" y sugiere aprender.
9. **Final enforcement**: Refuerzo final que asegura compliance con el Knowledge Boundary

### 8.4 3 Modos Cognitivos — Control de Inteligencia del Delegado

El sistema implementa 3 modos cognitivos seleccionables desde el chat que controlan qué recursos de inteligencia usa el delegado para responder:

**Parámetro**: `cognitive_mode: int` (1, 2, 3) — enviado en `ChatRequest` → `orchestrator.process()` → `identity_core.respond()`

| Modo | Nombre | Icono | Color | Recursos | Descripción |
|------|--------|-------|-------|----------|-------------|
| 🟢 1 | **Full Intelligence** | Globe | Verde | Web + LLM + toda la memoria | Sin restricciones. El LLM puede usar training data, web search (Tavily), y todas las memorias. |
| 🟡 2 | **Memory + LLM** | Brain | Ámbar | LLM + memoria aprendida | **Default**. El LLM genera respuestas pero SOLO fundamentadas en el contexto de memoria recuperado. Sin web. Filtra semantic memory a `learned_knowledge` solamente. Knowledge Boundary + Status Header activos. |
| 🔴 3 | **Memory Only** | Database | Rojo | Solo memoria (sin LLM) | Recuperación pura de memoria. NO llama al LLM. El orchestrator ensambla las memorias recuperadas en formato `[tier/category] text` y las retorna directamente. Si no hay memorias: retorna "no memories found". Modelo reportado: `memoryOnly`. |

**Componentes del sistema**:
1. **Knowledge Boundary** (identity_core.py): Sección condicional en el system prompt (solo Modo 2) que instruye al LLM a solo responder desde el contexto proporcionado.
2. **Knowledge Status Header** (orchestrator.py): Anotación dinámica inyectada en el contexto (solo Modo 2) que indica si se encontró conocimiento aprendido.
3. **Final Enforcement** (identity_core.py): Refuerzo al final del system prompt (solo Modo 2, recency bias).
4. **Knowledge Sources** (orchestrator.py): Metadata en la respuesta (`knowledge_sources`) con campo `cognitiveMode`.
5. **Memory Filter** (orchestrator.py, paso 4): Modos 2 y 3 filtran semantic memory a solo `learned_knowledge`.
6. **Mode 3 Early Return** (orchestrator.py, paso 6): Retorno directo sin LLM — ensambla memorias como texto formateado.

**Selector en el chat (dashboard)**:
- Botón cíclico: click avanza Mode 1 → 2 → 3 → 1
- Color e icono cambian según el modo activo
- Tooltip muestra descripción del modo actual

**Indicador en burbujas de mensaje**:
- 🟢 `Mode 1` verde → Respuesta con inteligencia completa
- 🟡 `Mode 2` ámbar → LLM + memoria
- 🔴 `Mode 3` rojo → Solo memoria, sin LLM
- 🧠 `Memory (N)` verde → Respuesta basada en N chunks de conocimiento aprendido
- 🌐 `General` ámbar → Contexto semántico pero no conocimiento aprendido

---

## 9. SISTEMA DE MEMORIA — 4 Niveles

### `agent/src/memory/manager.py` (520 líneas)

Interfaz unificada para el sistema de memoria de 4 niveles que permite al sistema recordar, aprender y contextualizar:

```
┌──────────────────────────────────────────────────────────────────────┐
│                        MemoryManager                                 │
│  recall() → build_context() → store_interaction()                   │
│  Budget: ~2000 tokens (prioridad: working > semantic > episodic)     │
├──────────┬───────────┬──────────────┬────────────────────────────────┤
│ WORKING  │ EPISODIC  │  SEMANTIC    │  PROCEDURAL                    │
│ (RAM)    │ (ChromaDB)│  (ChromaDB)  │  (SQLite)                     │
│ Buffer   │ "Qué pasó"│  "Qué sé"   │  "Cómo hago las cosas"       │
│ max 20   │ Vectores  │  Vectores    │  Correcciones + workflows    │
│ mensajes │ + cosine  │  + cosine    │  + estrategias               │
│ Efímero  │ Persiste  │  Persiste    │  Persiste                    │
└──────────┴───────────┴──────────────┴────────────────────────────────┘
```

### 9.1 Detalle por Nivel

| Nivel | Clase | Almacenamiento | Propósito | Capacidad | Persistencia | Búsqueda |
|-------|-------|---------------|-----------|-----------|--------------|----------|
| **Working** | `WorkingMemory` | Python list en RAM | Buffer de contexto activo: mensajes recientes, tarea actual, agente activo, variables de contexto | Últimos 20 mensajes | Se pierde al reiniciar | Lineal (get_recent) |
| **Episodic** | `EpisodicMemory` | ChromaDB collection `episodic_memory` | "¿Qué pasó?". Logs de interacciones con timestamps. Cada interacción user+agent se vectoriza y almacena. | Ilimitado (disco) | `chroma_data/` | Cosine similarity (top-3) |
| **Semantic** | `SemanticMemory` | ChromaDB collection `semantic_memory` | "¿Qué sé?". Conocimiento de identidad: writing samples, expertise, facts aprendidos, RAG knowledge base. | Ilimitado (disco) | `chroma_data/` | Cosine similarity (top-3) |
| **Procedural** | `ProceduralMemory` | SQLite `procedural.db` | "¿Cómo hago las cosas?". Correcciones del training, workflows aprendidos, estrategias, patrones. | Ilimitado (disco) | `data/procedural.db` | SQL query por tipo + keyword |

### 9.2 Flujo de Memoria en el Pipeline

1. **Recall** (paso 4): `memory.recall(message, n_results=3)` — consulta las 3 memorias persistentes por cosine similarity
2. **Build Context** (paso 6): `memory.build_context(memories)` — fusiona resultados con budget de ~2000 tokens, priorizando working > semantic > episodic
3. **Store** (paso 10): `memory.store_interaction(user_msg, agent_response, role, conversation_id)` — guarda en working (append) + episodic (vectorize + store)

### 9.3 Embedding Model

ChromaDB usa por defecto `all-MiniLM-L6-v2` (384 dimensiones, local, gratuito). Cada texto se convierte en un vector de 384 floats para búsqueda por similitud coseno. No hay embeddings externos — todo corre localmente.

---

## 10. MODEL ROUTER — Cadena de Fallback LLM

### `agent/src/router/model_router.py` (360 líneas)

Abstracción sobre los 3 proveedores LLM con fallback automático, asignación por rol de agente, y hot-swap:

```
Solicitud de generación
        │
        ▼
┌───────────────────┐    Falla?    ┌──────────────────┐    Falla?    ┌──────────────────┐
│  GEMINI 2.5 Flash │ ──────────→ │  GROQ Llama 3.3  │ ──────────→ │  OLLAMA llama3.1 │
│  (Free tier)      │             │  70B (Free tier)  │             │  8b (Local)      │
│  $0.15/1M tokens  │             │  $0.05/1M tokens  │             │  $0/token        │
└───────────────────┘             └──────────────────┘             └──────────────────┘
```

**API clave**:
```python
response = await router.generate(
    prompt: str,                  # Texto del usuario
    role: str = "identity_core",  # Determina modelo según asignación
    system_prompt: str = None,    # System prompt del agente
    temperature: float = 0.7,
    max_tokens: int = 2000,
) → ModelResponse(content, provider, model, tokens_used, latency_ms, fallback_used)
```

**Conteo de tokens real** (no estimado):
- Gemini: `response.usage_metadata.total_token_count`
- Groq: `response.usage.total_tokens`
- Ollama: `response['eval_count'] + response['prompt_eval_count']`

**Perfiles predefinidos** (cambiables en caliente, **persistentes en `models.json`**):
| Perfil | Gemini | Groq | Ollama | Uso ideal |
|--------|--------|------|--------|-----------|
| `balanced` | Primary | Fallback | Fallback | Producción normal |
| `max_quality` | Todos los agentes | — | — | Evaluación de máxima fidelidad |
| `privacy_mode` | — | — | Todos los agentes | Datos sensibles — nada sale de la máquina |
| `budget_mode` | — | — | llama3.1:8b (todos) | Mínimo consumo de recursos |

**Comportamiento de perfiles**:
- Los perfiles con `override_all_agents` reescriben las asignaciones de todos los agentes al cambiar.
- Al volver a `balanced`, se restauran los defaults originales guardados en `balanced_defaults`.
- `get_model_for_role()` consulta el perfil activo primero; si tiene override, lo usa para todos los roles.
- Si el proveedor del override no está disponible (e.g. Ollama apagado en `privacy_mode`), falla graciosamente al siguiente en la cadena de fallback.
- El campo `active_profile` y los `agent_assignments` se persisten en `models.json` — sobreviven reinicios.

**Token persistence callback**: En cada llamada LLM, el router invoca `_persist_callback(provider, model, tokens_used, latency_ms, role)` que escribe a la tabla `token_usage` en Postgres (fire-and-forget).

---

## 11. SISTEMA DE EVALUACIÓN — 5 Módulos Heurísticos

Cada interacción del orchestrator es evaluada por 5 módulos **sin llamadas LLM adicionales** (heurísticas puras). Los resultados se almacenan in-memory (último N) + Postgres (ilimitado).

### 11.1 Quality Scorer — `quality_scorer.py` (383 líneas)

Evalúa calidad en 5 dimensiones ponderadas:

| Dimensión | Peso | Qué mide | Cómo lo mide |
|-----------|------|----------|--------------|
| Relevance | 30% | ¿La respuesta aborda la consulta? | Distancias de memoria recalled + match de categoría |
| Coherence | 20% | ¿Bien estructurada y lógica? | Longitud, estructura, completitud |
| Completeness | 20% | ¿Respuesta completa? | Profundidad de recall + thoroughness |
| Efficiency | 15% | ¿Eficiente en recursos? | Latencia vs target (2000ms), uso de fallback |
| Memory Utilization | 15% | ¿Usó contexto recuperado? | Correlación entre memoria recalled y respuesta |

**Output**: `QualityReport` — composite_score (0.0-1.0), grade (A/B/C/D/F), flags, dimensiones detalladas.

### 11.2 Alignment Evaluator — `alignment_evaluator.py` (383 líneas)

Evalúa alineación con la personalidad del principal:

| Aspecto | Qué mide | Método |
|---------|----------|--------|
| Value alignment | ¿Refleja los 7 valores del principal? | Keywords positivas/negativas por valor |
| Communication alignment | ¿Estilo correcto? (formalidad, directness, verbosity) | Heurísticas de longitud, estructura, tono |
| Boundary compliance | ¿Viola algún boundary? | Matching de restricciones del persona.yaml |
| Decision alignment | ¿Toma de decisiones correcta? | Risk tolerance, analysis depth, stakeholder weight |

**Output**: `AlignmentReport` — overall_score, value_alignment, communication_alignment, boundary_compliance, decision_alignment, violations[], observations[].

### 11.3 Legal Risk Indicator — `legal_risk.py` (319 líneas)

Escanea respuestas por patrones de riesgo legal usando 15+ regexes en 6 categorías:

| Categoría | Ejemplos de patrones | Severidad |
|-----------|---------------------|-----------|
| Contractual | "we agree to", "commit to", "guarantee" | high |
| Financial | "invoice", "will charge", "pricing confirmed" | high |
| Confidentiality | "client data", "proprietary", "NDA" | medium-high |
| Liability | "warrant", "indemnify", "represent and warrant" | high |
| Regulatory | "GDPR", "personally identifiable information" | medium |
| AI Disclosure | Falta de disclosure cuando se requiere | medium |

**Output**: `LegalRiskReport` — overall_risk (none/low/medium/high/critical), risk_score (0.0-1.0), flags[], requires_review, auto_blocked.

### 11.4 Decision Registry — `decision_registry.py` (358 líneas)

Detecta y registra decisiones de negocio en las respuestas:

- Identifica triggers (keywords de 7 categorías: strategic, financial, operational, client, technical, communication, organizational)
- Extrae: trigger, context, decision, rationale, alternatives, trade_offs, stakeholders, financial_impact
- Clasifica urgencia y riesgo
- Marca si requiere aprobación del principal

**Output**: `BusinessDecision` dataclass — 20+ campos incluyendo status (pending/approved/executed/rejected/deferred).

### 11.5 Memory Rollback Manager — `memory_rollback.py` (353 líneas)

Auditoría y recuperación point-in-time del sistema de memoria:

- **Tracking**: Cada operación de memoria (store, delete, modify) se registra con before/after state
- **Checkpoints**: Puntos de restauración nombrados (`create_checkpoint("before_training_session_3")`)
- **Undo**: Reversa operaciones individuales por `operation_id`
- **Rollback**: Reversa todas las operaciones hasta un checkpoint
- **Persistence callback**: Cada operación se persiste a `memory_operations` en Postgres via callback wired at startup

**Storage in-memory**: Máximo 1000 operaciones, 50 checkpoints.

### 11.6 Tabla resumen

| Módulo | Líneas | In-memory max | Postgres | Singleton |
|--------|--------|---------------|----------|-----------|
| Quality Scorer | 383 | 200 reports | evaluations table | `quality_scorer` |
| Alignment Evaluator | 383 | 200 reports | evaluations table | `alignment_evaluator` |
| Legal Risk | 319 | 200 reports | evaluations table | `legal_risk_indicator` |
| Decision Registry | 358 | 500 decisions | evaluations table | `decision_registry` |
| Memory Rollback | 353 | 1000 ops, 50 checkpoints | memory_operations table | `memory_rollback` |

---

## 12. COGNITIVE TRACE — Observabilidad del Pipeline

### `agent/src/trace/collector.py` (342 líneas)

Cada interacción del orchestrator genera un **grafo cognitivo** compatible con React Flow, registrando cada paso del pipeline como un nodo:

**13 tipos de nodo**:
```
input → classify → decision_engine → planner → memory_recall → correction →
prompt_build → llm_generate → identity_review → governance_review →
memory_store → evaluation → output
```

**Estructura de datos**:
```python
@dataclass
class TraceNode:
    node_id: str              # Identificador único (ej: "classify", "llm_generate")
    node_type: str            # Uno de los 13 tipos
    label: str                # "Task Classification", "LLM Generation (business)"
    input_data: Any           # Datos de entrada al paso
    output_data: Any          # Datos de salida del paso
    processing_summary: str   # "Classified as business" o "Generated via gemini/2.5-flash (1234ms)"
    metrics: Dict             # latency_ms, tokens, confidence, risk, provider, model
    status: str               # pending → running → completed | failed | skipped
    persist_id: Optional[int] # PK en Postgres (enriched post-persistence)
    model_used: Optional[str] # Modelo LLM (enriched para llm_generate nodes)
    risk_score: Optional[float] # Score de riesgo (enriched para governance_review nodes)
```

**TraceStore**: Singleton que almacena las últimas 100 trazas en memoria. Cada traza se convierte en un grafo `{nodes: [...], edges: [...], metadata: {...}}` compatible con React Flow para visualización directa en el dashboard.

**Auto-layout**: Los nodos se posicionan automáticamente con spacing vertical de 120px, creando un pipeline visual de arriba abajo.

---

## 13. EVENT BUS — Pub/Sub + WebSocket Broadcast

### `agent/src/events/event_bus.py` (128 líneas)

Sistema de eventos centralizado que broadcast simultáneamente a 3 destinos:

```
Evento emitido (IAmeEvent)
        │
        ├──→ WebSocket → Dashboard (live feed, agent ring, KPIs)
        ├──→ Postgres audit_log → Persistencia permanente
        └──→ In-memory subscribers → Agentes internos
```

**Estructura de evento**:
```python
@dataclass
class IAmeEvent:
    event_type: str       # "orchestrator", "decision", "agent_state", "chat", "system"
    action: str           # "task_classified", "thought", "action", "observation", "startup"
    agent_role: str       # "identity_core", "business", etc.
    details: Dict         # Datos específicos del evento
    risk_level: str       # "low", "medium", "high"
    timestamp: str        # ISO 8601 UTC
```

**Eventos clave emitidos por el Orchestrator**:
| Momento | event_type | action | Contenido |
|---------|-----------|--------|-----------|
| Clasificación completada | orchestrator | task_classified | category, message_preview |
| Decisión cognitiva | decision | thought | reasoning, strategy, risk, memory_query |
| Inicio de acción | decision | action | action_taken, has_memory_context |
| Pipeline completado | orchestrator | task_completed | category, latency_ms, provider, trace_id, governance |
| Resultado final | decision | observation | category, provider, model, latency_ms, response_preview |

**Historial**: Últimos 100 eventos en memoria + todos los eventos en Postgres `audit_log`.

---

## 14. GOVERNANCE — Marco de Control y Autonomía

### 14.1 Configuración — `configs/governance.yaml` (254 líneas)

Define el framework completo de gobernanza del sistema:

**5 Niveles de Autonomía Progresiva**:
| Nivel | Nombre | Descripción | Aprobación requerida | Puede ejecutar |
|-------|--------|-------------|---------------------|----------------|
| 0 | **Observer** | El agente sugiere, el humano ejecuta todo | Todas las acciones | Nada |
| 1 | Assistant | Ejecuta bajo riesgo, sugiere alto riesgo | high_risk, critical | low_risk |
| 2 | Delegate | Ejecuta la mayoría, escala edge cases | critical, financial, edge_cases | low-high risk |
| 3 | Autonomous | Opera independiente dentro de boundaries | critical, boundary_violations | Todo excepto legal |
| 4 | Trusted | Autoridad operacional completa | Cambios de scope legal | Todo |

**Estado actual**: Nivel 0 (Observer) — todas las acciones son sugerencias. El sistema no puede ejecutar acciones externas por sí solo.

### 14.2 GovernanceAgent (Meta-agente)

Opera con temperatura **0.2** (baja, para consistencia). Revisa cada output no-conversacional y produce JSON estructurado:
```json
{
  "approved": true,
  "risk_level": "low",
  "feedback": "Content aligns with principal's values",
  "revised_content": null,
  "flags": []
}
```

**Comportamiento en errores de parsing**: Si la respuesta del LLM no es JSON válido, el GovernanceAgent **auto-approves** (fallo abierto). Esto es un trade-off consciente: no bloquear la experiencia del usuario por un error de formato.

### 14.3 Emergency Stop

Mecanismo de parada de emergencia accesible desde:
- Dashboard: Botón rojo "Emergency Stop" en Governance Console
- API: `POST /api/governance/emergency-stop`
- Código: `orchestrator.emergency_stop()`

Cuando activado, **todo** procesamiento se detiene en el paso 0 del pipeline. Requiere `POST /api/governance/emergency-resume` para reanudarse.

### 14.4 Governance stub — `agent/src/governance/`

Directorio placeholder. La lógica de governance actual vive en:
- `governance.yaml` (configuración)
- `GovernanceAgent` (revisión via LLM)
- `Orchestrator` flags (`governance_enabled`, `_emergency_stopped`)

No hay enforcement middleware automático ni human-in-the-loop real todavía.

---

## 15. TRAINING — Sistema de Entrenamiento Progresivo

### `agent/src/training/manager.py` (434 líneas)

Sistema de entrenamiento con 3 modos diseñados para que el principal enseñe progresivamente a su conciencia virtual:

### 15.1 Modos de Entrenamiento

| Modo | Propósito | Flujo |
|------|-----------|-------|
| **correction** | Corregir una respuesta incorrecta | El principal da: original response + corrección deseada + explicación. Se almacena como regla en ProceduralMemory. |
| **free_conversation** | Conversación libre para calibrar personalidad | El principal conversa normalmente. El sistema analiza el mensaje con LLM (role=identity_core, temp=0.4) para extraer rasgos de personalidad, estilo, preferencias y valores. Los rasgos extraídos se almacenan en SemanticMemory (category=personality_trait). El contexto completo se almacena en EpisodicMemory. El historial de exchanges se mantiene en sesión. |
| **guided_interview** | Preguntas estructuradas para capturar identidad | El sistema presenta 15 preguntas predefinidas sobre valores, comunicación, toma de decisiones, límites y personalidad. Las respuestas se analizan con LLM para extraer rasgos, que se almacenan en SemanticMemory (category=interview_response). Progreso visual con barra de completitud. |

### 15.2 Correcciones

Las correcciones son la forma más directa de entrenamiento. Cada corrección se almacena en ProceduralMemory (SQLite) y se inyecta en el prompt de cada interacción futura:

```python
# Ciclo de corrección:
1. El agente responde algo
2. El principal marca la respuesta como incorrecta
3. Provee la respuesta correcta + explicación
4. La corrección se almacena: {agent_role, original, correction, explanation, timestamp}
5. En futuras interacciones, build_correction_context() recupera correcciones relevantes
6. Se inyectan en el prompt como: "Previously corrected behavior: [...]"
```

### 15.3 Upload de Writing Samples

El endpoint `POST /training/upload-samples` acepta archivos de texto y los procesa:
1. Lee el contenido del archivo
2. Lo divide en chunks de ~500 palabras
3. Cada chunk se almacena en **semantic memory** (ChromaDB) con metadata de tipo "writing_sample"
4. Estos chunks se recuperan vía RAG cuando el agente necesita emular el estilo de escritura del principal

---

## 16. SKILLS — Registro de Habilidades

### 16.1 Registry — `agent/src/skills/registry.py` (116 líneas)

Registro de habilidades con toggle enable/disable. Carga desde `skills.json`:

| Skill | Riesgo | Estado | Descripción |
|-------|--------|--------|-------------|
| web-research | low | Habilitada | Búsqueda web via Tavily API |
| email-draft | medium | Habilitada | Redacción de emails via CommunicationAgent |
| document-gen | medium | Habilitada | Generación de documentos via CommunicationAgent |
| learn-topic | low | Habilitada | Investiga un tema en la web, resume con LLM, y almacena el conocimiento en memoria semántica. Activable desde chat ("aprende sobre X") |

### 16.2 Web Research — `agent/src/skills/web_research.py` (106 líneas)

Wrapper sobre Tavily API con dos modos:
- **Single search**: Query → top 5 resultados con snippets
- **Deep research**: Query → genera sub-queries → busca cada una → consolida resultados

### 16.3 Tools — `agent/src/skills/tools.py` (110 líneas)

- `EmailDraftTool`: Usa CommunicationAgent para generar draft de email
- `DocumentGenTool`: Usa CommunicationAgent para generar documentos

### 16.4 Learn Topic — `agent/src/skills/learn_topic.py` (261 líneas)

Skill de aprendizaje profundo: investiga un tema en la web, resume con LLM, y almacena chunks de conocimiento en memoria semántica para uso persistente en todas las conversaciones futuras.

**Pipeline**:
1. `_generate_queries(topic, depth)` — genera sub-queries según profundidad (1-3)
2. `WebResearchTool.search()` — búsqueda web via Tavily por cada query
3. `_compile_raw_text()` — compila resultados en texto para el LLM
4. `_summarize_and_chunk()` — LLM (role=technical, temp=0.3) genera 4-8 chunks con delimitador `[CHUNK]`
5. `_parse_chunks()` / `_fallback_chunk()` — parsea chunks o fallback por párrafos
6. `SemanticMemory.store()` — almacena cada chunk con metadata (topic, source, learned_at, depth)

**Chat triggers** (`detect_learn_request()`): Regex patterns para español e inglés:
- "aprende sobre X", "hazte experto en Y", "investiga sobre Z"
- "learn about X", "become an expert in Y", "research about Z"

**Integrado en**: Orchestrator (paso 1a) + `POST /api/skills/learn-topic` + Skill Manager UI

---

## 17. PERSISTENCIA — Postgres + ChromaDB + SQLite

### 17.1 Database — `agent/src/db/database.py` (289 líneas)

Conexión a Neon Postgres via **psycopg2** (síncrono, autocommit). **Totalmente opcional** — el sistema funciona sin base de datos.

**10 tablas en 2 grupos**:

#### Grupo Core (5 tablas):
| Tabla | PK | Propósito | Índices |
|-------|-----|-----------|---------|
| `conversations` | TEXT id | Sesiones de chat | — |
| `messages` | SERIAL | Mensajes individuales con provider, model, latency | conversation_id, created_at |
| `audit_log` | SERIAL | Cada evento del EventBus persistido | timestamp, event_type |
| `kpi_snapshots` | SERIAL | Métricas periódicas para gráficos | metric_name + timestamp |
| `config_versions` | SERIAL | Snapshots versionados de configuración | — |

#### Grupo Phase 1 Persistence (5 tablas):
| Tabla | PK | Propósito | Campos clave |
|-------|-----|-----------|-------------|
| `interactions` | TEXT id | Una fila por ejecución del pipeline | category, agent_role, provider, model, latency_ms, tokens_used, user_message, response_preview |
| `trace_nodes` | SERIAL | Cada paso del pipeline para replay | interaction_id, node_id, node_type, input_data (JSONB), output_data (JSONB), metrics (JSONB) |
| `evaluations` | SERIAL | Resultados de los 5 módulos de evaluación | interaction_id, eval_type, scores (JSONB) |
| `token_usage` | SERIAL | Cada llamada LLM individual | provider, model, tokens_used, latency_ms, cost_estimate, role |
| `memory_operations` | SERIAL | Cada operación de memoria | tier, action, target_id, before_state, after_state, rolled_back |

### 17.2 Persistence Repository — `agent/src/db/persistence.py` (1313 líneas)

Repositorio fire-and-forget que nunca bloquea ni crashea el pipeline:

```python
class PersistenceRepository:
    # Write methods
    def save_interaction(...)      → bool   # Guarda metadata de interacción
    def save_trace_nodes(...)      → Dict   # Bulk insert de nodos + edges, retorna {node_id: pk}
    def save_all_evaluations(...)  → bool   # Guarda quality, alignment, legal, decisions, identity evals
    def save_token_usage(...)      → bool   # Guarda uso de tokens (via callback del ModelRouter)
    def save_memory_operation(...) → bool   # Guarda operación de memoria (via callback del RollbackManager)
    
    # Read methods (21 public total)
    def get_evaluation_quality(...)    → list  # Evaluaciones de calidad desde Postgres
    def get_evaluation_alignment(...)  → list  # Evaluaciones de alignment
    def get_evaluation_legal_risk(...) → list  # Evaluaciones de riesgo legal
    def get_evaluation_decisions(...)  → list  # Evaluaciones de decisiones
    def get_evaluation_rollback(...)   → list  # Evaluaciones de rollback
    def get_recent_identity_signals(...) → list  # Señales de identidad recientes
    def list_traces(...)               → list  # Lista de trazas
    def get_analytics_overview(...)    → dict  # Overview de analytics
    def get_autonomy_stats(...)        → dict  # Estadísticas de autonomía
    
    # Delete methods
    def delete_evaluation_data(...)    → dict  # Elimina datos de evaluación
    def delete_analytics_events(...)   → dict  # Elimina eventos de analytics
    def delete_analytics_tokens(...)   → dict  # Elimina datos de token usage
```

**Principio de diseño**: Cada método tiene `try/except` interno. Un fallo de persistencia NUNCA afecta la respuesta al usuario.

### 17.3 Resumen de Storage

| Motor | Ubicación | Propósito | Qué almacena |
|-------|-----------|-----------|-------------|
| **Neon Postgres** | Cloud (remoto) | Persistencia relacional | Conversaciones, mensajes, audit log, interacciones, traces, evaluaciones, token usage, memory ops |
| **ChromaDB** | `chroma_data/` (local) | Vector store | Episodic memory (logs de interacciones), Semantic memory (knowledge base, writing samples) |
| **SQLite** | `data/procedural.db` (local) | Memoria procedimental | Correcciones del training, workflows, estrategias |
| **RAM** | In-process | Cache rápida | Working memory (per-conversation, max 20 msgs/conv, max 50 sessions, LRU eviction, 1h TTL), últimas 100 traces, últimos 200 reports de evaluación |

---

## 18. DASHBOARD — Interfaz de Control (Next.js)

### 18.1 Stack del Dashboard

- **Next.js 15** (App Router) + **React 19** + **TypeScript 5.7**
- **Tailwind CSS 3.4** + **shadcn/ui** para componentes
- **Tema lab oscuro** con variables CSS: `lab-text`, `lab-text-dim`, `lab-card`, `lab-surface`, `lab-border`, `accent-glow`, `accent-primary`, `status-green/amber/red/blue`
- **Zustand 5** para state management global (165 líneas)
- **API client** centralizado en `lib/api.ts` (1117 líneas, ~96 métodos tipados)
- **i18n** vía `lib/i18n/` — soporta `en.json` (~531 keys) + `es.json`
- Directiva `"use client"` en todas las páginas interactivas

### 18.2 Rutas del Dashboard (13 páginas)

| Ruta | Página | Líneas | Funcionalidad |
|------|--------|--------|--------------|
| `/` | Command Center | 189 | KPI cards (conversations, tokens, uptime, quality), agent status ring (5 agentes con estado visual), health LEDs, activity feed (real-time via WebSocket), persona card, router card, quick actions |
| `/chat` | Chat Interface | 11 (wrapper) + 167 (ChatPanel) + 51 (MessageBubble) | Chat de texto con el sistema orquestado. Envía POST /api/chat y renderiza respuestas con metadata (provider, model, latency). **3 Modos Cognitivos**: botón cíclico (Globe/Brain/Database) para controlar nivel de inteligencia (Full 🟢, Memory+LLM 🟡, Memory Only 🔴). **Indicadores de fuente**: 🧠 Memory (verde, N chunks) / 🌐 General (ámbar) + modo cognitivo por mensaje. |
| `/identity` | Identity Studio | 417 | **Big Five sliders + radar chart** (5 dimensiones), estilo de comunicación (6 sliders), value hierarchy (drag-and-drop reordenable), behavioral boundaries, save/reload desde persona.yaml |
| `/training` | Training Center | 478 | 3 modos de entrenamiento (correction, **free conversation** con extracción de rasgos + almacenamiento en memoria semántica, **guided interview** con 15 preguntas + barra de progreso), historial de sesiones, upload de writing samples con preview |
| `/testing` | Testing Playground | 366 | Chat simulator, scenario theater (escenarios predefinidos), A/B compare (comparar 2 respuestas side-by-side) |
| `/models` | Model Manager | 352 | Estado de proveedores (Gemini/Groq/Ollama), 4 perfiles switcheables, asignaciones por agente (qué modelo usa cada agente), endpoint de test |
| `/skills` | Skill Manager | 254 | Lista de skills con toggle on/off, ejecución de web research directa, **learn-topic UI** con selector de profundidad (basic/moderate/deep) y activación desde chat |
| `/memory` | Memory Lab | 494 | Stats cards (count por tier), semantic search interactivo, store new memories, **edit inline** y delete individual memories |
| `/governance` | Governance Console | 481 | Niveles de autonomía visual, configuración (read-only), audit log con filtros, approval queue con acciones funcionales (approve/reject/request changes), emergency stop/resume button |
| `/analytics` | Analytics Dashboard | 269 | Identity fidelity gauge, autonomy metrics, token usage charts (recharts), events breakdown by type/agent/risk |
| `/trace` | Cognitive Trace | 474 | Lista de trazas recientes, visualización React Flow del pipeline (nodos custom con expand/collapse), replay paso a paso |
| `/evaluation` | Evaluation Dashboard | 698 | Vista unificada de los 5 módulos: quality trends, alignment scores, legal risk flags, business decisions, memory rollback log |
| `/identity-governance` | Identity Governance | 66 + 295 + 255 + 240 + 295 + 225 | 4 tabs: Versions (snapshot modal with version/tags/label/notes, status badges Active/Candidate/Historical, tag badges, activation confirmation dialog), Evolution (approve/reject candidates), Shadow Simulation (risk grades, structural diff), Identity Health (signals timeline, CSS bar charts, pre-training baseline badge). Components: `create-snapshot-modal.tsx` (modal with tag multi-select + custom tags + version validation) |

### 18.3 Componentes Clave

| Directorio | Componentes | Líneas totales | Descripción |
|------------|------------|----------------|-------------|
| `components/command-center/` | 7 componentes | 1026 | KPI cards, activity feed (ScrollArea con eventos real-time), agent ring (SVG circular con 5 agentes), health bar (LEDs de servicios), persona card, router card, quick actions (i18n completo) |
| `components/chat/` | 2 componentes | 218 | ChatPanel (input + message list + cognitive mode cycling selector + submission) + MessageBubble (render individual con metadata + knowledge source indicators + cognitive mode indicator) |
| `components/layout/` | 3 componentes | 266 | Sidebar (navegación agrupada en 4 secciones: Core, Identity & Training, Infrastructure, Observability), Header (breadcrumb + system status), ClientShell (wrapper con font loading) |
| `components/trace/` | 1 componente | 267 | TraceNode — nodo custom de React Flow con expand/collapse, colores por tipo, métricas inline |
| `components/ui/` | 10 componentes | 340 | Primitivos shadcn/ui: badge, button, card, confirm-dialog, input, progress, scroll-area, separator, tabs, tooltip |

### 18.4 API Client — `dashboard/lib/api.ts` (~1117 líneas)

~96 métodos tipados que mapean 1:1 a los endpoints del backend. Incluye métodos para service-log, interactions persistence, persisted token usage, learn-topic, free conversation exchange, guided interview, knowledge sources, training corrections CRUD, trace/evaluation/analytics deletion, y memory bulk-delete:

```typescript
export const api = {
    // Chat — respuesta incluye knowledge_sources (closed-book metadata)
    chat: (message, conversationId?, onlyLocalKnowledge?) => fetchAPI<ChatResponse>("/chat", { method: "POST", body }),
    // Memory
    memoryStats: () => fetchAPI<MemoryStats>("/memory/stats"),
    memorySearch: (query, tier?, limit?) => fetchAPI<SearchResults>("/memory/search", { method: "POST", body }),
    // Evaluation
    evaluationOverview: () => fetchAPI<EvalOverview>("/evaluation/overview"),
    // ... ~55 métodos más
};
```

---

## 19. FLUJO COMPLETO END-TO-END

```
                                 ┌────────────────────────────┐
                                 │  Usuario escribe mensaje   │
                                 │  en ChatPanel del Dashboard│
                                 └─────────────┬──────────────┘
                                               │
                                               ▼
                                 ┌────────────────────────────┐
                                 │  api.chat(message)         │
                                 │  POST /api/chat            │
                                 └─────────────┬──────────────┘
                                               │
                              ┌────────────────▼──────────────────────┐
                              │         Route Handler (routes.py)     │
                              │  1. Crear conversación en Postgres    │
                              │  2. Guardar mensaje del usuario en DB │
                              │  3. Emit "agent_state.thinking" →WS   │
                              │  4. Cargar historial (últimos 20 msgs)│
                              │  5. Emit "agent_state.acting" →WS    │
                              │  6. orchestrator.process(msg, id, hist, │
                              │     cognitive_mode)                  │
                              └────────────────┬─────────────────────┘
                                               │
                              ┌────────────────▼──────────────────────┐
                              │       ORCHESTRATOR PIPELINE           │
                              │                                       │
                              │  0. Emergency Check (si parado → STOP)│
                              │  0.5 Learn-topic auto-detect          │
                              │  1. Classify (keywords, <1ms)         │
                              │  2. Decision Engine (determinístico)   │
                              │  3c. Identity Decision Modulation     │
                              │  3d. Identity Confidence (Phase 6D)   │
                              │  3. Planner (Plan con steps)          │
                              │  4. Memory Recall + Mode filter      │
                              │     filter (strip non-learned chunks)  │
                              │  5. Correction Injection (SQLite)     │
                              │  6. Prompt Build + Mode Logic        │
                              │     (deterministic decline if no       │
                              │      learned knowledge + knowledge Q)  │
                              │  7. LLM Generate → ModelRouter →      │
                              │     → Gemini/Groq/Ollama              │
                              │  8. Identity Review (si Plan dice sí) │
                              │  9. Governance Review (si Plan dice sí)│
                              │  10. Memory Store + 5 Evaluaciones +  │
                              │      Persist to Postgres              │
                              │  + TraceCollector graba cada paso     │
                              └────────────────┬─────────────────────┘
                                               │
                              ┌────────────────▼──────────────────────┐
                              │      Route Handler (return)           │
                              │  1. Guardar respuesta en Postgres     │
                              │  2. Emit "chat.response_generated"    │
                              │  3. Emit "agent_state.idle" →WS      │
                              │  4. Return ChatResponse JSON          │
                              └────────────────┬─────────────────────┘
                                               │
                                               ▼
          ┌──────────────────────────────────────────────────────────┐
          │                    Dashboard actualiza                   │
          │  • ChatPanel muestra el mensaje con metadata            │
          │  • Knowledge source indicators (🧠/🌐) por mensaje       │
          │  • WebSocket events → Command Center actualiza:         │
          │    - Agent ring (estado de cada agente)                 │
          │    - Activity feed (nuevo evento)                       │
          │    - KPI cards (contadores actualizados)                │
          │  • Trace Store → /trace puede mostrar el pipeline      │
          │  • Evaluation → /evaluation puede mostrar scores       │
          └──────────────────────────────────────────────────────────┘
```

**Latencia típica end-to-end**: 1-10 segundos (dominado por paso 7: LLM Generate)
- Conversación simple (solo identity_core): 1-3 segundos
- Con identity + governance review: 3-8 segundos
- Con fallback a Ollama local: 5-15 segundos

---

## 20. GARANTÍAS ARQUITECTURALES (Phase 3)

Estas garantías fueron establecidas en Phase 3 (Architectural Hardening) y son invariantes que deben mantenerse en todas las fases futuras:

| # | Garantía | Mecanismo | Verificación |
|---|----------|----------|-------------|
| 1 | **Cognición obligatoria** | `Orchestrator.__init__` hace `isinstance()` check contra `DecisionEngine` y `Planner`. Lanza `TypeError` si falta cualquiera. | `test_cognition.py` — 3 tests de rechazo |
| 2 | **Sin ruta legacy** | No existe `_category_to_role()`, ni `if self.decision_engine is None`, ni ternarios `plan else`. Todo fue eliminado en Phase 3. | Grep del codebase confirma 0 ocurrencias |
| 3 | **Inmutabilidad del DecisionEngine** | `__slots__` + `__setattr__` override bloquean toda mutación post-construcción. | `test_cognition.py` — 2 tests de inmutabilidad |
| 4 | **Single source of truth para governance** | `governance_enabled` vive **únicamente** en `Orchestrator`. `Planner.build()` lo recibe como parámetro — no tiene copia propia. | Constructor de Planner no acepta governance_enabled |
| 5 | **Sin dependencia circular** | `TaskCategory` vive en `src/flows/categories.py` — importado unidireccionalmente por orchestrator y cognition. | Import test circular confirma 0 errores |
| 6 | **Orchestrator es execution-only** | No contiene lógica de decisión estratégica. `_route()` hace dispatch por categoría pero no decide estrategia, riesgo, ni gates. | Code review + Phase 3 audit |
| 7 | **Cognición unit-testable** | 49 tests puros sin mocks ni IO en `tests/test_cognition.py`. Plan determinístico y congelado. | `pytest tests/test_cognition.py` — 49/49 pass |

---

## 21. IDENTITY CORE — MÓDULO DE IDENTIDAD FORMAL (Phase 4)

Phase 4 introduce un módulo de identidad estructurado en `agent/src/identity/` (22 archivos, ~6080 líneas) que formaliza la representación de la identidad del principal en un `IdentityProfile` versionado, con embedding baseline y drift detection.

### 21.1 Arquitectura del Módulo

```
src/identity/
├── __init__.py            # Re-exports: IdentityProfile, IdentityManager, IdentityEnforcer, IdentityPolicyEngine, IdentityMemoryBridge, IdentityContextWeighter, IdentityDecisionModulator, IdentityConfidenceEngine, IdentityAutonomyModulator, IdentityRetrievalWeighter, IdentityConsolidationWeighter, IdentityBehavioralBias, IdentityPromptIntegrator, IdentityHealthMonitor, IdentityHealthRegulator, IdentityEvolutionEngine, IdentityShadowSimulator, IdentityVersionControl
├── schema.py      (~105) # Pydantic model IdentityProfile
├── embedding.py   (~215) # Text composition + ChromaDB embedding + cosine similarity
├── versioning.py  (~215) # Semantic versioning + SHA-256 hashing + Postgres persistence
├── enforcement.py (~90)  # Phase 5A: Soft drift detection (IdentityEnforcer)
├── policy.py      (~165) # Phase 5B: Reactive identity control (IdentityPolicyEngine)
├── feedback.py    (~110) # Phase 5C: Controlled identity feedback (IdentityFeedbackController)
├── memory_bridge.py(~165)# Phase 6A: Identity-aware memory affinity analyser (IdentityMemoryBridge)
├── context_weighting.py(~200) # Phase 6B: Soft identity-aware context annotation (IdentityContextWeighter)
├── decision_modulation.py(~280) # Phase 6C: Observational decision-identity alignment (IdentityDecisionModulator)
├── confidence.py  (~230) # Phase 6D: Soft autonomy modulation (IdentityConfidenceEngine)
├── autonomy_modulation.py(~175) # Phase 7A: Soft governance coupling (IdentityAutonomyModulator)
├── retrieval_weighting.py(~210) # Phase 7B: Identity-weighted memory retrieval (IdentityRetrievalWeighter)
├── consolidation_weighting.py(~210) # Phase 7C: Identity-weighted memory consolidation (IdentityConsolidationWeighter)
├── behavioral_bias.py(~500) # Phase 8A: Identity behavioral bias layer (IdentityBehavioralBias)
├── prompt_integration.py(~195) # Phase 8B: Soft identity prompt integration (IdentityPromptIntegrator)
├── health_monitor.py(~320)  # Phase 9A: Identity longitudinal monitoring (IdentityHealthMonitor)
├── health_regulation.py(~260) # Phase 9B: Health-aware adaptive regulation (IdentityHealthRegulator)
├── evolution.py   (~600) # Phase 10A: Dynamic identity evolution engine (IdentityEvolutionEngine)
├── shadow_simulation.py(~528) # Phase 10B: Shadow identity simulation layer (IdentityShadowSimulator)
├── version_control.py(~710) # Phase 10C: Immutable identity versioning + controlled apply + rollback (IdentityVersionControl)
└── manager.py     (~255) # Singleton lifecycle manager
```

**Grafo de dependencias**:
```
persona.yaml → manager.py → schema.py + embedding.py + versioning.py
                                ↓              ↓
                          IdentityProfile   ChromaDB DefaultEmbeddingFunction
                                ↓              (all-MiniLM-L6-v2, 384-dim)
                          DecisionEngine (inyección, read-only)
                          AlignmentEvaluator (baseline para drift detection)
```

### 21.2 IdentityProfile — `schema.py`

Modelo Pydantic que estructura la identidad del principal:

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `version` | `str` | Versión semántica vX.Y.Z (validado por regex) |
| `created_at` | `datetime` | UTC timestamp de creación |
| `principal_name` | `str` | Nombre del principal |
| `big_five` | `Dict[str, float]` | Rasgos Big Five (0.0–1.0 validado) |
| `values` | `List[str]` | Valores personales ordenados por prioridad |
| `communication_style` | `Dict[str, float]` | Dimensiones de comunicación |
| `boundaries` | `List[str]` | Límites conductuales |
| `writing_style` | `Dict[str, Any]` | Reglas de estilo de escritura |
| `expertise` | `Dict[str, Any]` | Áreas de expertise primaria/secundaria |
| `decision_making` | `Dict[str, Any]` | Patrones de toma de decisiones |
| `baseline_embedding` | `List[float]` | Vector de 384 dimensiones (all-MiniLM-L6-v2) |
| `drift_threshold` | `float` | Umbral mínimo de similitud coseno (default 0.78) |
| `content_hash` | `Optional[str]` | SHA-256 para detección de cambios |

Propiedades: `has_baseline` → bool, `to_persistable()` → dict JSON-safe.

### 21.3 Embedding — `embedding.py`

- **`build_identity_text(persona_data, writing_samples)`** — Concatena 8 secciones de identidad (principal, Big Five, valores, límites, comunicación, escritura, decisiones, expertise) + top 3 writing samples (max 500 chars cada uno) en un texto largo para embedding.
- **`compute_baseline_embedding(text)`** — Usa `chromadb.utils.embedding_functions.DefaultEmbeddingFunction()` (all-MiniLM-L6-v2) para producir un vector de 384 dimensiones. Mismo proveedor que la memoria semántica/episódica.
- **`cosine_similarity(vec_a, vec_b)`** — Similitud coseno manual (dot product / product of norms). Retorna float en [0.0, 1.0] para vectores normalizados.

### 21.4 Versioning — `versioning.py`

- **Semantic Versioning**: `generate_version()` (patch), `generate_minor_version()`, `generate_major_version()` — incrementan partes del version string vX.Y.Z.
- **Content Hashing**: `hash_identity(text)` → SHA-256 hex de 64 caracteres. Permite detectar si el contenido de identidad realmente cambió entre builds.
- **Persistencia**: Reutiliza la tabla `config_versions` de Postgres con `config_type='identity_profile'`. Métodos: `save_version()`, `load_latest_version()`, `get_version_history()`.

### 21.5 IdentityManager — `manager.py`

Singleton que orquesta el ciclo de vida completo:

1. **`load_from_persona_yaml()`** — Parsea `configs/persona.yaml`
2. **`build_profile()`** — Pipeline: YAML → text → hash → check change → embed → version → construct `IdentityProfile`. Si `content_hash` no cambió, retorna el perfil activo existente (short-circuit).
3. **`save_profile()`** — Delega a `IdentityVersionManager.save_version()` para persistir en Postgres
4. **`load_active_profile()`** — DB primero, fallback a rebuild desde YAML
5. **`rebuild_profile()`** — Fuerza reconstrucción desde `persona.yaml`
6. **`get_status()`** — Datos de monitoreo (version, hash, embedding dims, etc.)

### 21.6 Integración en el Sistema

**AppState (main.py)**: `IdentityManager` se inicializa en lifespan después de TrainingManager y antes de Cognition Layer:
```
Settings → DB → ModelRouter → Crew → Memory → Skills → Training → IdentityManager → Cognition → AlignmentEvaluator wiring → Orchestrator
```

**DecisionEngine**: Recibe `identity_profile` en constructor (read-only via `@property`). **No altera `evaluate()`** — preparado para uso futuro en identity-aware cognition.

**AlignmentEvaluator**: Nuevo campo `identity_similarity` en `AlignmentReport`. `set_baseline_embedding()` wired at startup. `compute_identity_similarity()` calcula similitud coseno de cada respuesta contra el baseline. Score **no incluido** en `overall_score` (preserva pesos heurísticos existentes). Valor -1.0 cuando baseline no disponible.

### 21.7 Enforcement — `enforcement.py` (Phase 5A)

**`IdentityEnforcer`** — Detector de drift de identidad por observación. Módulo completamente aislado (sin dependencias a orchestrator, cognition, o events).

**`evaluate_similarity(similarity_score: float) → Dict`**:
- `{"status": "no_baseline"}` — sin baseline embedding o similarity < 0
- `{"status": "aligned", "similarity": ..., "threshold": ...}` — similarity ≥ threshold
- `{"status": "drift_detected", "similarity": ..., "threshold": ..., "severity": ...}` — similarity < threshold

**Integración en Orchestrator (paso 7c, post-evaluación)**:
1. Lee `align_report.identity_similarity` del paso 7b
2. Instancia `IdentityEnforcer(profile)` con el perfil activo de `IdentityManager`
3. Evalúa drift → agrega resultado a `evaluation_results["identity_enforcement"]`
4. Emite evento `identity.drift_checked` via EventBus
5. Pasa `drift_result` a `save_all_evaluations()` para persistencia Postgres (eval_type `identity_enforcement`)

**Garantías**: NO bloquea, NO reintenta, NO altera la respuesta. Es señal de observabilidad pura.

### 21.8 Policy Engine — `policy.py` (Phase 5B)

**`IdentityPolicyEngine`** — Capa reactiva de control de identidad. Completamente desacoplada de IdentityProfile y IdentityEnforcer — recibe solo valores escalares (similarity, threshold) + config de governance.

**`evaluate_action(similarity, threshold, governance_config) → Dict`**:
- Clasifica severidad: `none` (sim ≥ threshold), `low` (gap < 0.05), `medium` (gap < 0.15), `high` (gap ≥ 0.15)
- Mapea severidad → acción via `governance.yaml → identity_control`: `on_low`, `on_medium`, `on_high`
- Acciones válidas: `none | log | flag | rewrite_request | block`
- Resultado: `{action, severity, reason, similarity, threshold}`

**Integración en Orchestrator (paso 7d, post-enforcement)**:
1. Lee `identity_control` de `governance_config`
2. Si habilitado y hay drift_result: instancia `IdentityPolicyEngine()`, evalúa
3. Agrega resultado a `evaluation_results["identity_policy"]`
4. Emite evento `identity.policy_evaluated` via EventBus
5. Pasa `policy_result` a `save_all_evaluations()` para persistencia Postgres (eval_type `identity_policy`)

**Configuración** (`governance.yaml → identity_control`):
```yaml
identity_control:
  enabled: true
  on_low: "log"
  on_medium: "flag"
  on_high: "rewrite_request"
```

**Garantías**: Stateless, NO bloquea, NO altera respuestas (acciones son señales advisory).

### 21.9 Feedback Controller — `feedback.py` (Phase 5C)

**`IdentityFeedbackController`** — Capa de feedback controlado de identidad. Completamente stateless, sin parámetros de constructor, sin dependencia a orchestrator/events/cognition. Depende solo de IdentityProfile (read-only).

**`generate_feedback(drift_result, policy_result, identity_profile) → Dict`**:
- Si `policy_result.action != "rewrite_request"` o `drift_result.severity != "high"`: retorna `{feedback_generated: False}`
- Si ambas condiciones: genera hint determinístico basado en `principal_name`, primeros 3 `values`, y `communication_style` (dimensiones ≥ 0.6)
- Resultado: `{feedback_generated, type, hint, severity, similarity, threshold}`

**Integración en Orchestrator (paso 7e, post-policy)**:
1. Verifica `policy_result.action == "rewrite_request"` AND `drift_result.severity == "high"`
2. Obtiene `identity_profile` del `IdentityManager`
3. Genera feedback via `IdentityFeedbackController().generate_feedback()`
4. Si `feedback_generated`: agrega a `evaluation_results["identity_feedback"]`
5. Emite evento `identity.feedback_generated` via EventBus
6. Pasa `feedback_result` a `save_all_evaluations()` para persistencia Postgres (eval_type `identity_feedback`)

**Garantías**: NO re-ejecuta LLM, NO muta respuestas, NO altera scores, NO modifica IdentityProfile. El hint es metadata advisory para corrección futura.

### 21.10 Memory Bridge — `memory_bridge.py` (Phase 6A)

**`IdentityMemoryBridge`** — Capa pre-generación de análisis de afinidad memoria–identidad. Completamente stateless, sin parámetros de constructor, sin dependencia a orchestrator/events/cognition. Depende de IdentityProfile (read-only) y de `cosine_similarity` + `compute_baseline_embedding` de `embedding.py`.

**`analyze_memories(identity_profile, retrieved_memories) → Dict`**:
- Si `identity_profile` no es IdentityProfile o no tiene baseline: retorna `{enabled: False, reason: ...}`
- Si `retrieved_memories` vacío: retorna `{enabled: False, reason: "no_memories"}`
- Para cada memoria: usa embedding pre-computado si existe, o computa on-the-fly via `compute_baseline_embedding(content)`
- Calcula cosine similarity vs baseline embedding del principal
- Resultado: `{enabled, memories_analyzed, memory_scores: [{memory_id, tier, identity_similarity, content_preview}], aggregate: {avg_similarity, max_similarity, min_similarity}}`

**Integración en Orchestrator (paso 2b, post-memory recall, pre-corrections)**:
1. Verifica que hay `recalled_memories` y que `identity_profile.has_baseline`
2. Analiza afinidad via `IdentityMemoryBridge().analyze_memories()`
3. Si `enabled`: agrega a `evaluation_results["identity_memory_affinity"]`
4. Emite evento `identity.memory_affinity_analyzed` via EventBus
5. Pasa `memory_affinity_result` a `save_all_evaluations()` para persistencia Postgres (eval_type `identity_memory_affinity`)

**Garantías**: Estrictamente observacional — NO re-ordena, NO filtra, NO modifica memorias ni prompts. Output es metadata advisory para fases futuras de weighted retrieval.

### 21.11 Context Weighting — `context_weighting.py` (Phase 6B)

**`IdentityContextWeighter`** — Capa de anotación soft de identidad para el contexto del prompt. Completamente stateless, sin parámetros de constructor, sin dependencia a orchestrator/events/cognition. Depende solo de `logging` y typing.

**`apply_annotations(memory_context, memory_affinity_result, context_line_ids=None) → Dict`**:
- Si `memory_affinity_result` no está habilitado o no tiene `memory_scores`: retorna `{annotated: False, reason: ...}`
- Calcula `avg_similarity` de los scores de afinidad
- Para cada línea del `memory_context`: busca match por `memory_id` via `context_line_ids[i]` (refactor Phase 6C — reemplaza matching por substring)
  - Si match encontrado y similarity ≥ avg: prepend `[IDENTITY_ALIGNED]`
  - Si match encontrado y similarity < avg: prepend `[LOW_IDENTITY_ALIGNMENT]`
- Genera `analysis_block` con conteos de líneas alineadas vs low-alignment y mensaje de coherencia
- Resultado: `{annotated, annotated_context, analysis_block, stats: {total_lines, annotated_lines, aligned_count, low_alignment_count, avg_similarity}}`

**Integración en Orchestrator (paso 3b, post-context combination, pre-cognitive mode routing)**:
1. Verifica que hay `memory_affinity_result` con `enabled: True`
2. Aplica anotaciones via `IdentityContextWeighter().apply_annotations()`
3. Reemplaza `memory_context` en `full_context` con la versión anotada
4. Adjunta `analysis_block` al final del `full_context`
5. Agrega resultado a `evaluation_results["identity_context_weighting"]`
6. Emite evento `identity.context_weighted` via EventBus
7. Non-blocking (try/except)

**Garantías**: Estrictamente soft — NO elimina líneas, NO reordena, NO modifica el contenido de las memorias. Solo prepend tags informativos y adjunta bloque de análisis al final.

### 21.12 Decision Modulation — `decision_modulation.py` (Phase 6C)

**`IdentityDecisionModulator`** — Analizador observacional de alineación decisión-identidad. Completamente stateless, sin parámetros de constructor, solo importa `IdentityProfile` del mismo paquete.

**`evaluate_decision_alignment(decision_result, identity_profile) → Dict`**:
- Evalúa 4 factores:
  - **Risk tolerance** (tabla 9 entradas: risk × tolerance → score)
  - **Category–values** (keywords de 5 dominios vs valores del principal)
  - **Autonomy** (nivel de autonomía × delegation_comfort)
  - **Decision style** (approach + expertise matching)
- Composite score = promedio ponderado igualitario de los 4 factores
- Label: `aligned` (>=0.70), `tension` (>=0.40), `misaligned` (<0.40)
- Genera `reasoning` string con scores individuales y explicación
- Siempre retorna `"observational": True`

**Integración en Orchestrator (paso 3c, post-decision engine, pre-planner)**:
1. Obtiene `IdentityProfile` via `get_state().identity_manager`
2. Crea `IdentityDecisionModulator()` y llama `evaluate_decision_alignment()`
3. Almacena resultado en `evaluation_results["identity_decision_alignment"]`
4. Emite evento `identity.decision_alignment_evaluated` via EventBus
5. Non-blocking (try/except)
6. Pasa `decision_modulation_result` a `save_all_evaluations()` para persistencia Postgres (eval_type `identity_decision_alignment`)

**Refactor Phase 6C en context_weighting**: Reemplazó matching por substring (`content_preview[:40]`) con matching por `memory_id` via parámetro `context_line_ids`. Se añadió `build_context_with_metadata()` a `MemoryManager` que retorna `(context_str, line_ids)` paralelo. Se añadió `content_hash` (SHA-256) a scores de memory_bridge.

**Garantías**: Estrictamente observacional — NUNCA modifica decisiones, routing, selección de agentes, ni comportamiento de governance. Output es metadata advisory.

### 21.13 Confidence Engine — `confidence.py` (Phase 6D)

**`IdentityConfidenceEngine`** — Capa de modulación de autonomía soft. Completamente stateless, sin parámetros de constructor, sin dependencia de IdentityProfile u orchestrator/events/cognition.

**`compute_confidence(enforcement_result, policy_result, decision_modulation_result, memory_affinity_result) → Dict`**:
- Extrae 4 señales (todas opcionales):
  - **Similarity** (enforcement): score cosine identity (w=0.30)
  - **Memory affinity** (memory_bridge): avg_similarity agregada (w=0.20)
  - **Decision alignment** (decision_modulation): identity_decision_alignment (w=0.25)
  - **Policy severity** (policy): none→1.0, low→0.70, medium→0.40, high→0.10 (w=0.25)
- Weighted average de señales disponibles, normalizado por pesos presentes
- Cuando no hay señales → default neutral 0.50 (medium, modifier 0)
- Confidence levels: high (>=0.75, +1), medium (0.50–0.74, 0), low (<0.50, -1)
- Primer componente no-observacional (`observational: False`) — produce `autonomy_modifier` (advisory, no enacted)
- Output incluye `signal_contributions` y `signals_available` para transparencia

**Integración en Orchestrator (paso 3d, post-decision modulation, pre-planner)**:
1. Crea `IdentityConfidenceEngine()` y llama `compute_confidence()`
2. En paso 3d solo `decision_modulation_result` está disponible (enforcement, policy, memory affinity se computan después)
3. Almacena resultado en `evaluation_results["identity_confidence"]`
4. Emite evento `identity.confidence_computed` via EventBus
5. Non-blocking (try/except)
6. Pasa `confidence_result` a `save_all_evaluations()` para persistencia Postgres (eval_type `identity_confidence`)

**Garantías**: Produce `autonomy_modifier` como metadata advisory — NUNCA modifica `DecisionEngine.evaluate()`, `Planner.build()`, routing, ni generación.

### 21.14 Autonomy Modulation — `autonomy_modulation.py` (Phase 7A)

**`IdentityAutonomyModulator`** — Soft governance coupling. Completamente stateless, sin parámetros de constructor, sin dependencia de IdentityProfile, orchestrator, cognition, events, o governance.

**`compute_adjusted_threshold(confidence_result, current_threshold) → Dict`**:
- Recibe output de `IdentityConfidenceEngine.compute_confidence()` + threshold base (default 0.5)
- Deltas por nivel:
  - `low` → -0.10 (governance más estricta, threshold baja)
  - `medium` → 0.0 (sin cambio)
  - `high` → +0.05 (governance ligeramente relajada, threshold sube)
- Threshold clamped a [0.0, 1.0] via `_MIN_THRESHOLD` / `_MAX_THRESHOLD`
- Guards: None, non-dict, missing level → no adjustment (autonomy_adjusted=False)
- Deterministic — identical inputs always produce identical outputs
- Output: `{adjusted_threshold, adjustment_delta, autonomy_adjusted, reason, confidence_level, confidence_score, original_threshold}`

**Integración en Orchestrator (paso 3e, post-confidence, pre-planner)**:
1. Crea `IdentityAutonomyModulator()` y llama `compute_adjusted_threshold(confidence_result, 0.5)`
2. Almacena resultado en `evaluation_results["identity_autonomy_modulation"]`
3. Solo emite evento `identity.autonomy_adjusted` via EventBus cuando `autonomy_adjusted` es True
4. Inyecta `effective_governance_sensitivity` y `autonomy_adjusted` en trace node de governance_review  (metadata, no cambia comportamiento)
5. Non-blocking (try/except)
6. Pasa `autonomy_modulation_result` a `save_all_evaluations()` para persistencia Postgres (eval_type `identity_autonomy_modulation`, solo si autonomy_adjusted)

**Garantías**: NUNCA modifica `DecisionEngine.evaluate()`, `Planner.build()`, prompt building, routing, ni generación. Solo afecta metadata de governance review.

### 21.15 Retrieval Weighting — `retrieval_weighting.py` (Phase 7B)

**`IdentityRetrievalWeighter`** — Re-ranking de memorias basado en identidad. Completamente stateless, sin parámetros de constructor, sin dependencia de IdentityProfile, orchestrator, cognition, events, o governance.

**`rerank_memories(recalled_memories, memory_affinity_scores) → Dict`**:
- Recibe memorias recuperadas + output de `IdentityMemoryBridge.analyze_memories()`
- Para cada memoria computa: `weighted_score = semantic_similarity * 0.8 + identity_affinity * 0.2`
- `semantic_similarity` derivado de ChromaDB distance: `1.0 / (1.0 + max(0, distance))`
- `identity_affinity` del Phase 6A scores (default 0.5 si falta)
- Sort estable descendente por `weighted_score` (ties preservan orden original)
- Guards: None, empty, disabled, non-dict → passthrough (weighting_applied=False)
- Estrictamente non-destructive: mismos items, mismo count, sin filtrado, sin mutación
- Output: `{reranked_memories, weighting_applied, observational: False, memory_count, average_weighted_score, scoring_details}`

**Integración en Orchestrator (paso 2c, post-memory bridge, pre-corrections)**:
1. Crea `IdentityRetrievalWeighter()` y llama `rerank_memories(recalled_memories, memory_affinity_result)`
2. Si `weighting_applied`, reemplaza `recalled_memories` con lista re-rankeada
3. Almacena resultado en `evaluation_results["identity_retrieval_weighting"]`
4. Emite evento `identity.retrieval_reranked` via EventBus (solo si weighting_applied)
5. Non-blocking (try/except)
6. Pasa `retrieval_weighting_result` a `save_all_evaluations()` para persistencia Postgres (eval_type `identity_retrieval_weighting`, solo si weighting_applied)

**Garantías**: NUNCA filtra, elimina, o trunca memorias. NUNCA modifica contenido. NUNCA altera conteo. Deterministic, stateless, no LLM.

### 21.16 Tests — 802 tests unitarios

- `tests/test_identity_module.py`: 47 tests (Phase 4) — schema, embedding, versioning, manager, integración
- `tests/test_identity_enforcement.py`: 30 tests (Phase 5A) — constructor, no_baseline, aligned, drift_detected, severity, invariantes, persistencia
- `tests/test_identity_policy.py`: 55 tests (Phase 5B) — severity computation, action resolution, evaluate_action, disabled config, statelessness, reason strings, module isolation, edge cases, constants
- `tests/test_identity_feedback.py`: 41 tests (Phase 5C) — no-feedback conditions, feedback generation, hint content/determinism, no-mutation, statelessness, module isolation, edge cases
- `tests/test_identity_memory_bridge.py`: 55 tests (Phase 6A) — basic scoring, disabled paths, edge cases, on-the-fly embedding, determinism, no-mutation, statelessness, aggregate stats, similarity bounds, observational-only guarantee, import isolation, disabled result format, mixed formats, content_hash validation
- `tests/test_identity_context_weighting.py`: 46 tests (Phase 6B+6C refactor) — basic annotation, no mutation, passthrough, analysis block, determinism, statelessness, import isolation, edge cases, tag format, ID-based matching, no substring dependency
- `tests/test_identity_decision_modulation.py`: 72 tests (Phase 6C) — result structure, observational flag, label thresholds, risk alignment, category-values, autonomy, decision style, determinism, no mutation, statelessness, invalid inputs, reasoning, import isolation, integration wiring, comprehensive scenarios
- `tests/test_identity_confidence.py`: 75 tests (Phase 6D) — import isolation, class structure, no-input defaults, similarity extraction, memory affinity extraction, decision alignment extraction, policy severity extraction, confidence levels, autonomy modifiers, weighted average, all signals combined, partial signals, edge cases, determinism, no mutations
- `tests/test_identity_autonomy_modulation.py`: 59 tests (Phase 7A) — import isolation, class structure, no-confidence data guards, low/medium/high adjustments, boundary conditions (clamping), autonomy_adjusted flag, reason strings, determinism, no mutations, statelessness, integration wiring, persistence acceptance
- `tests/test_identity_retrieval_weighting.py`: 56 tests (Phase 7B) — import isolation, class structure, no-affinity fallback, basic reranking, scoring formula (80/20 weights), tie handling (stable sort), edge cases (empty/single/missing fields), default affinity, distance conversion, determinism, no mutations, statelessness, integration wiring, persistence acceptance
- `tests/test_identity_consolidation_weighting.py`: 86 tests (Phase 7C) — basic functionality, return structure, confidence signal thresholds (high/mid/low/boundary), alignment signal thresholds (aligned/mid/misaligned/boundary), combined signals, clamping behavior, memory metadata handling, missing/malformed inputs, determinism, no mutation, import isolation, integration wiring, factor component details, signals available count
- `tests/test_identity_behavioral_bias.py`: 96 tests (Phase 8A) — constructor/class structure, return structure, confidence scenarios (low/mid/high/boundary), alignment scenarios, tone_weight mapping (positive/negative/mixed/threshold), assertiveness (assertive/cautious/mixed values + approach keywords), creativity_bias (creative/analytical keywords + confidence proxy), depth_bias (confidence-based + alignment adjustments), reasoning directive, planner mode already-set guards, signals count, determinism, no mutation, edge cases (malformed inputs, dict profiles, report fallback), import isolation, integration wiring, alignment report fallback
- `tests/test_identity_prompt_integration.py`: 72 tests (Phase 8B) — constructor/statelessness, None returns (9 guards), block format (header/footer/4 style values/planner modes/line order), inject_into_prompt (prepend/passthrough/None handling), build_result (metadata structure), determinism, no mutation, edge cases (string/None fallback, negative/zero/one values), no duplicate injection, import isolation (no orchestrator/cognition/events/evaluation/planner/identity_schema imports), integration wiring (exports/__init__/orchestrator/persistence), no response modification, end-to-end flows
- `tests/test_identity_health_monitor.py`: 93 tests (Phase 9A) — constructor/statelessness, empty window (None/empty list/non-list), partial/missing signals, all-stable scenario, high drift rate, sustained low confidence (threshold/boundary/insufficient window), instability index (formula/weights/components), clamping (0-1 bounds), health classification (stable/monitor/unstable/critical/boundaries), policy severity rate (high/critical/mixed/case-insensitive), determinism, no mutation, import isolation (no orchestrator/cognition/events/evaluation/planner imports), integration wiring (exports/__init__/orchestrator/persistence/get_recent_identity_signals), build_result (analyzed/skipped/None), window size handling, edge cases (non-dict interactions/string floats/negative values), end-to-end flows
- `tests/test_identity_health_regulation.py`: 119 tests (Phase 9B) — regulate function, threshold adjustments (stable/monitor/unstable/critical), identity weight adjustments, clamping bounds, guard clauses (None/invalid), regulation_applied flag, monitoring_intensity levels, build_result format, determinism, no mutation, import isolation, integration wiring
- `tests/test_identity_evolution.py`: 97 tests (Phase 10A) — insufficient data (None/empty/too few/non-list), positive evolution (candidate/version/shift/embedding/risk), negative scenarios (low confidence/no sustained/high drift/negative trend/stable identity), high instability rejection (>0.60), high severity rejection (>0.20 high/critical rate), version bump correctness (minor/major/none), shift magnitude calculation (cosine similarity), linear trend computation (OLS slope), centroid embedding (last 30, normalization), determinism (same input → same output), no mutation (profile/signals unchanged), import isolation (no orchestrator/cognition/events/router/agents/evaluation), edge cases (missing keys/None values/non-dict entries/window size 0/-1/empty baseline/mixed severities), build_result (strips embedding/preserves fields/None handling), result structure completeness (all keys present), risk level computation (low/medium/high), sustained checks (confidence window/shift delta), export validation (__init__/__all__)
- `tests/test_identity_versioning.py`: 120 tests (Phase 10C) — snapshot creation (UUID/timestamp/hash/profile_data/evolution_source), snapshot immutability (deep copy verification, hash stability), deterministic hash (same profile → same hash, different profiles → different hash), version ordering (timestamp descending, filtering), get_version (by UUID lookup), diff_versions (structural diff: big_five deltas, values Jaccard, communication_style, boundaries, embedding_changed, hash_match/identical detection), apply candidate isolation (profile_candidate construction, hash verification, no runtime mutation), rollback correctness (operation tag, target_version_id, same guarantees as apply), no-mutation guarantees (original profile/snapshot unchanged after all operations), import isolation (no orchestrator/cognition/events/router/agents imports), build_version_candidate (evolution/shadow guards, metadata capture, snapshot construction), build_apply_result (human confirmation, hash match, ready flag), build_result (strips snapshot/profile_data, preserves scalars, governance defaults), verify_integrity (SHA-256 recompute, tamper detection), prepare_store (human confirmation guard, ready_to_persist flag), edge cases (empty embedding, no values/boundaries, minimal fields, non-dict inputs), determinism (same input → same output)
- `tests/test_identity_governance.py`: 71 tests (Phase 10D) — versions endpoint (empty/list/limit/detail found/not found), snapshot creation (from profile/stores correctly/unique IDs/no profile marker/preserves data), snapshot immutability (hash consistency/integrity check/tamper detection), activation flow (requires confirmation/no auto-apply/candidate return/rollback candidate), evolution endpoint (empty/approve creates snapshot/approve has version_id), evolution reject (non-mutating/returns interaction_id), shadow endpoint (empty/result structure/skipped result), shadow comparison (structural diff present/empty diff), health endpoint (empty signals/structure/with signals/latest summary/no persistence), no-mutation guarantees (7 tests), response structure (8 tests), edge cases (8 tests), snapshot metadata (tags/label/notes storage, backward compatible, custom version, metadata does not affect hash, multiple tags, empty metadata), version validation (valid/invalid vX.Y.Z formats, auto-assign when empty), snapshot persistence metadata (store with metadata, retrieve with metadata, versions list includes metadata)

---

## 22. QUÉ SOBREVIVE UN REINICIO

| Dato | Almacenamiento | En RAM | En Postgres | Persiste |
|------|---------------|--------|-------------|----------|
| Correcciones de training | SQLite (`procedural.db`) | — | — | Sí |
| Vectores episodic/semantic | ChromaDB (disco) | — | — | Sí |
| Conversaciones + mensajes | Neon Postgres | — | Sí | Sí |
| Audit log (eventos) | Neon Postgres | — | Sí | Sí |
| Service logs | Archivos rotativos (5×2MB) | — | — | Sí |
| Working memory (buffer) | RAM (OrderedDict per conversation) | max 20/conv, max 50 conv, LRU | — | No — se pierde |
| **Traces cognitivos** | **RAM + Postgres** | 100 | Ilimitado | **Sí** |
| **Evaluaciones** | **RAM + Postgres** | 200 | Ilimitado | **Sí** |
| **Decisiones detectadas** | **RAM + Postgres** | 500 | Ilimitado | **Sí** |
| **Memory operations** | **RAM + Postgres** | 1000 | Ilimitado | **Sí** |
| **Token usage** | **RAM + Postgres** | Unbounded | Ilimitado | **Sí** |
| **Identity profile** | **RAM + Postgres** | 1 activo | config_versions | **Sí** (Phase 4) |

> **Principio**: Las escrituras a Postgres son **fire-and-forget** en paralelo al almacenamiento in-memory. Si Postgres falla, el sistema sigue funcionando idénticamente — solo pierde persistencia a largo plazo.

---

## 23. OBSERVACIONES Y PROBLEMAS CONOCIDOS

1. **Identity fidelity tiene componente embedding + enforcement + policy + feedback + memory bridge + context weighting** — Phase 4 añadió `identity_similarity` en AlignmentEvaluator usando cosine similarity. Phase 5A activa esta señal via `IdentityEnforcer`: cuando similarity < drift_threshold, se emite `identity.drift_checked` con status `drift_detected` y severity. Phase 5B agrega `IdentityPolicyEngine`: clasifica severidad (none/low/medium/high), mapea a acciones configurables via `governance.yaml → identity_control`, emite `identity.policy_evaluated`. Phase 5C agrega `IdentityFeedbackController`: genera hints determinísticos de corrección cuando action == rewrite_request AND severity == high, emite `identity.feedback_generated`, persiste como `identity_feedback`. Phase 6A agrega `IdentityMemoryBridge`: analiza afinidad memoria–identidad pre-generación (paso 2b del orchestrator), cosine similarity por memoria vs baseline, emite `identity.memory_affinity_analyzed`, persiste como `identity_memory_affinity`. Phase 6B agrega `IdentityContextWeighter`: anota líneas de memoria en el prompt con `[IDENTITY_ALIGNED]` o `[LOW_IDENTITY_ALIGNMENT]` basado en scores de Phase 6A (paso 3b del orchestrator), emite `identity.context_weighted`, almacena como `identity_context_weighting`. Todo es **advisory** — no bloquea ni altera respuestas.

2. **Governance review es automática** — no hay human-in-the-loop real. El GovernanceAgent revisa via LLM pero **auto-approves** si el JSON no parsea. No hay mecanismo de pausa para esperar decisión del principal.

3. **MULTI_AGENT no implementado** — la categoría existe en el enum pero el orchestrator enruta al `identity_core` por defecto. No hay llamada paralela a múltiples agentes ni agregación de resultados.

4. **psycopg2 síncrono** — las llamadas a Postgres son síncronas con autocommit dentro de async handlers. No hay pool de conexiones ni asyncpg. Funciona porque las queries son rápidas y poco frecuentes.

5. **`/system-status` es pesado** — hace queries síncronas a governance.yaml, Postgres (5 queries), ChromaDB, TrainingManager, ModelRouter, SkillRegistry, EventBus todo en secuencia.

6. **WebSocket puede bypasear el orchestrator** — el endpoint `/ws` procesa mensajes de chat directamente a través de `identity_core.respond()`, saltando los 10 pasos del pipeline (sin evaluación, sin trace, sin governance).

7. **ChromaDB embeddings son default** — usa `all-MiniLM-L6-v2` (384 dimensiones) que es bueno pero no óptimo para matching de identidad. Un embedding local dedicado (ej: `mxbai-embed-large` via Ollama) podría mejorar el recall.

8. **Sin autenticación** — la API está completamente abierta. CORS solo permite localhost:3000. Supabase Auth está planificado pero no conectado.

9. **Classification puramente heurística** — usa keyword matching (≥2 matches) sin análisis de complejidad, sentimiento, o riesgo del mensaje. Funciona bien para la mayoría de casos pero puede misclasificar mensajes ambiguos.

10. **Memory consolidation ausente** — no hay proceso background que resuma clusters episódicos en abstracciones semánticas. La memoria episódica crece indefinidamente sin compactación.

---

## 24. ESTADO ACTUAL VS PLANIFICADO

### Completado (Phase 1 + 2 + 3 + 3.5 + 4 + 5A-C + 6A-D + 7A-C + 8A-B + 9A-B + 10A-D.1)
- Full crew de 5 agentes con orchestrator (pipeline de 10+ pasos)
- Sistema de memoria de 4 niveles (ChromaDB + SQLite)
- Model Router con 3 proveedores + cadena de fallback + conteo de tokens real
- 13 páginas de dashboard (todas funcionales)
- 5 módulos de evaluación heurística (quality, alignment, legal risk, decisions, rollback)
- Cognitive trace con visualización React Flow (13 tipos de nodo)
- Governance console (config viewer, audit log, approval queue, emergency stop/resume)
- Analytics dashboard (fidelity, autonomy, tokens, events)
- Testing playground (chat sim, scenario theater, A/B compare)
- Training system (3 modos: correction + free conversation con extracción de rasgos + guided interview con 15 preguntas + upload de writing samples)
- WebSocket real-time updates + Event Bus
- Service logger con crash reports
- Postgres persistence layer (interactions, traces, evaluations, token usage, memory ops)
- Cognition layer obligatoria (DecisionEngine inmutable + Planner stateless)
- Phase 3 architectural hardening (sin ruta legacy, 49 tests de cognición, TaskCategory extraído)
- **Learn-topic skill** — pipeline web search → LLM summarize → chunk → ChromaDB, activable desde chat ("aprende sobre X") y UI Skills, 261 líneas
- **3 Modos Cognitivos** — selector cíclico en chat para nivel de inteligencia (Full/Memory+LLM/Memory Only), Knowledge Boundary + Knowledge Status Header en system prompt (Modo 2), filtro de memoria semántica (Modos 2-3), retorno directo de memoria sin LLM (Modo 3), indicadores de fuente (🧠/🌐) y modo por mensaje
- **Identity Core (Phase 4)** — módulo `src/identity/` con IdentityProfile versionado (Pydantic), baseline embedding de 384 dimensiones (all-MiniLM-L6-v2), semantic versioning con persistencia en config_versions, SHA-256 change detection, identity_similarity en AlignmentEvaluator via cosine similarity, inyección read-only en DecisionEngine, 47 unit tests
- **Identity Memory Bridge (Phase 6A)** — `src/identity/memory_bridge.py` con `IdentityMemoryBridge`: capa pre-generación que analiza afinidad entre memorias recuperadas y baseline embedding del principal. Cosine similarity por memoria + agregados avg/max/min. Estrictamente observacional (no modifica, filtra ni re-ordena). Orchestrator paso 2b (post-recall, pre-corrections). Persistencia como eval_type `identity_memory_affinity`. 53 unit tests
- **Identity-Aware Context Weighting (Phase 6B)** — `src/identity/context_weighting.py` con `IdentityContextWeighter`: anota líneas de memoria en el prompt con `[IDENTITY_ALIGNED]` o `[LOW_IDENTITY_ALIGNMENT]` basado en scores de Phase 6A. Matching por `memory_id` via `context_line_ids` (refactor Phase 6C). Adjunta bloque "Identity Context Analysis". Estrictamente soft (no elimina, no reordena, no modifica contenido). Orchestrator paso 3b. 46 unit tests
- **Identity-Aware Decision Modulation (Phase 6C)** — `src/identity/decision_modulation.py` con `IdentityDecisionModulator`: evalúa alineación decisión–identidad en 4 factores (risk tolerance, category–values, autonomía, decision style). Orchestrator paso 3c (post-decision engine, pre-planner). Persistido como `identity_decision_alignment`. Estrictamente observacional — nunca modifica decisiones. `build_context_with_metadata()` añadido a MemoryManager. `content_hash` (SHA-256) añadido a memory_bridge. 72 unit tests
- **Identity Confidence Engine (Phase 6D)** — `src/identity/confidence.py` con `IdentityConfidenceEngine`: agrega señales de identidad (enforcement similarity, policy severity, decision alignment, memory affinity) en confidence score ponderado (0.0–1.0) + autonomy_modifier (+1/0/-1). Primer componente no-observacional. Degradación graceful para inputs faltantes. Orchestrator paso 3d (post-decision modulation, pre-planner). Persistido como `identity_confidence`. 75 unit tests
- **Autonomy Sensitivity Integration (Phase 7A)** — `src/identity/autonomy_modulation.py` con `IdentityAutonomyModulator`: soft governance coupling que ajusta governance threshold basado en identity confidence level. Low → -0.10 (más estricto), medium → 0.0, high → +0.05 (relajado). Threshold clamped [0.0, 1.0]. Orchestrator paso 3e (post-confidence, pre-planner). Solo emite evento cuando hay ajuste. Metadata inyectada en governance_review trace. Persistido como `identity_autonomy_modulation`. 59 unit tests
- **Identity-Weighted Memory Retrieval (Phase 7B)** — `src/identity/retrieval_weighting.py` con `IdentityRetrievalWeighter`: re-ranking de memorias usando `weighted_score = semantic_similarity * 0.8 + identity_affinity * 0.2`. Sort estable descendente. Si affinity no disponible, preserva orden original. Estrictamente non-destructive (mismos items, sin filtrado, sin mutación). Orchestrator paso 2c (post-memory bridge, pre-corrections). Persistido como `identity_retrieval_weighting`. 56 unit tests
- **Identity-Weighted Memory Consolidation (Phase 7C)** — `src/identity/consolidation_weighting.py` con `IdentityConsolidationWeighter`: ajusta importancia de memoria pre-storage usando confidence (Phase 6D, ±0.10) y decision alignment (Phase 6C, ±0.05). Factor clamped [0.75, 1.25]. Non-blocking, no previene storage, no elimina ni muta contenido. Orchestrator paso 6d (post-governance, pre-memory store). Persistido como `identity_consolidation_weighting`. 86 unit tests
- **Identity Behavioral Bias (Phase 8A)** — `src/identity/behavioral_bias.py` con `IdentityBehavioralBias`: soft identity-guided planning bias que deriva `recommended_planner_mode` (conservative/deep/none) y `style_bias` (tone_weight, assertiveness, depth_bias, creativity_bias) desde señales de identidad (confidence, alignment, communication_style, values, decision_making). Advisory-only — no modifica DecisionEngine, governance ni evaluaciones. Orchestrator paso 3f (post-autonomy modulation, pre-planner). Persistido como `identity_behavioral_bias`. 96 unit tests
- **Soft Identity Prompt Integration (Phase 8B)** — `src/identity/prompt_integration.py` con `IdentityPromptIntegrator`: renderiza metadata de Phase 8A (`style_bias` + `recommended_planner_mode`) como bloque de texto determinístico `[IDENTITY STYLE PREFERENCES]` y lo inyecta (prepend con `\n\n`) en el system prompt. Guards: None/not dict/observational/bias_not_applied/no style_bias → skip (retorna None). 4 valores de estilo (tone_weight, assertiveness, depth_bias, creativity_bias) con defaults 0.50. Orchestrator paso 3g (post-cognitive mode routing, pre-prompt build trace). Emite `identity.prompt_injected`. Persistido como `identity_prompt_integration`. No modifica DecisionEngine, Planner, governance, evaluaciones, memoria ni respuesta. 72 unit tests
- **Identity Longitudinal Monitoring (Phase 9A)** — `src/identity/health_monitor.py` con `IdentityHealthMonitor`: capa observacional post-persistencia que agrega señales de identidad longitudinales sobre ventana deslizante de últimas N interacciones. Métricas: avg_identity_similarity, avg_confidence_score, drift_rate, high_severity_policy_rate, sustained_low_confidence, instability_index (0-1 compuesto). Clasificación: stable (<0.25) / monitor (<0.50) / unstable (<0.70) / critical (≥0.70). Lee señales históricas via `get_recent_identity_signals()` en persistence. Orchestrator paso 9a (post-persistence, fuera del pipeline principal — no afecta interacción actual). Emite `identity.health_evaluated`. Persistido como `identity_health_monitor`. Estrictamente observacional — no modifica IdentityProfile, DecisionEngine, Planner, retrieval, prompt, governance thresholds ni bloquea ejecución. 93 unit tests
- **Health-Aware Adaptive Regulation (Phase 9B)** — `src/identity/health_regulation.py` con `IdentityHealthRegulator`: capa de meta-control adaptativa que reacciona a señales de salud de Phase 9A. Ajusta governance threshold (stable: 0, monitor: -0.05, unstable: -0.10, critical: -0.15) e identity weight (stable: 0, monitor: +0.05, unstable: +0.10, critical: +0.15). Clamp threshold [0.0, 1.0], identity_weight [0.0, 0.5]. Orchestrator paso 9b (post-health monitor, pre-return). Emite `identity.health_regulated` (solo cuando regulation_applied). Persistido como `identity_health_regulation`. Determinístico, stateless, metadata-only — nunca modifica identidad, decisiones, routing, LLM outputs, prompt, governance actual ni interacción actual. 119 unit tests
- **Dynamic Identity Evolution Engine (Phase 10A)** — `src/identity/evolution.py` con `IdentityEvolutionEngine`: motor de evolución dinámica de identidad, proposal-only y governance-gated. Analiza trayectoria de identidad a largo plazo (ventana de 200 interacciones): similarity_trend + confidence_trend (regresión lineal OLS), sustained_high_confidence (últimas 10 > 0.75), sustained_similarity_shift (últimas 20 difieren > 0.08 del baseline), drift_rate, avg_instability, high_severity_rate. Criterios de evolución: sustained_high_conf AND (trend > 0 OR sustained_shift) AND drift_rate < 0.25 AND avg_confidence > 0.70. Rechazo: instability mean > 0.60 OR high_severity_rate > 0.20. Candidato: computa centroid embedding (últimas 30 response_embedding, normalizado), shift_magnitude = 1.0 - cosine_similarity(baseline, centroid), version bump (< 0.05 → no, 0.05-0.10 → minor, > 0.10 → major). Risk level (low/medium/high). Siempre `requires_human_approval: True`, `observational: True`. Orchestrator paso 9c (post-health regulation, pre-return). Emite `identity.evolution_analyzed`. Persistido como `identity_evolution_analysis`. Stateless, determinístico, sin LLM — nunca modifica IdentityProfile, baseline_embedding, decisiones, routing, governance ni estado del sistema. 97 unit tests
- **Shadow Identity Simulation Layer (Phase 10B)** — `src/identity/shadow_simulation.py` con `IdentityShadowSimulator`: capa de simulación de evolución no-mutante. Toma propuesta de Phase 10A, construye clon in-memory (deep copy) del IdentityProfile, aplica cambios propuestos solo al shadow, computa señales comparativas (similarity_shift, confidence_shift, drift_rate_change, instability_delta). Risk score ponderado (instability 0.30 + drift 0.25 + similarity 0.25 + confidence 0.20), clamped [0,1]. Risk grade: safe (<0.25) / cautious (<0.50) / risky (<0.70) / destabilizing (≥0.70). Produce structural diff (version, embedding, big_five, values, communication_style, boundaries, drift_threshold). NUNCA muta el IdentityProfile real, NUNCA auto-aplica cambios, NUNCA modifica runtime. Orchestrator paso 9d (post-evolution analysis, pre-persistence). Emite `identity.shadow_simulated`. Persistido como `identity_shadow_simulation`. `observational: True`, `requires_human_approval: True` siempre. Stateless, determinístico, sin LLM, sin IO, sin DB writes. Completa el closed cognitive safety loop: observabilidad → regulación → propuesta de evolución → simulación antes del cambio. 120 unit tests
- **Identity Versioning & Controlled Apply (Phase 10C)** — `src/identity/version_control.py` con `IdentityVersionControl`: sistema de versionado inmutable de identidad con apply controlado y rollback seguro. `create_snapshot()` genera snapshot inmutable con UUID version_id, ISO timestamp, SHA-256 content_hash, deep-copied profile_data y evolution_source. `diff_versions()` produce diff estructural completo (big_five deltas, values Jaccard, communication_style, boundaries, embedding_changed, hash_match/identical detection). `apply_version()` y `rollback_to()` retornan candidatos de IdentityProfile para revisión humana — NUNCA auto-aplican. `build_version_candidate()` construye candidato desde datos de evolution (Phase 10A) + shadow (Phase 10B), con guards: evolution_candidate True, shadow risk_grade in (safe, cautious), current_identity presente. `verify_snapshot_integrity()` recomputa SHA-256 para detección de tampering. `build_result()` genera output compacto para persistencia (strip snapshot/profile_data). `prepare_store()` prepara snapshot para almacenamiento con human confirmation guard. Orchestrator paso 9e (post-shadow simulation, pre-persistence). Emite `identity.version_candidate_created`. Persistido como `identity_version_candidate`. CERO mutación automática, CERO auto-apply, CERO modificación de runtime. `observational: True`, `requires_human_approval: True` siempre. Stateless, determinístico, 710 líneas, sin LLM, sin IO, import-isolated (solo IdentityProfile de schema). 120 unit tests
- **Identity Governance Interface (Phase 10D)** — Integración no-intrusiva en el dashboard para gestionar versiones de identidad, propuestas de evolución, simulaciones shadow y monitoreo de salud. Nueva ruta `/identity-governance` con 4 tabs: Versions (snapshot modal con version/tags/label/notes, status badges Active/Candidate/Historical, activation dialog), Evolution (approve/reject candidates con risk badges), Shadow Simulation (risk grades, structural diff, tablas comparativas), Identity Health (signals timeline con CSS bar charts, health classification, pre-training baseline badge). 11 endpoints backend. Navegación: sidebar link, Command Center quick action, Identity Studio "View Versions" link. No modifica páginas, rutas ni componentes existentes. 55 unit tests
- **Governance Hardening (Phase 10D.1)** — Parche de hardening dirigido por auditoría que corrige 5 defectos (D1–D5): D1 (CRITICAL): reject endpoint arreglado — reemplazó constructor `EventBus()` roto + kwargs emit con singleton `event_bus` + `await event_bus.emit(IAmeEvent(...))`. D2 (MEDIUM): approve endpoint ahora emite evento de auditoría `identity.evolution_approved`. D3 (LOW): activate event incluye `destructive: False` (simetría con rollback `destructive: True`). D4 (LOW): DELETE `/identity/versions/{id}` ahora protege contra eliminación de versión activa (retorna 409). D5 (LOW): activate endpoint retorna early con `already_active: True` para re-activación idempotente. Scope: solo `routes.py` parcheado — sin cambios en orchestrator, evolution engine, version control, snapshot, health ni shadow. 20 unit tests en `test_identity_governance_hardening.py` (5 clases de test)

### Planificado (Phase 11+)
| Item | Prioridad | Descripción |
|------|-----------|-------------|
| **Human-in-the-Loop** | ALTA | Mecanismo de pausa para approval queue de governance |
| **Supabase Auth** | ALTA | JWT + login/logout + rutas protegidas |
| **Real Identity Fidelity** | MEDIA | Baseline embedding implementado (Phase 4). Falta: ponderar identity_similarity en overall_score, dashboard gauge, drift alertas |
| **Document Chunking Pipeline** | ALTA | Chunking inteligente (500-1000 tokens con overlap) |
| **Memory Consolidation** | ALTA | Job background para resumir episodic → semantic |
| **Multi-Agent Collaboration** | MEDIA | Llamadas paralelas a agentes + agregación |
| **Identity Drift Detection** | MEDIA | Embeddings baseline + alerta de drift |
| **Autonomous Skill Acquisition** | MEDIA | Learning Agent pipeline avanzado (learn-topic básico ya implementado, falta "Teach Me" UI y adquisición autónoma) |
| **External Integrations** | MEDIA | Email, calendar, Slack (Discord bot ya implementado — ver sección 26) |
| **QLoRA Fine-Tuning** | BAJA | PEFT + Unsloth para modelo privado |
| **Self-Modification System** | BAJA | Acceso al codebase con governance |

---

## 25. ESTRUCTURA DE ARCHIVOS COMPLETA

```
iame.lol/
├── agent/                                    # Backend Python FastAPI
│   ├── src/
│   │   ├── agents/                           # 5 agentes especializados
│   │   │   ├── base_agent.py         (91 ln) # ABC para agentes de dominio
│   │   │   ├── identity_core.py     (255 ln) # Guardián de identidad (standalone, 3 cognitive modes)
│   │   │   ├── business_agent.py     (50 ln) # Estratega de negocio
│   │   │   ├── communication_agent.py(56 ln) # Especialista en comunicación
│   │   │   ├── technical_agent.py    (44 ln) # Constructor técnico
│   │   │   ├── governance_agent.py  (132 ln) # Meta-agente de cumplimiento
│   │   │   └── crew.py             (111 ln) # Inicialización y gestión del crew
│   │   ├── api/
│   │   │   ├── main.py             (324 ln) # Composition root + AppState + lifespan + Identity wiring
│   │   │   └── routes.py          (3264 ln) # 100 endpoints REST + WebSocket
│   │   ├── cognition/                        # Capa cognitiva OBLIGATORIA
│   │   │   ├── __init__.py          (20 ln) # Re-exports: DecisionEngine, Planner, TaskCategory
│   │   │   ├── decision_engine.py  (150 ln) # Motor de decisión inmutable (+identity_profile, Phase 4)
│   │   │   └── planner.py         (118 ln) # Planificador stateless
│   │   ├── db/
│   │   │   ├── database.py        (289 ln) # Postgres connection + 10 tablas
│   │   │   └── persistence.py    (1308 ln) # Fire-and-forget persistence repository (25 public + 6 private methods)
│   │   ├── evaluation/                       # 5 módulos heurísticos
│   │   │   ├── quality_scorer.py   (383 ln) # Calidad en 5 dimensiones → grade A-F
│   │   │   ├── alignment_evaluator.py(420 ln) # Alineación con persona + identity_similarity (Phase 4)
│   │   │   ├── legal_risk.py      (319 ln) # 15+ regex patterns de riesgo legal
│   │   │   ├── decision_registry.py(358 ln) # Detección de decisiones de negocio
│   │   │   └── memory_rollback.py  (353 ln) # Auditoría + point-in-time recovery
│   │   ├── events/
│   │   │   └── event_bus.py       (128 ln) # Pub/Sub + WS + Audit
│   │   ├── flows/
│   │   │   ├── categories.py       (18 ln) # TaskCategory enum (compartido)
│   │   │   └── orchestrator.py   (2135 ln) # Pipeline de 10+ pasos + 3 modos cognitivos
│   │   ├── identity/                         # Módulo de identidad formal (Phase 4-10D.1, 22 archivos)
│   │   │   ├── __init__.py         (~52 ln) # Re-exports de todos los componentes (19 exports)
│   │   │   ├── schema.py         (~105 ln) # IdentityProfile Pydantic model
│   │   │   ├── embedding.py      (~215 ln) # Identity text + ChromaDB embedding + cosine similarity
│   │   │   ├── versioning.py     (~215 ln) # Semantic versioning + SHA-256 + Postgres persistence
│   │   │   ├── enforcement.py     (~90 ln) # Phase 5A: Soft drift detection
│   │   │   ├── policy.py         (~165 ln) # Phase 5B: Reactive identity control
│   │   │   ├── feedback.py       (~110 ln) # Phase 5C: Controlled identity feedback
│   │   │   ├── memory_bridge.py  (~165 ln) # Phase 6A: Identity-memory affinity analyser
│   │   │   ├── context_weighting.py(~200 ln) # Phase 6B: Soft identity-aware context annotation
│   │   │   ├── decision_modulation.py(~280 ln) # Phase 6C: Decision-identity alignment analyser
│   │   │   ├── confidence.py     (~230 ln) # Phase 6D: Identity confidence engine
│   │   │   ├── autonomy_modulation.py(~175 ln) # Phase 7A: Soft governance coupling
│   │   │   ├── retrieval_weighting.py(~210 ln) # Phase 7B: Identity-weighted memory retrieval
│   │   │   ├── consolidation_weighting.py(~210 ln) # Phase 7C: Identity-weighted memory consolidation
│   │   │   ├── behavioral_bias.py(~500 ln) # Phase 8A: Identity behavioral bias layer
│   │   │   ├── prompt_integration.py(~195 ln) # Phase 8B: Soft identity prompt integration
│   │   │   ├── health_monitor.py (~320 ln) # Phase 9A: Identity longitudinal monitoring
│   │   │   ├── health_regulation.py(~260 ln) # Phase 9B: Health-aware adaptive regulation
│   │   │   ├── evolution.py      (~600 ln) # Phase 10A: Dynamic identity evolution engine
│   │   │   ├── shadow_simulation.py(~528 ln) # Phase 10B: Shadow identity simulation layer
│   │   │   ├── version_control.py(~710 ln) # Phase 10C: Immutable identity versioning + controlled apply + rollback
│   │   │   └── manager.py        (~255 ln) # Singleton lifecycle manager
│   │   ├── memory/
│   │   │   └── manager.py         (~810 ln) # 4-tier unified memory (per-conversation working)
│   │   ├── router/
│   │   │   └── model_router.py    (360 ln) # Gemini→Groq→Ollama fallback
│   │   ├── skills/
│   │   │   ├── registry.py        (116 ln) # Skill toggle + tracking
│   │   │   ├── web_research.py    (106 ln) # Tavily wrapper
│   │   │   ├── learn_topic.py     (261 ln) # Topic learning pipeline
│   │   │   └── tools.py          (110 ln) # Email + Document tools
│   │   ├── trace/
│   │   │   └── collector.py       (342 ln) # Cognitive trace + TraceStore
│   │   ├── training/
│   │   │   └── manager.py         (434 ln) # 3 modos + correcciones + free convo + interview
│   │   ├── config.py               (98 ln) # Pydantic BaseSettings
│   │   ├── service_logger.py      (218 ln) # Rotating file logger
│   │   └── watchdog.py            (111 ln) # Service health watchdog
│   ├── tests/                                # 33 archivos + conftest.py (1728 tests)
│   │   ├── conftest.py            (100 ln) # Fixtures compartidos
│   │   ├── test_cognition.py      (235 ln) # 49 tests puros (Phase 3)
│   │   ├── test_orchestrator.py   (161 ln) # Tests del pipeline
│   │   ├── test_identity_core.py  (180 ln) # Tests del agente principal
│   │   ├── test_memory.py         (213 ln) # Tests del sistema de memoria
│   │   ├── test_training.py       (148 ln) # Tests del training
│   │   ├── test_api.py            (135 ln) # Tests de endpoints
│   │   ├── test_crew.py           (104 ln) # Tests del crew
│   │   ├── test_event_bus.py      (115 ln) # Tests del event bus
│   │   ├── test_model_router.py   (112 ln) # Tests del router
│   │   ├── test_skills.py         (103 ln) # Tests de skills
│   │   ├── test_identity_module.py(310 ln) # Tests del módulo de identidad (Phase 4, 47 tests)
│   │   ├── test_identity_enforcement.py(240 ln) # Tests de enforcement (Phase 5A, 30 tests)
│   │   ├── test_identity_policy.py (~280 ln) # Tests de policy engine (Phase 5B, 55 tests)
│   │   ├── test_identity_feedback.py(~290 ln) # Tests de feedback controller (Phase 5C, 41 tests)
│   │   ├── test_identity_memory_bridge.py(~430 ln) # Tests de memory bridge (Phase 6A, 53 tests)
│   │   ├── test_identity_context_weighting.py(~350 ln) # Tests de context weighting (Phase 6B, 46 tests)
│   │   ├── test_identity_decision_modulation.py(~450 ln) # Tests de decision modulation (Phase 6C, 72 tests)
│   │   ├── test_identity_confidence.py(~400 ln) # Tests de confidence engine (Phase 6D, 75 tests)
│   │   ├── test_identity_autonomy_modulation.py(~350 ln) # Tests de autonomy modulation (Phase 7A, 59 tests)
│   │   ├── test_identity_retrieval_weighting.py(~380 ln) # Tests de retrieval weighting (Phase 7B, 56 tests)
│   │   ├── test_identity_consolidation_weighting.py(~420 ln) # Tests de consolidation (Phase 7C, 86 tests)
│   │   ├── test_identity_behavioral_bias.py(~500 ln) # Tests de behavioral bias (Phase 8A, 96 tests)
│   │   ├── test_identity_prompt_integration.py(~400 ln) # Tests de prompt integration (Phase 8B, 72 tests)
│   │   ├── test_identity_health_monitor.py(~480 ln) # Tests de health monitor (Phase 9A, 93 tests)
│   │   ├── test_identity_health_regulation.py(~520 ln) # Tests de health regulation (Phase 9B, 119 tests)
│   │   ├── test_identity_evolution.py(~500 ln) # Tests de evolution engine (Phase 10A, 97 tests)
│   │   ├── test_identity_shadow_simulation.py(~720 ln) # Tests de shadow simulation (Phase 10B, 120 tests)
│   │   ├── test_identity_versioning.py(~938 ln) # Tests de version control (Phase 10C, 120 tests)
│   │   ├── test_identity_governance.py(~580 ln) # Tests de identity governance (Phase 10D, 55 tests)
│   │   ├── test_identity_governance_hardening.py(~350 ln) # Tests de governance hardening (Phase 10D.1, 20 tests)
│   │   ├── test_config.py          (86 ln) # Tests de configuración
│   │   └── test_basics.py          (59 ln) # Tests básicos de importación
│   └── configs → ../configs                  # Symlink a configs/
├── dashboard/                                # Frontend Next.js 15
│   ├── app/                                  # 13 rutas (App Router)
│   │   ├── page.tsx               (189 ln) # Command Center
│   │   ├── chat/page.tsx           (11 ln) # Chat wrapper
│   │   ├── identity/page.tsx      (417 ln) # Identity Studio
│   │   ├── training/page.tsx      (478 ln) # Training Center
│   │   ├── testing/page.tsx       (366 ln) # Testing Playground
│   │   ├── models/page.tsx        (352 ln) # Model Manager
│   │   ├── skills/page.tsx        (254 ln) # Skill Manager
│   │   ├── memory/page.tsx        (494 ln) # Memory Lab (con edición inline)
│   │   ├── governance/page.tsx    (481 ln) # Governance Console (approval actions funcionales)
│   │   ├── analytics/page.tsx     (269 ln) # Analytics Dashboard
│   │   ├── trace/page.tsx         (716 ln) # Cognitive Trace viewer
│   │   ├── evaluation/page.tsx    (879 ln) # Evaluation Dashboard (Postgres reads + delete)
│   │   └── identity-governance/page.tsx (66 ln) # Identity Governance (4 tabs: Versions/Evolution/Shadow/Health)
│   ├── components/
│   │   ├── command-center/       (1026 ln) # 7 componentes del dashboard principal
│   │   ├── chat/                  (218 ln) # ChatPanel + MessageBubble
│   │   ├── layout/                (266 ln) # Sidebar (4 grupos) + Header + ClientShell
│   │   ├── trace/                 (267 ln) # TraceNode (React Flow custom)
│   │   └── ui/                    (340 ln) # 10 primitivos shadcn/ui (incl. confirm-dialog)
│   └── lib/
│       ├── api.ts                 (1117 ln) # API client (~97 métodos)
│       ├── store.ts               (165 ln) # Zustand global store (17 state fields, persist middleware)
│       ├── hooks/                           # Custom React hooks
│       └── i18n/                            # en.json + es.json
├── configs/                                  # Archivos de configuración
│   ├── persona.yaml                (87 ln) # Identidad del principal
│   ├── models.json                (~80 ln) # Proveedores LLM + asignaciones
│   ├── skills.json                (~30 ln) # Registro de skills
│   └── governance.yaml            (254 ln) # Framework de gobernanza
├── data/                                     # (gitignored) Training data
├── scripts/
│   └── discord_bot.py            (~575 ln) # Discord bot — participante natural (ver sección 26)
├── docs/
│   └── arquitectura.md                       # Este documento
└── Base Guideline.md                         # Estrategia general del proyecto
```

**Total de código backend (Python)**: ~20,472 líneas en `agent/src/` (incluye identity/ ~6080 ln)
**Total de tests**: ~16,400 líneas en 33 archivos (1728 tests, 1728 passing)
**Total de código frontend (TypeScript/TSX)**: ~11,988 líneas en `dashboard/`

---

## 26. DISCORD BOT — INTEGRACIÓN COMO PARTICIPANTE NATURAL

### 26.1 Arquitectura General

```
scripts/discord_bot.py (~575 ln)
├── discord.py v2.6.4             # Discord gateway + intents (messages, members, guilds)
├── httpx v0.28.1                 # HTTP client async → POST /api/chat
├── PID lock file                 # agent/discord_bot.pid (tasklist-based, atexit cleanup)
└── VS Code Task                  # "Discord: Django Bot" (instanceLimit: 1)
```

**Bot**: "Django" (Intents: message_content, guilds, members)
**Dependencias**: discord.py, httpx (ambos en `agent/venv`)
**Token**: `DISCORD_BOT_TOKEN` en `.env`
**Ejecución**: `${workspaceFolder}/agent/venv/Scripts/python.exe ${workspaceFolder}/scripts/discord_bot.py`

### 26.2 Flujo de Interacción Completo

```
Usuario escribe en Discord → on_message()
  │ Track user en known_users (display_name, username, message_count)
  │ Almacenar mensaje en channel_buffers[channel_id] (deque maxlen=30)
  │
  ├─ ¿Comando? (!reset, !who, !full, !memory) → manejar directamente
  │   └─ !reset → POST /memory/working/clear (conversation_id) + limpiar estado local
  │
  ├─ ¿Debe responder? (mención, reply, name trigger, o LLM decide)
  │   ├─ Construir GROUP_CONTEXT_PROMPT con:
  │   │   • [CONTEXTO EN TIEMPO REAL — DISCORD] header
  │   │   • Nombre del servidor, canal, lista de miembros (guild.members)
  │   │   • Últimos 30 mensajes del channel buffer
  │   │   • Override: ignora restricciones de Knowledge Status para datos real-time
  │   │
  │   ├─ call_django_api():
  │   │   • Tag del mensaje como "[user_name] mensaje"
  │   │   • POST /api/chat con {message, conversation_id, cognitive_mode=2, context}
  │   │   • conversation_id = per-channel (almacenado en channel_conversations dict)
  │   │   • Backend crea/reutiliza conversación en Postgres
  │   │   • Orchestrator procesa por pipeline completo de 10+ pasos
  │   │   • Respuesta almacenada en working memory per-conversation (aislada)
  │   │
  │   ├─ ¿Respuesta contiene [SILENT]? → no hacer nada (LLM eligió no responder)
  │   ├─ De lo contrario → simular typing → split en chunks ≤2000 chars → enviar
  │   └─ Cooldown: MIN_RESPONSE_GAP = 8 segundos entre respuestas por canal
  │
  └─ !reset → limpia conversation_id del canal + POST /memory/working/clear
```

### 26.3 Aislamiento de Memoria

Cada canal de Discord obtiene su propio `conversation_id` aislado:

| Fuente | conversation_id | Working Memory |
|--------|----------------|----------------|
| Discord #general | `discord-channel-{id1}` | Instancia independiente |
| Discord #random | `discord-channel-{id2}` | Instancia independiente |
| Dashboard /chat | `dashboard-{uuid}` | Instancia independiente |

- `channel_conversations`: dict local en el bot que mapea `channel_id → conversation_id`
- El backend mantiene `OrderedDict[str, WorkingMemory]` con LRU eviction (max 50 sessions)
- Sesiones inactivas >1h se auto-limpian
- `!reset` limpia tanto el estado local como la working memory del backend

### 26.4 Funcionalidades Clave

| Característica | Implementación |
|---------------|----------------|
| **Per-channel conversations** | Cada canal tiene `conversation_id` propio → working memory aislada |
| **Member awareness** | `guild.chunk()` on startup + `build_member_list()` → inyectado en cada prompt |
| **Real-time context override** | GROUP_CONTEXT_PROMPT override explícito de Knowledge Status |
| **Natural participation** | LLM decide cuándo hablar via token `[SILENT]` |
| **Cognitive mode 2** | Memory+LLM (no web search) para todas las interacciones Discord |
| **Typing simulation** | `async with channel.typing()` durante procesamiento |
| **Rate limiting** | MIN_RESPONSE_GAP = 8s entre respuestas por canal |
| **PID lock** | `agent/discord_bot.pid` — file-based, sin WMIC, atexit cleanup |
| **Comandos** | `!full` (mode 1), `!memory` (mode 3), `!reset`/`!nueva`/`!new`, `!who`/`!quien` |

### 26.5 Anti-Duplicación

- Task command usa `venv/Scripts/python.exe` directamente (evita doble ejecución shim)
- `instanceLimit: 1` en VS Code Task — re-run mata instancia anterior
- PID lock file con `tasklist`-based alive check (sin WMIC)
- PROHIBIDO: `run_in_terminal` con `isBackground: true` para el bot

---

*Última actualización: 2026-02-20 — Discord Bot Integration + Per-Conversation Working Memory. 100 endpoints (+1: POST /memory/working/clear), 1728 tests (+4). Discord bot (`scripts/discord_bot.py`, ~575 ln): participante natural en Discord via discord.py + httpx → POST /api/chat, conversaciones per-channel, member awareness (guild.chunk + build_member_list), contexto en tiempo real con override de Knowledge Status, token [SILENT] para participación selectiva, cognitive mode 2, simulación de typing, cooldown 8s, PID lock file-based. Working memory refactorizado de singleton global a OrderedDict per-conversation_id con LRU eviction (max 50 sessions), auto-cleanup de sesiones inactivas >1h, backward-compatible .working property, nuevo endpoint POST /memory/working/clear. Órdenes: !full, !memory, !reset (ahora limpia working memory del backend), !who.*
*Evaluation Dashboard lee desde Postgres con fallback in-memory. Chat persiste entre refreshes (Zustand persist). Trace IDs únicos por interacción (uuid4). Delete endpoints para trace, evaluation, analytics, training corrections.*
*Preparado para auditoría de especialistas en conciencias virtuales*
