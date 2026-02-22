# Base de Datos

> [← Seguridad](security.md) · [API →](../api/README.md)

---

## Sobre este documento

Este documento describe la **capa de persistencia** de Django — los 3 sistemas de base de datos que almacenan todo lo que Django sabe, hace y recuerda. Sin base de datos, Django perdería toda su historia, evaluaciones y memorias semánticas cada vez que el servidor se reinicia. La persistencia es lo que le da continuidad a largo plazo.

### ¿Qué cubre este documento?

Documenta las **3 bases de datos** (Neon Postgres remoto, ChromaDB local, SQLite local), las **15 tablas de Postgres** organizadas en 4 grupos (Core, Persistence, Hybrid Search, Teleología), el `DatabaseManager` (289 líneas) que maneja las conexiones, y el `PersistenceRepository` (1,507 líneas, 25+ métodos) que proporciona escritura fire-and-forget y lectura de todos los artefactos del pipeline.

### ¿Cuál es su función en la arquitectura?

La base de datos es la **memoria a largo plazo del sistema** (distinta de la [Memoria](memory.md) de Django que es funcional). Mientras que ChromaDB guarda las memorias semánticas y episódicas que Django consulta para responder, y SQLite guarda los procedimientos aprendidos, Postgres guarda todo lo operacional: interacciones, trazas cognitivas, evaluaciones, uso de tokens, operaciones de memoria, metas, y el audit log.

### ¿Cómo afecta al comportamiento de Django?

**Principio clave: la base de datos es completamente opcional**. Si Postgres no está disponible, Django sigue funcionando — simplemente no persiste historia. Esto es intencional ("fire-and-forget"): un fallo de DB **nunca** bloquea una respuesta al usuario.

- ChromaDB es **esencial** para la memoria semántica y episódica — sin ella, Django no recuerda conocimiento aprendido
- SQLite es **esencial** para correcciones y procedimientos — sin ella, el training pierde efecto entre reinicios
- Postgres permite ver **historia** en dashboards (Analytics, Evaluation, Trace, Identity Governance) pero su ausencia no afecta respuestas

### ¿Cómo interactúa con las demás piezas?

La capa de datos es **consumida por casi todo el sistema**:
- **[Pipeline](pipeline.md) paso 10**: `PersistenceRepository.save_interaction()` + `save_trace_nodes()` + `save_all_evaluations()` persisten cada interacción
- **[Memoria](memory.md)**: ChromaDB almacena las colecciones `episodic_memory` y `semantic_memory`. SQLite almacena `procedural.db`. Las operaciones se registran en Postgres via `save_memory_operation()`
- **[Identidad](identity.md)**: los snapshots de versión se almacenan como `config_versions` en Postgres. Las señales históricas se leen via `get_recent_identity_signals()` para Health Monitor y Evolution
- **[Evaluación](evaluation.md)**: los 5 módulos de evaluación persisten sus resultados en la tabla `evaluations`
- **[Eventos](events.md)**: el EventBus persiste cada evento en la tabla `audit_log` de Postgres
- **[Model Router](../integrations/README.md)**: el callback de persistencia de tokens se conecta durante el [Startup](startup.md) y guarda cada llamada LLM en `token_usage`
- **[Teleología](teleology.md)**: 4 tablas dedicadas (`goals`, `goal_progress`, `goal_conflicts`, `reward_signals`)
- **[Dashboard](../dashboard/README.md)**: múltiples páginas consultan Postgres para historia (Analytics, Evaluation, Trace, Identity Governance, Governance Console)
- **[API](../api/README.md)**: los endpoints de persistence (4) exponen interacciones, trazas y evaluaciones históricas
- **[Startup](startup.md)**: `DatabaseManager` es el segundo componente en inicializarse (después de Settings), seguido inmediatamente por `PersistenceRepository`

---

## 3 Bases de Datos

| DB | Tipo | Ubicación | Driver | Uso |
|----|------|-----------|--------|-----|
| **Neon Postgres** | SQL relacional (remoto) | Neon cloud | psycopg2 (sync, autocommit) | Core data, persistence, teleología |
| **ChromaDB** | Vector DB (local) | `chroma_data/` | chromadb | Semantic/episodic memory (RAG) |
| **SQLite** | SQL relacional (local) | `data/procedural.db` | sqlite3 | Correcciones de training |

> **Postgres es totalmente opcional** — la app degrada gracefully sin conexión a DB.

---

## Postgres (Neon) — 15 Tablas

### Core (5 tablas)

| Tabla | Propósito | Escritura desde |
|-------|-----------|-----------------|
| `conversations` | Conversaciones (UUID, created_at) | `POST /chat` |
| `messages` | Mensajes (user + assistant per conversation) | `POST /chat` |
| `audit_log` | Log de auditoría (todos los [eventos](events.md)) | EventBus → `set_db_callback()` |
| `kpi_snapshots` | Snapshots de KPIs periódicos | Background jobs |
| `config_versions` | Versiones de configuración (identity profiles, configs) | [IdentityManager](identity.md), ID governance |

### Persistence (5 tablas)

