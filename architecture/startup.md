# Inicialización del Sistema

> [← Arquitectura](README.md) · [Pipeline →](pipeline.md)

---

## Sobre este documento

Este documento explica **cómo arranca Django** — el proceso exacto que ocurre desde que el servidor se enciende hasta que Django está listo para recibir su primera interacción. Parece un detalle técnico menor, pero es fundamental: el **orden de inicialización** determina qué componentes existen, cómo se conectan entre sí, y qué pasa si alguno falla.

### ¿Qué cubre este documento?

Documenta el singleton `AppState` (el contenedor global que almacena todos los componentes del sistema), la secuencia `lifespan()` de 19 pasos que los inicializa uno por uno, los callbacks que se conectan entre módulos durante el arranque, la secuencia de apagado, y el middleware HTTP que se ejecuta en cada request.

### ¿Cuál es su función en la arquitectura?

Este es el **compositor** — la pieza que construye el orquesta antes de que suene la primera nota. Sin la secuencia de startup, los módulos individuales (cognición, memoria, identidad, etc.) serían piezas sueltas sin conexión. Es aquí donde se materializan las dependencias: el `DecisionEngine` recibe el `IdentityProfile`, el `Orchestrator` recibe el `DecisionEngine` y el `Planner`, el `AlignmentEvaluator` recibe el baseline embedding, y los callbacks de persistencia se conectan al `ModelRouter` y al `MemoryRollbackManager`.

### ¿Cómo afecta al comportamiento de Django?

El orden de arranque es **crítico**. Si un componente se inicializa antes de que su dependencia esté lista, Django operaría con capacidades reducidas o directamente no arrancaría. Por ejemplo:
- Si `IdentityManager` no carga antes de `DecisionEngine`, Django no tendría perfil de identidad — sus decisiones serían genéricas
- Si `MemoryManager` no existe antes del `Orchestrator`, Django no recordaría nada
- Si la base de datos falla, Django sigue funcionando (degradación graceful) pero sin persistencia histórica

### ¿Cómo interactúa con las demás piezas?

El startup es literalmente donde **todo se conecta con todo**:
- Carga la [Configuración](../../configs/) (persona.yaml, models.json, governance.yaml) y la inyecta en los módulos correspondientes
- Inicializa la [Base de Datos](database.md) y crea el `PersistenceRepository` que usan los [Eventos](events.md) y el [Pipeline](pipeline.md)
- Construye el [Model Router](../../docs/integrations/README.md) y le conecta el callback de persistencia de tokens
- Crea los [Agentes](agents.md) (AgentCrew) cargando sus prompts de sistema desde persona.yaml
- Inicializa los 4 niveles de [Memoria](memory.md) (ChromaDB + SQLite + working)
- Carga el perfil de [Identidad](identity.md) desde Postgres (o lo reconstruye desde YAML) y lo inyecta en el DecisionEngine
- Construye los componentes de [Cognición](cognition.md) (DecisionEngine + Planner) como objetos inmutables
- Instancia el [Orchestrator](pipeline.md) con todas las dependencias y le configura el callback de rollback
- Registra el middleware de [Seguridad](security.md) y los hooks de [Teleología](teleology.md)
- Emite `system.startup` via el [EventBus](events.md) — la primera señal de vida del sistema

---

## AppState — Singleton Global

`src/api/main.py` (434 líneas) define `AppState`, el singleton que contiene todos los componentes. Se accede via `get_state()` con lazy import en cada handler.

---

## Secuencia de Lifespan (19 pasos)

| Paso | Componente | Wiring | Doc |
|------|-----------|--------|-----|
| 1 | `Settings` | `get_settings()` → Pydantic BaseSettings | [config](../config/README.md) |
| 2 | `Database` | `.connect()` → `.initialize_schema()` → `event_bus.set_db_callback()` | [database](database.md) |
| 3 | `PersistenceRepository` | `PersistenceRepository(database)` | [database](database.md) |
| 4 | `ModelRouter` | Wira callback de persistencia de tokens | [integrations](../integrations/README.md) |
| 5 | `AgentCrew` | 5 agentes con persona + governance config | [agents](agents.md) |
| 6 | `MemoryManager` | `.set_model_router()` → `.set_database()` | [memory](memory.md) |
| 7 | `SkillRegistry` | skills.json | [config](../config/README.md) |
| 8 | `TrainingManager` | `await load_corrections()` | [training](../dashboard/training-center.md) |
| 9 | `IdentityManager` | `.load_active_profile()` (DB first, fallback rebuild) | [identity](identity.md) |
| 10 | `DecisionEngine` | Inmutable, `autonomy_level=0`, `identity_profile=profile` | [cognition](cognition.md) |
| 11 | `Planner` | Stateless, sin args | [cognition](cognition.md) |
| 12 | Wire AlignmentEvaluator | `.set_baseline_embedding()` | [evaluation](evaluation.md) |
| 13 | `MiddlewareRegistry` | Pipeline hooks | [pipeline](pipeline.md) |
| 14 | `Orchestrator` | Requiere DecisionEngine + Planner (`isinstance` check) | [pipeline](pipeline.md) |
| 15 | Security Middleware | Registrado en PRE_CLASSIFY, prioridad 10 | [security](security.md) |
| 16 | Teleology | GoalManager, PriorityEngine, RewardModel, PlanExecutor, ConflictDetector, TeleologicalGovernance, BackgroundReevaluator | [teleology](teleology.md) |
| 17 | Teleology Middleware | 3 hooks vía `build_teleology_middleware()` | [teleology](teleology.md) |
| 18 | Memory rollback persistence | Callback wiring | [evaluation](evaluation.md) |
| 19 | Emit `system.startup` | Via EventBus | [events](events.md) |

---

## Shutdown

```
service_log.shutdown()
  → background_reevaluator.stop()
    → memory_manager.close()
      → database.close()
```

---

## HTTP Middleware

| Middleware | Función |
|-----------|---------|
| `ServiceMonitorMiddleware` | Logs requests >5s y errores 500 |
| `CORSMiddleware` | Permite localhost:3000 |

---

## Temas Relacionados

- [Pipeline del Orchestrator](pipeline.md) — Qué pasa después del startup
- [Base de Datos](database.md) — Schema de las 15 tablas
- [Configuración](../config/README.md) — Variables de entorno y archivos YAML/JSON
- [EventBus](events.md) — Sistema de eventos que arranca con `system.startup`
