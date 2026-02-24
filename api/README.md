# API REST — 120 Endpoints

> [← Base de Datos](../architecture/database.md) · [Dashboard →](../dashboard/README.md)

**Archivo**: `src/api/routes.py` (3,900 líneas)

---

## Sobre este documento

Este documento es la **referencia completa de los 120 endpoints** de la API REST de ADLRA — el contrato técnico entre el backend y todo lo que lo consume (el dashboard, el bot de Discord, o cualquier cliente externo). Cada endpoint está documentado con su método HTTP, path, descripción, bases de datos que toca, y eventos que emite.

### ¿Qué cubre este documento?

Documenta los **19 grupos de endpoints** organizados por funcionalidad: Health & Status (4), Service Log (2), Chat (2), Configuration (2), Persona (5), Crew (1), Trace (6), Persisted Interactions (4), Memory (7), Skills (5), Dynamic Skills (5), Training (14), Models (4), Evaluation (21), Governance (6), Analytics (6), Identity Governance (11), Middleware (2), y Teleology/Goals (11). Para cada endpoint se indica método, path, qué hace, qué DB consulta, y qué eventos emite.

### ¿Cuál es su función en la arquitectura?

La API es la **interfaz pública del backend** — la única forma de comunicarse con Django desde fuera. El dashboard, el bot de Discord, y cualquier herramienta externa interactúan exclusivamente a través de estos endpoints. Sin la API, los 25+ pasos del pipeline, los 22 módulos de identidad, los 4 niveles de memoria, y todo lo demás estarían encerrados en el backend sin forma de acceder a ellos.

### ¿Cómo afecta al comportamiento de Django?

- El endpoint **`POST /chat`** es el más crítico — es el punto de entrada que inicia el [pipeline completo](../architecture/pipeline.md) para cada mensaje
- El **WebSocket `/ws`** permite comunicación bidireccional en tiempo real (recibe eventos del [EventBus](../architecture/events.md), puede enviar chat directo que bypasea el orchestrator)
- Los endpoints de **Persona** permiten modificar en caliente quién es Django — cambian `persona.yaml` y recargan todos los [agentes](../architecture/agents.md)
- Los endpoints de **Training** permiten corregir, entrenar y enseñar a Django en tiempo real
- Los endpoints de **Identity Governance** permiten gestionar versiones de identidad, aprobar/rechazar evoluciones, y hacer rollback
- **Emergency Stop** (`POST /governance/emergency-stop`) bloquea instantáneamente todo el pipeline

### ¿Cómo interactúa con las demás piezas?

La API es el **puente central**:
- **[Dashboard](../dashboard/README.md)**: `lib/api.ts` (~111 métodos) consume estos endpoints. Cada página del dashboard mapea a un grupo de endpoints específico
- **[Discord Bot](../integrations/README.md)**: usa `POST /api/chat` y `POST /memory/working/clear` exclusivamente
- **[Pipeline](../architecture/pipeline.md)**: el handler de `POST /chat` crea la conversación en DB, guarda mensajes, emite eventos de estado, y llama a `orchestrator.process()`
- **[Base de Datos](../architecture/database.md)**: la mayoría de endpoints leen/escriben Postgres y/o ChromaDB — la columna "DB" de cada tabla lo indica
- **[Eventos](../architecture/events.md)**: los endpoints de mutación emiten eventos via el EventBus que llegan al dashboard en tiempo real
- **[Config](../config/README.md)**: los endpoints de reload (`/router/reload`, `/persona/reload`) recargan configuración sin reiniciar el servidor
- **[Startup](startup.md)**: todas las rutas se registran en `routes.py` y se montan en la app FastAPI durante la inicialización

El patrón general de cada endpoint es: lazy import de `get_state()` → acceder al componente necesario → ejecutar la operación → devolver JSON.

---

## Resumen

| Métrica | Valor |
|---------|-------|
| Total endpoints | **120** |
| GET | 56 |
| POST | 37 |
| PUT | 10 |
| DELETE | 13 |
| WebSocket | 1 |
| Grupos | 19 |

Patrón backend:
```python
@router.get("/endpoint")
async def handler():
    from src.api.main import get_state   # lazy import
    state = get_state()
    result = state.component.method()
    return {"status": "ok", **result}
```

Cliente dashboard: `dashboard/lib/api.ts` (~111 métodos) → ver [Dashboard](../dashboard/README.md).

---

## 10.1 Health & Status (4 endpoints)

