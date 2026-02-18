# IAme — Arquitectura Completa del Sistema

> **Documento de referencia técnica para auditoría de arquitectura de conciencia virtual.**
> Este documento describe con total precisión el estado actual del sistema IAme (Autonomous Digital Delegate System — ADLRA), un sistema multi-agente diseñado para aprender a representar la identidad, personalidad y estilo de toma de decisiones de su principal (humano) a través de entrenamiento progresivo, memoria contextual y retroalimentación humana directa.

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
21. [Qué Sobrevive un Reinicio](#21-qué-sobrevive-un-reinicio)
22. [Observaciones y Problemas Conocidos](#22-observaciones-y-problemas-conocidos)
23. [Estado Actual vs Planificado](#23-estado-actual-vs-planificado)
24. [Estructura de Archivos Completa](#24-estructura-de-archivos-completa)

---

## 1. VISIÓN GENERAL DEL SISTEMA

IAme es un **delegado digital autónomo**: un sistema de IA multi-agente que aprende progresivamente a actuar como su "principal" (el humano que lo entrena). No es un chatbot genérico — es una **identidad virtual personalizada** que:

- **Piensa** con la misma estructura de decisión que su principal (valores, prioridades, tolerancia al riesgo).
- **Habla** con el mismo estilo de comunicación (formalidad, humor, empatía, verbosidad calibrada).
- **Decide** respetando los mismos límites (boundaries hardcodeados, análisis costo-beneficio, prioridades de stakeholders).
- **Aprende** de correcciones directas del humano, acumulando reglas conductuales en memoria procedimental.
- **Se evalúa a sí mismo** con 5 módulos heurísticos que miden calidad, alineación con la persona, riesgo legal, decisiones de negocio y operaciones de memoria — todo sin llamadas LLM adicionales.

**Filosofía de diseño**: El sistema opera en **$0/mes** usando exclusivamente tiers gratuitos (Gemini Free, Groq Free, Neon Free, ChromaDB local, Ollama local). La privacidad es prioridad: datos de identidad nunca salen de la máquina local; solo los outputs de agentes van a LLMs cloud.

**Estado actual**: Phase 3 completada (Architectural Hardening). El sistema está listo para la próxima etapa: entrenamiento real de la conciencia virtual con datos de identidad del principal.

---

## 2. DIAGRAMA DE ALTO NIVEL

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                 DASHBOARD (Next.js 15 / React 19 / Tailwind / shadcn/ui)     │
│                 Puerto 3000 — 12 rutas + WebSocket client                    │
├──────────────────────────┬───────────────────────────────────────────────────┤
│  Zustand Store (119 ln)  │  API Client — lib/api.ts (650 ln, ~60 métodos)   │
│  Estado global del UI    │  Comunicación tipada con el backend               │
├──────────────────────────┴───────────────────────────────────────────────────┤
│                                     ▼ HTTP / WebSocket                       │
├──────────────────────────────────────────────────────────────────────────────┤
│         FastAPI Backend — Puerto 8000 — Prefijo /api — 73 endpoints          │
│                           routes.py (1606 ln) + main.py (249 ln)             │
├──────────┬───────────┬──────────┬───────────┬──────────┬─────────────────────┤
│ COGNICIÓN│ ORQUESTA- │ AGENT    │ MEMORIA   │ EVALUA-  │ TRACE              │
│ Decision │ DOR       │ CREW (5) │ 4 niveles │ CIÓN     │ Collector          │
│ Engine   │ Pipeline  │          │ Manager   │ 5 módulos│ 13 tipos nodo      │
│ (138 ln) │ 10 pasos  │ Identity │ (520 ln)  │ heuríst. │ (342 ln)           │
│ Planner  │ (829 ln)  │ Business │           │ (1796 ln │                    │
│ (118 ln) │           │ Comms    │           │  total)  │                    │
│ Categ.   │           │ Tech     │           │          │                    │
│ (18 ln)  │           │ Govern.  │           │          │                    │
├──────────┴───────────┴──────────┴───────────┴──────────┴─────────────────────┤
│ Model Router (329 ln) │ Skill Registry (116 ln) │ Training Mgr (194 ln)      │
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
| **Frontend** | Next.js + React + TypeScript | 15 / 19 / 5.7 | Dashboard de control con 12 rutas | Puerto 3000, App Router |
| **UI** | shadcn/ui + Tailwind CSS + lucide-react | 3.4 | Componentes primitivos + tema lab oscuro | Variables CSS: lab-text, lab-card, accent-glow |
| **Estado UI** | Zustand | 5 | Store global del cliente | 119 líneas |
| **LLM primario** | Google Gemini 2.5 Flash | — | Proveedor principal (free tier) | Via google-generativeai SDK |
| **LLM secundario** | Groq (Llama 3.3 70B) | — | Fallback #1 (free tier) | Via groq SDK |
| **LLM local** | Ollama (qwen2.5:32b) | — | Fallback #2 + modo privacidad | Siempre disponible localmente |
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
| `configs/models.json` | ~80 | 3 proveedores LLM con modelos específicos, cadena de fallback (gemini → groq → ollama), asignaciones por agente, 4 perfiles (balanced, max_quality, privacy_mode, budget_mode) | JSON |
| `configs/skills.json` | ~55 | Registro de 4 skills (web-research, email-draft, document-gen, learn-topic) con toggle enable/disable, nivel de riesgo por skill, estadísticas de uso | JSON |
| `configs/governance.yaml` | 254 | **Framework de gobernanza completo**: 5 niveles de autonomía (Observer→Trusted), clasificación de riesgo por acción (low/medium/high/critical con ejemplos), operaciones prohibidas, reglas de privacidad, escalamiento, controles de emergencia | YAML |

**Nota para auditores**: `persona.yaml` es el archivo fundamental de identidad. Contiene la "personalidad transferible" del principal — cada campo afecta directamente cómo los agentes generan respuestas. El campo `boundaries` contiene reglas que **nunca** pueden ser violadas por ningún agente, sin importar el contexto.

---

## 5. CAPA DE ENTRADA — API REST + WebSocket

### 5.1 Composition Root — `agent/src/api/main.py` (249 líneas)

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

### 5.2 API Endpoints — `agent/src/api/routes.py` (1606 líneas, 73 endpoints)

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
| **Trace** | `/trace/list`, `/trace/{id}`, `/trace/latest/graph`, `/trace/replay` | GET×3, POST | Trazas cognitivas, grafos React Flow, replay paso a paso |
| **Persistence** | `/interactions`, `/interactions/{id}/trace`, `/interactions/{id}/evaluations`, `/token-usage/persisted` | GET×4 | Datos persistidos en Postgres |
| **Memory** | `/memory/stats`, `/memory/search`, `/memory/semantic/store`, `/memory/semantic/{id}` (PUT, DELETE), `/memory/episodic/{id}` (DELETE) | GET, POST×2, PUT, DELETE×2 | CRUD completo del sistema de memoria |
| **Skills** | `/skills`, `/skills/{id}/toggle`, `/skills/web-research`, `/skills/learn-topic` | GET, POST×3 | Gestión de habilidades + herramienta de aprendizaje |
| **Training** | `/training/status`, `/training/session/start`, `/training/session/end`, `/training/correction`, `/training/history`, `/training/upload-samples` | GET×2, POST×4 | Sesiones de entrenamiento, correcciones, upload de muestras de escritura |
| **Models** | `/models/config`, `/models/assignment`, `/models/profile`, `/models/test` | GET, PUT×2, POST | Gestión de proveedores LLM, perfiles, asignaciones por agente |
| **Evaluation** | `/evaluation/overview`, quality/×3, alignment/×2, legal/×3, decisions/×5, rollback/×5 | GET×14, POST×3, PUT×1 | 19 endpoints para los 5 módulos de evaluación |
| **Governance** | `/governance/config`, `/governance/audit-log`, `/governance/approvals`, `/governance/emergency-stop`, `/governance/emergency-resume`, `/governance/emergency-status` | GET×3, POST×2, GET×1 | Configuración, auditoría, aprobaciones, parada de emergencia |
| **Analytics** | `/analytics/overview`, `/analytics/identity-fidelity`, `/analytics/autonomy`, `/analytics/token-usage` | GET×4 | KPIs, fidelidad de identidad, métricas de autonomía, uso de tokens |

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
    __slots__ = ("_autonomy_level",)                    # Solo un atributo permitido
    def __init__(self, autonomy_level: int = 0):
        object.__setattr__(self, "_autonomy_level", autonomy_level)  # Bypass del override
    def __setattr__(self, _name, _value):
        raise AttributeError("DecisionEngine is immutable")  # Bloquea toda mutación
    @property
    def autonomy_level(self) -> int: return self._autonomy_level  # Solo lectura
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

### `agent/src/flows/orchestrator.py` (829 líneas)

El Orchestrator es el **cerebro central** del sistema. Recibe un mensaje de usuario y lo procesa a través de un pipeline determinístico de 10 pasos, donde cada paso emite eventos vía EventBus y crea nodos de trace vía TraceCollector.

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
| 3 | **Planner** | `Planner.build()` — construye Plan con pasos ordenados. Respeta `governance_enabled` del Orchestrator. | No | <1ms |
| 4 | **Memory Recall** | Consulta working memory (RAM) + episodic (ChromaDB cosine search) + semantic (ChromaDB cosine search). Máximo 3 resultados por tier. Budget de ~2000 tokens. | No | 50-200ms |
| 5 | **Correction Injection** | Extrae correcciones conductuales de ProceduralMemory (SQLite) para el agente target. Se inyectan como reglas en el prompt. | No | <5ms |
| 6 | **Prompt Build** | Fusiona: memory context + correction context + extra context + conversation history. | No | <1ms |
| 7 | **LLM Generate** | Ruta al agente asignado → `ModelRouter.generate()` → proveedor LLM en la cadena de fallback. | Sí | 500-5000ms |
| 8 | **Identity Review** | **Per plan**: Si el Plan incluye paso `identity_review`, el IdentityCoreAgent revisa el output para alineación con la personalidad del principal. Puede reescribir la respuesta. | Sí | 500-3000ms |
| 9 | **Governance Review** | **Per plan**: Si el Plan incluye paso `governance_review`, el GovernanceAgent revisa el output. Produce JSON con `approved`, `risk_level`, `flags`, `revised_content`. Auto-approves en parse errors. | Sí | 500-3000ms |
| 10 | **Memory Store + Evaluation + Persist** | (a) Almacena en working + episodic memory. (b) Ejecuta 5 módulos de evaluación heurística (sin LLM). (c) Persiste a Postgres (fire-and-forget): interaction, trace_nodes, evaluations, token_usage. | No | 10-100ms |

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
    │  (222 ln)   │    │  (91 ln)    │        │  (91 ln)    │
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
| **Identity Core** | `IdentityCoreAgent` | `identity_core.py` (222 ln) | **Guardián de la personalidad**. Responde COMO el principal. Construye system prompt dinámico desde persona.yaml con Big Five traits mapeados a descripciones textuales + valores + estilo de comunicación + boundaries + writing style. | Dinámico (~1500 tokens) con persona completa | Gemini | 0.7 |
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

### `agent/src/router/model_router.py` (329 líneas)

Abstracción sobre los 3 proveedores LLM con fallback automático, asignación por rol de agente, y hot-swap:

```
Solicitud de generación
        │
        ▼
┌───────────────────┐    Falla?    ┌──────────────────┐    Falla?    ┌──────────────────┐
│  GEMINI 2.5 Flash │ ──────────→ │  GROQ Llama 3.3  │ ──────────→ │  OLLAMA qwen2.5  │
│  (Free tier)      │             │  70B (Free tier)  │             │  32b (Local)     │
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

### `agent/src/training/manager.py` (194 líneas)

Sistema de entrenamiento con 3 modos diseñados para que el principal enseñe progresivamente a su conciencia virtual:

### 15.1 Modos de Entrenamiento

| Modo | Propósito | Flujo |
|------|-----------|-------|
| **correction** | Corregir una respuesta incorrecta | El principal da: original response + corrección deseada + explicación. Se almacena como regla en ProceduralMemory. |
| **free_conversation** | Conversación libre para calibrar personalidad | El principal conversa normalmente. El sistema observa y aprende patrones. |
| **guided_interview** | Preguntas estructuradas para capturar identidad | El sistema hace preguntas específicas sobre valores, preferencias, estilo. |

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

### 16.4 Learn Topic — `agent/src/skills/learn_topic.py` (311 líneas)

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

### 17.2 Persistence Repository — `agent/src/db/persistence.py` (491 líneas)

Repositorio fire-and-forget que nunca bloquea ni crashea el pipeline:

```python
class PersistenceRepository:
    def save_interaction(...)      → bool   # Guarda metadata de interacción
    def save_trace_nodes(...)      → Dict   # Bulk insert de nodos + edges, retorna {node_id: pk}
    def save_all_evaluations(...)  → bool   # Guarda quality, alignment, legal, decisions
    def save_token_usage(...)      → bool   # Guarda uso de tokens (via callback del ModelRouter)
    def save_memory_operation(...) → bool   # Guarda operación de memoria (via callback del RollbackManager)
```

**Principio de diseño**: Cada método tiene `try/except` interno. Un fallo de persistencia NUNCA afecta la respuesta al usuario.

### 17.3 Resumen de Storage

| Motor | Ubicación | Propósito | Qué almacena |
|-------|-----------|-----------|-------------|
| **Neon Postgres** | Cloud (remoto) | Persistencia relacional | Conversaciones, mensajes, audit log, interacciones, traces, evaluaciones, token usage, memory ops |
| **ChromaDB** | `chroma_data/` (local) | Vector store | Episodic memory (logs de interacciones), Semantic memory (knowledge base, writing samples) |
| **SQLite** | `data/procedural.db` (local) | Memoria procedimental | Correcciones del training, workflows, estrategias |
| **RAM** | In-process | Cache rápida | Working memory (20 msgs), últimas 100 traces, últimos 200 reports de evaluación |

---

## 18. DASHBOARD — Interfaz de Control (Next.js)

### 18.1 Stack del Dashboard

- **Next.js 15** (App Router) + **React 19** + **TypeScript 5.7**
- **Tailwind CSS 3.4** + **shadcn/ui** para componentes
- **Tema lab oscuro** con variables CSS: `lab-text`, `lab-text-dim`, `lab-card`, `lab-surface`, `lab-border`, `accent-glow`, `accent-primary`, `status-green/amber/red/blue`
- **Zustand 5** para state management global (119 líneas)
- **API client** centralizado en `lib/api.ts` (650 líneas, ~60 métodos tipados)
- **i18n** vía `lib/i18n/` — soporta `en.json` + `es.json`
- Directiva `"use client"` en todas las páginas interactivas

### 18.2 Rutas del Dashboard (12 páginas)

| Ruta | Página | Líneas | Funcionalidad |
|------|--------|--------|--------------|
| `/` | Command Center | 189 | KPI cards (conversations, tokens, uptime, quality), agent status ring (5 agentes con estado visual), health LEDs, activity feed (real-time via WebSocket), persona card, router card, quick actions |
| `/chat` | Chat Interface | 11 (wrapper) + 167 (ChatPanel) + 51 (MessageBubble) | Chat de texto con el sistema orquestado. Envía POST /api/chat y renderiza respuestas con metadata (provider, model, latency) |
| `/identity` | Identity Studio | 417 | **Big Five sliders + radar chart** (5 dimensiones), estilo de comunicación (6 sliders), value hierarchy (drag-and-drop reordenable), behavioral boundaries, save/reload desde persona.yaml |
| `/training` | Training Center | 478 | 3 modos de entrenamiento (correction, free, guided), historial de sesiones, upload de writing samples con preview |
| `/testing` | Testing Playground | 366 | Chat simulator, scenario theater (escenarios predefinidos), A/B compare (comparar 2 respuestas side-by-side) |
| `/models` | Model Manager | 352 | Estado de proveedores (Gemini/Groq/Ollama), 4 perfiles switcheables, asignaciones por agente (qué modelo usa cada agente), endpoint de test |
| `/skills` | Skill Manager | 254 | Lista de skills con toggle on/off, ejecución de web research directa |
| `/memory` | Memory Lab | 494 | Stats cards (count por tier), semantic search interactivo, store new memories, **edit inline** y delete individual memories |
| `/governance` | Governance Console | 481 | Niveles de autonomía visual, configuración (read-only), audit log con filtros, approval queue con acciones funcionales (approve/reject/request changes), emergency stop/resume button |
| `/analytics` | Analytics Dashboard | 269 | Identity fidelity gauge, autonomy metrics, token usage charts (recharts), events breakdown by type/agent/risk |
| `/trace` | Cognitive Trace | 474 | Lista de trazas recientes, visualización React Flow del pipeline (nodos custom con expand/collapse), replay paso a paso |
| `/evaluation` | Evaluation Dashboard | 698 | Vista unificada de los 5 módulos: quality trends, alignment scores, legal risk flags, business decisions, memory rollback log |

### 18.3 Componentes Clave

| Directorio | Componentes | Líneas totales | Descripción |
|------------|------------|----------------|-------------|
| `components/command-center/` | 7 componentes | 1026 | KPI cards, activity feed (ScrollArea con eventos real-time), agent ring (SVG circular con 5 agentes), health bar (LEDs de servicios), persona card, router card, quick actions (i18n completo) |
| `components/chat/` | 2 componentes | 218 | ChatPanel (input + message list + submission) + MessageBubble (render individual con metadata) |
| `components/layout/` | 3 componentes | 266 | Sidebar (navegación agrupada en 4 secciones: Core, Identity & Training, Infrastructure, Observability), Header (breadcrumb + system status), ClientShell (wrapper con font loading) |
| `components/trace/` | 1 componente | 267 | TraceNode — nodo custom de React Flow con expand/collapse, colores por tipo, métricas inline |
| `components/ui/` | 8 componentes | 317 | Primitivos shadcn/ui: badge, button, card, input, progress, scroll-area, separator, tabs |

### 18.4 API Client — `dashboard/lib/api.ts` (~810 líneas)

~70 métodos tipados que mapean 1:1 a los endpoints del backend. Incluye métodos para service-log, interactions persistence, y persisted token usage:

```typescript
export const api = {
    // Chat
    chat: (message, conversationId?) => fetchAPI<ChatResponse>("/chat", { method: "POST", body }),
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
                              │  6. orchestrator.process(msg, id, hist)│
                              └────────────────┬─────────────────────┘
                                               │
                              ┌────────────────▼──────────────────────┐
                              │       ORCHESTRATOR PIPELINE           │
                              │                                       │
                              │  0. Emergency Check (si parado → STOP)│
                              │  1. Classify (keywords, <1ms)         │
                              │  2. Decision Engine (determinístico)   │
                              │  3. Planner (Plan con steps)          │
                              │  4. Memory Recall (ChromaDB ×2 + RAM) │
                              │  5. Correction Injection (SQLite)     │
                              │  6. Prompt Build (merge contextos)    │
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

## 21. QUÉ SOBREVIVE UN REINICIO

| Dato | Almacenamiento | En RAM | En Postgres | Persiste |
|------|---------------|--------|-------------|----------|
| Correcciones de training | SQLite (`procedural.db`) | — | — | Sí |
| Vectores episodic/semantic | ChromaDB (disco) | — | — | Sí |
| Conversaciones + mensajes | Neon Postgres | — | Sí | Sí |
| Audit log (eventos) | Neon Postgres | — | Sí | Sí |
| Service logs | Archivos rotativos (5×2MB) | — | — | Sí |
| Working memory (buffer) | RAM (Python list) | max 20 | — | No — se pierde |
| **Traces cognitivos** | **RAM + Postgres** | 100 | Ilimitado | **Sí** |
| **Evaluaciones** | **RAM + Postgres** | 200 | Ilimitado | **Sí** |
| **Decisiones detectadas** | **RAM + Postgres** | 500 | Ilimitado | **Sí** |
| **Memory operations** | **RAM + Postgres** | 1000 | Ilimitado | **Sí** |
| **Token usage** | **RAM + Postgres** | Unbounded | Ilimitado | **Sí** |

> **Principio**: Las escrituras a Postgres son **fire-and-forget** en paralelo al almacenamiento in-memory. Si Postgres falla, el sistema sigue funcionando idénticamente — solo pierde persistencia a largo plazo.

---

## 22. OBSERVACIONES Y PROBLEMAS CONOCIDOS

1. **Identity fidelity es heurístico** — el score actual se basa en conteo de correcciones y keyword matching, no en comparación de embeddings contra un baseline de identidad. La fidelidad real requerirá embedding-based scoring.

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

## 23. ESTADO ACTUAL VS PLANIFICADO

### Completado (Phase 1 + 2 + 3)
- Full crew de 5 agentes con orchestrator (pipeline de 10 pasos)
- Sistema de memoria de 4 niveles (ChromaDB + SQLite)
- Model Router con 3 proveedores + cadena de fallback + conteo de tokens real
- 12 páginas de dashboard (todas funcionales)
- 5 módulos de evaluación heurística (quality, alignment, legal risk, decisions, rollback)
- Cognitive trace con visualización React Flow (13 tipos de nodo)
- Governance console (config viewer, audit log, approval queue, emergency stop/resume)
- Analytics dashboard (fidelity, autonomy, tokens, events)
- Testing playground (chat sim, scenario theater, A/B compare)
- Training system (3 modos + upload de writing samples + correcciones)
- WebSocket real-time updates + Event Bus
- Service logger con crash reports
- Postgres persistence layer (interactions, traces, evaluations, token usage, memory ops)
- Cognition layer obligatoria (DecisionEngine inmutable + Planner stateless)
- Phase 3 architectural hardening (sin ruta legacy, 49 tests de cognición, TaskCategory extraído)

### Planificado (Phase 4+)
| Item | Prioridad | Descripción |
|------|-----------|-------------|
| **Human-in-the-Loop** | ALTA | Mecanismo de pausa para approval queue de governance |
| **Supabase Auth** | ALTA | JWT + login/logout + rutas protegidas |
| **Real Identity Fidelity** | CRÍTICA | Scoring basado en embeddings comparados con baseline |
| **Persona Version Control** | ALTA | Snapshots, diffs, rollback, ramas experimentales |
| **Document Chunking Pipeline** | ALTA | Chunking inteligente (500-1000 tokens con overlap) |
| **Memory Consolidation** | ALTA | Job background para resumir episodic → semantic |
| **Multi-Agent Collaboration** | MEDIA | Llamadas paralelas a agentes + agregación |
| **Identity Drift Detection** | MEDIA | Embeddings baseline + alerta de drift |
| **Autonomous Skill Acquisition** | MEDIA | Learning Agent pipeline |
| **External Integrations** | MEDIA | Email, calendar, Slack/Discord |
| **QLoRA Fine-Tuning** | BAJA | PEFT + Unsloth para modelo privado |
| **Self-Modification System** | BAJA | Acceso al codebase con governance |

---

## 24. ESTRUCTURA DE ARCHIVOS COMPLETA

```
iame.lol/
├── agent/                                    # Backend Python FastAPI
│   ├── src/
│   │   ├── agents/                           # 5 agentes especializados
│   │   │   ├── base_agent.py         (91 ln) # ABC para agentes de dominio
│   │   │   ├── identity_core.py     (222 ln) # Guardián de identidad (standalone)
│   │   │   ├── business_agent.py     (50 ln) # Estratega de negocio
│   │   │   ├── communication_agent.py(56 ln) # Especialista en comunicación
│   │   │   ├── technical_agent.py    (44 ln) # Constructor técnico
│   │   │   ├── governance_agent.py  (132 ln) # Meta-agente de cumplimiento
│   │   │   └── crew.py             (111 ln) # Inicialización y gestión del crew
│   │   ├── api/
│   │   │   ├── main.py             (249 ln) # Composition root + AppState + lifespan
│   │   │   └── routes.py          (1606 ln) # 73 endpoints REST + WebSocket
│   │   ├── cognition/                        # Capa cognitiva OBLIGATORIA
│   │   │   ├── __init__.py          (20 ln) # Re-exports: DecisionEngine, Planner, TaskCategory
│   │   │   ├── decision_engine.py  (138 ln) # Motor de decisión inmutable
│   │   │   └── planner.py         (118 ln) # Planificador stateless
│   │   ├── db/
│   │   │   ├── database.py        (289 ln) # Postgres connection + 10 tablas
│   │   │   └── persistence.py     (491 ln) # Fire-and-forget persistence repository
│   │   ├── evaluation/                       # 5 módulos heurísticos
│   │   │   ├── quality_scorer.py   (383 ln) # Calidad en 5 dimensiones → grade A-F
│   │   │   ├── alignment_evaluator.py(383 ln) # Alineación con persona
│   │   │   ├── legal_risk.py      (319 ln) # 15+ regex patterns de riesgo legal
│   │   │   ├── decision_registry.py(358 ln) # Detección de decisiones de negocio
│   │   │   └── memory_rollback.py  (353 ln) # Auditoría + point-in-time recovery
│   │   ├── events/
│   │   │   └── event_bus.py       (128 ln) # Pub/Sub + WS + Audit
│   │   ├── flows/
│   │   │   ├── categories.py       (18 ln) # TaskCategory enum (compartido)
│   │   │   └── orchestrator.py    (829 ln) # Pipeline de 10 pasos
│   │   ├── memory/
│   │   │   └── manager.py         (520 ln) # 4-tier unified memory
│   │   ├── router/
│   │   │   └── model_router.py    (329 ln) # Gemini→Groq→Ollama fallback
│   │   ├── skills/
│   │   │   ├── registry.py        (116 ln) # Skill toggle + tracking
│   │   │   ├── web_research.py    (106 ln) # Tavily wrapper
│   │   │   ├── learn_topic.py     (311 ln) # Topic learning pipeline
│   │   │   └── tools.py          (110 ln) # Email + Document tools
│   │   ├── trace/
│   │   │   └── collector.py       (342 ln) # Cognitive trace + TraceStore
│   │   ├── training/
│   │   │   └── manager.py         (194 ln) # 3 modos + correcciones
│   │   ├── config.py               (98 ln) # Pydantic BaseSettings
│   │   ├── service_logger.py      (218 ln) # Rotating file logger
│   │   └── watchdog.py            (111 ln) # Service health watchdog
│   ├── tests/                                # 12 archivos + conftest.py
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
│   │   ├── test_config.py          (86 ln) # Tests de configuración
│   │   └── test_basics.py          (59 ln) # Tests básicos de importación
│   └── configs → ../configs                  # Symlink a configs/
├── dashboard/                                # Frontend Next.js 15
│   ├── app/                                  # 12 rutas (App Router)
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
│   │   ├── trace/page.tsx         (474 ln) # Cognitive Trace viewer
│   │   └── evaluation/page.tsx    (698 ln) # Evaluation Dashboard
│   ├── components/
│   │   ├── command-center/       (1026 ln) # 7 componentes del dashboard principal
│   │   ├── chat/                  (218 ln) # ChatPanel + MessageBubble
│   │   ├── layout/                (266 ln) # Sidebar (4 grupos) + Header + ClientShell
│   │   ├── trace/                 (267 ln) # TraceNode (React Flow custom)
│   │   └── ui/                    (317 ln) # 8 primitivos shadcn/ui
│   └── lib/
│       ├── api.ts                 (807 ln) # API client (~70 métodos)
│       ├── store.ts               (147 ln) # Zustand global store
│       ├── hooks/                           # Custom React hooks
│       └── i18n/                            # en.json + es.json
├── configs/                                  # Archivos de configuración
│   ├── persona.yaml                (87 ln) # Identidad del principal
│   ├── models.json                (~80 ln) # Proveedores LLM + asignaciones
│   ├── skills.json                (~30 ln) # Registro de skills
│   └── governance.yaml            (254 ln) # Framework de gobernanza
├── data/                                     # (gitignored) Training data
├── docs/
│   └── arquitectura.md                       # Este documento
└── Base Guideline.md                         # Estrategia general del proyecto
```

**Total de código backend (Python)**: ~7,000 líneas en `agent/src/`
**Total de tests**: 1,851 líneas en 12 archivos (235 tests, 230 passing, 5 pre-existing failures)
**Total de código frontend (TypeScript/TSX)**: ~8,100 líneas en `dashboard/`

---

*Última actualización: 2025-02-18 — Learn Topic Skill implementada (Phase 3.2)*
*Pipeline: web search → LLM summarize → chunk → ChromaDB semantic memory*
*Activable desde chat ("aprende sobre X") y Skill Manager UI*
*Preparado para auditoría de especialistas en conciencias virtuales*