| Tabla | Propósito | Escritura desde |
|-------|-----------|-----------------|
| `interactions` | Interacciones completas del orchestrator | `save_interaction()` |
| `trace_nodes` | Nodos de [traza cognitiva](events.md) | `save_trace_nodes()` |
| `evaluations` | Resultados de los [5 módulos](evaluation.md) + identity evaluations | `save_all_evaluations()` |
| `token_usage` | Uso de tokens per LLM call | ModelRouter callback |
| `memory_operations` | Operaciones de [memoria](memory.md) (add/update/delete) | MemoryRollbackManager callback |

### Hybrid Search (1 tabla)

| Tabla | Propósito |
|-------|-----------|
| `episodic_search` | Mirror FTS con tsvector + GIN index para [hybrid search](memory.md) |

### Teleology (4 tablas)

| Tabla | Propósito | Escritura desde |
|-------|-----------|-----------------|
| `goals` | Metas del [sistema teleológico](teleology.md) | Goals API |
| `plans` | Planes de ejecución | PlanExecutor |
| `reward_signals` | Señales de recompensa (rolling 500) | RewardComputer middleware |
| `goal_events` | Eventos de lifecycle de metas | GoalManager |

---

## ChromaDB (local)

| Colección | Propósito | Embedding |
|-----------|-----------|-----------|
| `episodic_memory` | Logs de interacción, búsqueda semántica | all-MiniLM-L6-v2 (384-dim) |
| `semantic_memory` | Base de conocimiento, writing samples, `learned_knowledge` | all-MiniLM-L6-v2 (384-dim) |

Persiste en disco (`chroma_data/`). Categorías de semantic memory: `learned_knowledge`, `personality_trait`, `writing_sample`, custom.

---

## SQLite

Archivo: `data/procedural.db`. Almacena correcciones de [training](../dashboard/training-center.md) y patrones procedurales.

---

## PersistenceRepository — `src/db/persistence.py` (1,507 líneas)

**Fire-and-forget** — un fallo NUNCA afecta la respuesta al usuario. 25+ métodos:

### Write

| Método | Tabla(s) | Llamado desde |
|--------|----------|--------------|
| `save_interaction()` | `interactions` | Pipeline paso 10 |
| `save_trace_nodes()` | `trace_nodes` | Pipeline paso 10 |
| `save_all_evaluations()` | `evaluations` | Pipeline paso 10 |
| `save_token_usage()` | `token_usage` | ModelRouter callback |
| `save_memory_operation()` | `memory_operations` | MemoryRollbackManager callback |
| `store_identity_snapshot()` | `config_versions` | Identity governance API |

### Read

| Método | Tabla(s) | Usado por |
|--------|----------|-----------|
| `get_evaluation_quality()` | `evaluations` | Evaluation API |
| `get_evaluation_alignment()` | `evaluations` | Evaluation API |
| `get_evaluation_legal_risk()` | `evaluations` | Evaluation API |
| `get_evaluation_decisions()` | `evaluations` | Evaluation API |
| `get_evaluation_rollback()` | `evaluations` | Evaluation API |
| `get_recent_identity_signals()` | `evaluations` | [Health Monitor](identity.md) (paso 9a) |
| `list_traces()` | `interactions` + `trace_nodes` | Trace API |
| `get_analytics_overview()` | `audit_log` + `token_usage` | Analytics API |
| `get_autonomy_stats()` | `evaluations` | Analytics API |
| `get_identity_versions()` | `config_versions` | Identity governance API |

### Delete

| Método | Tabla(s) |
|--------|----------|
| `delete_evaluation_data()` | `evaluations` |
| `delete_analytics_events()` | `audit_log` |
| `delete_analytics_tokens()` | `token_usage` |

---

## Database Manager — `src/db/database.py` (354 líneas)

Gestión de conexión y schema:
- `.connect()` + `.initialize_schema()` → crea las 15 tablas si no existen
- `event_bus.set_db_callback()` → wiring de audit log
- `.close()` en shutdown

---

## Diagrama de Flujo de Datos

```
Orchestrator Pipeline
  ├── save_interaction() ──────── → interactions
  ├── save_trace_nodes() ──────── → trace_nodes
  ├── save_all_evaluations() ──── → evaluations (quality, alignment, legal, decisions, rollback, identity_*)
  └── EventBus.emit() ─────────── → audit_log

ModelRouter.generate()
  └── persist_callback() ──────── → token_usage

MemoryManager
  ├── store_interaction() ──────── → ChromaDB episodic_memory
  ├── store_semantic() ─────────── → ChromaDB semantic_memory
  └── rollback_callback() ──────── → memory_operations

TrainingManager
  └── submit_correction() ──────── → SQLite procedural.db

GoalManager
  ├── create/update/delete ──────── → goals
  └── events ────────────────────── → goal_events
```

---

## Temas Relacionados

- [Pipeline](pipeline.md) — Persistence en paso 10
- [Memoria](memory.md) — ChromaDB + SQLite
- [Evaluación](evaluation.md) — Datos de evaluación en Postgres
- [Eventos](events.md) — audit_log
- [Identidad](identity.md) — config_versions, identity signals
- [Teleología](teleology.md) — goals, plans, reward_signals
- [Startup](startup.md) — Secuencia de inicialización de DB (paso 2-3)