| Método | Path | Descripción | DB |
|--------|------|-------------|-----|
| `GET` | `/health` | Pulse: versión, proveedores, uptime, DB status | — |
| `GET` | `/router/status` | Model Router: providers, profiles, circuit breakers, latency | — |
| `GET` | `/events/recent` | Eventos recientes de EventBus/audit_log | Postgres `audit_log` |
| `GET` | `/system-status` | Status compuesto pesado para [Health Bar](../dashboard/command-center.md) | Postgres (6 queries), ChromaDB, in-memory |

Dashboard: [Command Center](../dashboard/command-center.md) HealthBar, RouterCard.

---

## 10.2 Service Log (2 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/service-log/events` | Eventos del JSONL log file |
| `GET` | `/service-log/report` | Crash report con detección de patrones |

---

## 10.3 Chat (2 endpoints)

| Método | Path | Descripción | DB | Eventos |
|--------|------|-------------|-----|---------|
| `POST` | `/chat` | Chat principal — [pipeline](../architecture/pipeline.md) completo. Input sanitization, conversation management | Write: conversations, messages. ChromaDB via orchestrator | `chat.message_received`, `agent_state.*`, `chat.response_generated` |
| `WS` | `/ws` | WebSocket real-time + chat directo (bypass orchestrator → identity_core) | — | Recibe todos los eventos del bus |

Dashboard: [Chat](../dashboard/chat.md).

---

## 10.4 Configuration (2 endpoints)

| Método | Path | Descripción | Eventos |
|--------|------|-------------|---------|
| `POST` | `/router/reload` | Hot-reload models.json | `config.router_reloaded` |
| `POST` | `/persona/reload` | Hot-reload persona.yaml across all agents | `config.persona_reloaded` |

Dashboard: [Command Center](../dashboard/command-center.md) Quick Actions.

---

## 10.5 Persona / Identity Studio (5 endpoints)

| Método | Path | Descripción | Eventos |
|--------|------|-------------|---------|
| `GET` | `/persona/info` | Persona actual: name, Big Five, values, communication, boundaries | — |
| `PUT` | `/persona/traits` | Actualizar Big Five (0.0-1.0) → escribe YAML → reload all agents | `config.traits_updated` |
| `PUT` | `/persona/values` | Reemplazar lista de valores | `config.values_updated` |
| `PUT` | `/persona/communication` | Actualizar sliders de comunicación | `config.communication_updated` |
| `PUT` | `/persona/boundaries` | Reemplazar boundaries | `config.boundaries_updated` |

Dashboard: [Identity Studio](../dashboard/identity-studio.md).

---

## 10.6 Agent Crew (1 endpoint)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/crew/agents` | Lista 5 [agentes](../architecture/agents.md) con roles y descripciones |

---

## 10.7 Cognitive Trace (6 endpoints)

| Método | Path | Descripción | DB |
|--------|------|-------------|-----|
| `GET` | `/trace/list` | Lista trazas recientes (Postgres → fallback memory) | interactions + trace_nodes |
| `GET` | `/trace/{id}` | Grafo completo de traza | interactions + trace_nodes + evaluations |
| `GET` | `/trace/latest/graph` | Traza más reciente | Postgres |
| `POST` | `/trace/replay` | Re-procesa mensaje por pipeline completo | Write: all orchestrator ops |
| `DELETE` | `/trace/{id}` | Elimina una traza | DELETE interactions, trace_nodes, evaluations |
| `DELETE` | `/trace` | Elimina TODAS las trazas | DELETE all |

Dashboard: [Cognitive Trace](../dashboard/cognitive-trace.md).

---

## 10.8 Persisted Interactions (4 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/interactions` | Lista interacciones con filtro de categoría y paginación |
| `GET` | `/interactions/{id}/trace` | Traza persisted de una interacción |
| `GET` | `/interactions/{id}/evaluations` | Evaluaciones de una interacción |
| `GET` | `/token-usage/persisted` | Token usage agregado desde Postgres |

---

## 10.9 Memory System (7 endpoints)

| Método | Path | Descripción | DB |
|--------|------|-------------|-----|
| `GET` | `/memory/stats` | Stats de los 4 [tiers](../architecture/memory.md) | ChromaDB + SQLite |
| `POST` | `/memory/search` | Búsqueda cross-tier con filtro | ChromaDB + working |
| `POST` | `/memory/working/clear` | Limpiar working memory (per-conversation o all) | — (ephemeral) |
| `POST` | `/memory/bulk-delete` | Eliminar múltiples memorias across tiers | ChromaDB + memory_operations |
| `PUT` | `/memory/semantic/{id}` | Actualizar contenido/categoría | ChromaDB + memory_operations |
| `DELETE` | `/memory/semantic/{id}` | Eliminar memoria semántica | ChromaDB + memory_operations |
| `DELETE` | `/memory/episodic/{id}` | Eliminar memoria episódica | ChromaDB + memory_operations |

Dashboard: [Memory Lab](../dashboard/memory-lab.md).

---

## 10.10 Skills (5 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/skills` | Lista skills registradas |
| `POST` | `/skills/{id}/toggle` | Enable/disable skill |
| `POST` | `/skills/web-research` | Ejecuta web research via Tavily |
| `POST` | `/skills/learn-topic` | Research → summarize → chunk → ChromaDB |
| `POST` | `/skills/explore-repo` | Explora repos locales/GitHub/docs. Body: `{type, target, store}` |

Dashboard: [Skill Manager](../dashboard/skill-manager.md).

---

## 10.10b Dynamic Skills — SkillForge (5 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/skills/dynamic` | Lista dynamic skills cargados (loaded_skills, sandbox_files, loaded_count) |
| `POST` | `/skills/dynamic/create` | Crear skill desde descripción en lenguaje natural via Claude |
| `DELETE` | `/skills/dynamic/{name}` | Desinstalar dynamic skill (remove from registry + delete file) |
| `POST` | `/skills/dynamic/{name}/test` | Ejecuta smoke test de dynamic skill |
| `POST` | `/skills/dynamic/{name}/reload` | Recarga dynamic skill desde disco |

---

## 10.11 Training System (14 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/training/status` | Status del sistema |
| `POST` | `/training/session/start` | Iniciar sesión de entrenamiento |
| `POST` | `/training/session/end` | Terminar sesión activa |
| `POST` | `/training/correction` | Enviar corrección (original→corrected) |
| `GET` | `/training/corrections` | Listar correcciones |
| `PUT` | `/training/corrections/{id}` | Actualizar corrección |
| `DELETE` | `/training/corrections/{id}` | Eliminar corrección |
| `DELETE` | `/training/suggestions/{idx}` | Eliminar suggestion individual |
| `DELETE` | `/training/suggestions` | Limpiar todas las suggestions |
| `GET` | `/training/history` | Historial de sesiones + suggestions |
| `POST` | `/training/exchange` | Intercambio de conversación libre (legacy) |
| `GET` | `/training/interview/questions` | 15 preguntas de entrevista guiada |
| `POST` | `/training/interview/answer` | Respuesta de entrevista (legacy) |
| `POST` | `/training/upload-samples` | Upload .txt/.md → chunk → ChromaDB semantic |

Dashboard: [Training Center](../dashboard/training-center.md).

---

## 10.12 Models (4 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/models/config` | Config completa + estado online de providers |
| `PUT` | `/models/assignment` | Asignar modelo a agente específico |
| `PUT` | `/models/profile` | Cambiar perfil activo (balanced/max_quality/privacy/budget) |
| `POST` | `/models/test` | Test health check de un provider |

Dashboard: [Model Manager](../dashboard/model-manager.md).

---

## 10.13 Evaluation System (21 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/evaluation/overview` | Overview combinado de 5 sistemas |
| `GET` | `/evaluation/quality/stats` | Estadísticas de calidad |
| `GET` | `/evaluation/quality/reports` | Reportes recientes |
| `GET` | `/evaluation/quality/{id}` | Reporte por interacción |
| `GET` | `/evaluation/alignment/stats` | Estadísticas de alignment |
| `GET` | `/evaluation/alignment/reports` | Reportes recientes |
| `GET` | `/evaluation/legal/stats` | Estadísticas de riesgo legal |
| `GET` | `/evaluation/legal/reports` | Reportes recientes |
| `GET` | `/evaluation/legal/flagged` | Assessments por encima de threshold |
| `GET` | `/evaluation/decisions/stats` | Stats del registro de decisiones |
| `GET` | `/evaluation/decisions` | Lista (filtro status/category) |
| `GET` | `/evaluation/decisions/pending` | Decisiones pendientes |
| `PUT` | `/evaluation/decisions/{id}/status` | Actualizar estado (approve/reject/defer) |
| `POST` | `/evaluation/decisions/register` | Registrar decisión manualmente |
| `GET` | `/evaluation/rollback/stats` | Stats de rollback |
| `GET` | `/evaluation/rollback/log` | Log de operaciones |
| `POST` | `/evaluation/rollback/undo/{id}` | Deshacer operación |
| `GET` | `/evaluation/rollback/checkpoints` | Checkpoints de memoria |
| `POST` | `/evaluation/rollback/checkpoint` | Crear checkpoint |
| `POST` | `/evaluation/rollback/checkpoint/{id}/rollback` | Rollback a checkpoint |
| `DELETE` | `/evaluation/data` | Eliminar TODOS los datos |

Dashboard: [Evaluation Dashboard](../dashboard/evaluation-dashboard.md).

---

## 10.14 Governance (6 endpoints)

| Método | Path | Descripción | Eventos |
|--------|------|-------------|---------|
| `GET` | `/governance/config` | Configuración YAML | — |
| `GET` | `/governance/audit-log` | Log de auditoría reciente | — |
| `GET` | `/governance/approvals` | Aprobaciones pendientes | — |
| `POST` | `/governance/emergency-stop` | **EMERGENCY STOP** | `governance.emergency_stop_activated` (critical) |
| `POST` | `/governance/emergency-resume` | Resume operaciones | `governance.emergency_stop_deactivated` |
| `GET` | `/governance/emergency-status` | Estado del emergency stop | — |

Dashboard: [Governance Console](../dashboard/governance-console.md).

---

## 10.15 Analytics (6 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/analytics/overview` | Eventos por tipo/agente/riesgo |
| `GET` | `/analytics/identity-fidelity` | Score de fidelidad |
| `GET` | `/analytics/autonomy` | Métricas de autonomía |
| `GET` | `/analytics/token-usage` | Token usage últimos 30 días |
| `DELETE` | `/analytics/events` | Eliminar audit_log |
| `DELETE` | `/analytics/tokens` | Eliminar token_usage |

Dashboard: [Analytics](../dashboard/analytics.md).

---

## 10.16 Identity Governance (11 endpoints)

| Método | Path | Descripción | Eventos |
|--------|------|-------------|---------|
| `GET` | `/identity/versions` | Listar snapshots | — |
| `GET` | `/identity/versions/{id}` | Detalle de snapshot | — |
| `POST` | `/identity/snapshot` | Crear snapshot manual | — |
| `POST` | `/identity/activate/{id}` | Activar versión (idempotent) | `identity.version_activated` |
| `POST` | `/identity/rollback/{id}` | **ROLLBACK** — invalida posteriores | `identity.version_rolled_back` (high) |
| `DELETE` | `/identity/versions/{id}` | Eliminar snapshot (guard: no active) | — |
| `GET` | `/identity/evolution` | Candidatos de evolución | — |
| `POST` | `/identity/evolution/{id}/approve` | Aprobar evolución → snapshot | `identity.evolution_approved` |
| `POST` | `/identity/evolution/{id}/reject` | Rechazar evolución | `identity.evolution_rejected` |
| `GET` | `/identity/shadow` | Shadow simulation results | — |
| `GET` | `/identity/health` | Señales de salud + latest health | — |

Dashboard: [Identity Governance](../dashboard/identity-governance.md).

---

## 10.17 Middleware (2 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/middleware/list` | Lista middleware registrado |
| `POST` | `/middleware/{name}/toggle` | Toggle enable/disable |

Dashboard: [Governance Console](../dashboard/governance-console.md) (Middleware tab).

---

## 10.18 Teleology / Goals (11 endpoints)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/goals` | Listar metas (filtro status/type) |
| `GET` | `/goals/{id}` | Detalle de meta |
| `POST` | `/goals` | Crear meta |
| `PUT` | `/goals/{id}` | Actualizar meta |
| `DELETE` | `/goals/{id}` | Eliminar meta |
| `POST` | `/goals/{id}/activate` | CREATED → ACTIVE |
| `POST` | `/goals/{id}/complete` | Marcar completada |
| `POST` | `/goals/{id}/cancel` | Cancelar |
| `GET` | `/goals/conflicts/detect` | Detectar conflictos |
| `GET` | `/goals/rewards/recent` | Señales de recompensa |
| `POST` | `/goals/{id}/governance` | Evaluar governance |

Dashboard: [Goals](../dashboard/goals.md).

---

## Temas Relacionados

- [Arquitectura](../architecture/README.md) — Visión general del backend
- [Pipeline](../architecture/pipeline.md) — Flujo end-to-end de un mensaje
- [Base de Datos](../architecture/database.md) — Tablas que los endpoints leen/escriben
- [Eventos](../architecture/events.md) — Eventos emitidos por endpoints
- [Dashboard](../dashboard/README.md) — Cliente que consume estos endpoints
