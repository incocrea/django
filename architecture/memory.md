# Memoria — src/memory/

> [← Cognición](cognition.md) · [Identidad →](identity.md)

**Directorio**: `src/memory/` (1,636 líneas, 4 archivos)

---

## Sobre este documento

Este documento describe el **sistema de memoria de 4 niveles** que le da a Django la capacidad de recordar. Sin memoria, cada conversación empezaría de cero — Django no sabría qué se dijo hace 5 minutos ni qué aprendió el mes pasado. La memoria es lo que transforma a Django de un bot sin contexto a un delegado con historia, conocimiento y continuidad.

### ¿Qué cubre este documento?

Documenta los **4 niveles de memoria** (Working, Episodic, Semantic, Procedural), el `MemoryManager` central (956 líneas) que los coordina, el presupuesto de tokens (~2000 por consulta), el aislamiento per-conversation, la búsqueda híbrida avanzada (RRF + MMR + temporal), el motor de compactación (Background Compaction Engine), y cómo el sistema de identidad se integra con la memoria.

### ¿Cuál es su función en la arquitectura?

La memoria es el **almacén contextual** de Django — donde se guardan las conversaciones, el conocimiento aprendido, los patrones de trabajo y las correcciones de entrenamiento. Provee la base de información que el [Pipeline](pipeline.md) consulta antes de generar cada respuesta. Sin memoria, Django tendría que depender solo del conocimiento pre-entrenado del LLM, que no sabe nada sobre Harold ni sobre interacciones previas.

### ¿Cómo afecta al comportamiento de Django?

- **Working Memory** (RAM efimera): mantiene el hilo de la conversación actual — permite que Django recuerde lo que se dijo hace 5 mensajes en esta misma charla
- **Episodic Memory** (ChromaDB): almacena los logs de conversaciones pasadas — permite que Django recuerde interacciones de hace días o semanas
- **Semantic Memory** (ChromaDB): conocimiento permanente aprendido — lo que Django sabe sobre temas específicos porque se lo enseñaron (via learn-topic, entrenamiento, o almacenamiento manual)
- **Procedural Memory** (SQLite): patrones de trabajo y correcciones — las reglas de comportamiento que Harold le ha enseñado
- El **modo cognitivo** cambia fundamentalmente qué memorias se usan: modo 1 (todo), modo 2 (filtra semántica a `learned_knowledge`), modo 3 (solo memoria, sin LLM)

### ¿Cómo interactúa con las demás piezas?

La memoria se cruza con casi todo el sistema:
- **[Pipeline](pipeline.md) paso 4**: `recall()` consulta los 4 niveles y devuelve contexto priorizado (working > semantic > episodic) hasta el presupuesto de ~2000 tokens
- **[Identidad](identity.md)**: 4 pasos de identidad trabajan sobre la memoria:
  - Paso 2b (Memory Bridge): calcula la afinidad de cada memoria con la identidad de Harold
  - Paso 2c (Retrieval Weighting): re-rankea las memorias usando 80% similitud semántica + 20% afinidad de identidad
  - Paso 3b (Context Weighting): anota las líneas de memoria como `[IDENTITY_ALIGNED]` o `[LOW_IDENTITY_ALIGNMENT]`
  - Paso 6d (Consolidation Weighting): ajusta la importancia de memorias antes de almacenarlas
- **[Evaluación](evaluation.md)**: el MemoryRollbackManager trackea cada operación de memoria con before/after para poder deshacer
- **[Base de Datos](database.md)**: ChromaDB persiste episodic + semantic en disco (`chroma_data/`), SQLite persiste procedural (`data/`), y las operaciones se registran en Postgres
- **[Startup](startup.md)**: el MemoryManager se inicializa antes del Orchestrator para estar listo cuando llegue el primer mensaje
- **[Discord Bot](../integrations/README.md)**: cada canal Discord tiene su propia working memory aislada via `conversation_id`
- Las **Skills** (learn-topic, repo-explorer) escriben directamente en Semantic Memory

---

## 4 Tiers de Memoria

| Tier | Storage | Propósito | Persistencia | Capacidad |
|------|---------|-----------|-------------|-----------|
| **Working** | OrderedDict per conversation_id | Buffer de conversación activa | Efímero (RAM) | Max 20 msgs × 50 sessions, LRU eviction, 1h TTL |
| **Episodic** | ChromaDB `episodic_memory` | Logs de interacción, búsqueda semántica | Disco (`chroma_data/`) | Sin límite |
| **Semantic** | ChromaDB `semantic_memory` | Base de conocimiento, writing samples, RAG | Disco (`chroma_data/`) | Sin límite |
| **Procedural** | SQLite `procedural.db` | Workflows, correcciones de training | Disco (`data/`) | Sin límite |

---

## MemoryManager — `manager.py` (956 líneas)

`MemoryManager` — el coordinador central de memoria.

### Flujo Principal

```
recall(conversation_id) → build_context() → store_interaction(conversation_id)
```

### Budget de Contexto

~2000 tokens máximo para contexto combinado. Prioridad de llenado:

1. **Working memory** (conversación actual) — prioridad máxima
2. **Semantic memory** (conocimiento aprendido) — segunda
3. **Episodic memory** (interacciones pasadas) — tercera

### Aislamiento por Conversación

Cada `conversation_id` obtiene su propia instancia de `WorkingMemory` aislada:

| Fuente | conversation_id | Working Memory |
|--------|----------------|----------------|
| Dashboard chat | UUID generado | Instancia propia |
| Discord #general | `discord_channel_123` | Instancia propia |
| Discord #dev | `discord_channel_456` | Instancia propia |

- `clear_working(conversation_id)` → limpia solo esa conversación
- `clear_all_working()` → limpia todas
- Sesiones inactivas >1h → auto-limpieza via LRU eviction
- `self.working` property → retorna instancia default (compatibilidad legacy)

### Métodos Clave

| Método | Descripción |
|--------|-------------|
| `recall(conversation_id)` | Consulta working + episodic + semantic, respeta budget |
| `build_context()` | Combina resultados del recall en texto para prompt |
| `build_context_with_metadata()` | Versión con IDs para [Context Weighting](identity.md) |
| `store_interaction(conversation_id)` | Guarda en working + episodic |
| `set_model_router()` | Wiring para embeddings |
| `set_database()` | Wiring para operaciones SQL |

---

## Hybrid Search — `hybrid_search.py`

Búsqueda híbrida avanzada que combina:

| Técnica | Descripción |
|---------|-------------|
| **RRF** (Reciprocal Rank Fusion) | Fusiona rankings de múltiples fuentes |
| **MMR** (Maximal Marginal Relevance) | Diversificación de resultados |
| **Temporal Decay** | Prioriza memorias recientes |
| **FTS** (Full-Text Search) | Via tabla `episodic_search` con tsvector + GIN index en [Postgres](database.md) |

---

## Compaction Engine — `compaction.py`

Motor de compactación para consolidación de memoria:
- Resume clusters episódicos en abstracciones semánticas
- Reduce volumen manteniendo información esencial

---

## Integración con Identidad

La memoria pasa por varios módulos de [identidad](identity.md) antes de llegar al prompt:

| Paso | Módulo | Efecto en Memoria |
|------|--------|-------------------|
| 2b | [Memory Bridge](identity.md) | Calcula afinidad identitaria per-memory |
| 2c | [Retrieval Weighting](identity.md) | Re-rankea: 0.8×semantic + 0.2×identity |
| 3b | [Context Weighting](identity.md) | Tags `[ALIGNED]`/`[LOW_ALIGNMENT]` en líneas |
| 6d | [Consolidation Weighting](identity.md) | Ajusta importance pre-storage |

---

## Integración con Cognitive Modes

| Mode | Comportamiento de Memoria |
|------|--------------------------|
| 1 (Full) 🟢 | Recall completo — todos los tiers sin restricción |
| 2 (Memory+LLM) 🟡 | Filtra semantic a solo `learned_knowledge` category |
| 3 (Memory Only) 🔴 | Solo recall — el resultado SE RETORNA DIRECTAMENTE sin LLM |

Ver: [Pipeline](pipeline.md) paso 4 y paso 6.

---

## Operaciones desde Dashboard

El [Memory Lab](../dashboard/memory-lab.md) permite:
- Ver stats de los 4 tiers
- Buscar memorias cross-tier
- Almacenar nuevas memorias semánticas
- Editar contenido y categoría
- Eliminar memorias (con audit trail via [rollback](evaluation.md))
- Bulk delete

Cada operación de memoria se persiste en [Postgres](database.md) tabla `memory_operations`.

---

## Temas Relacionados

- [Pipeline](pipeline.md) — Pasos 4 (recall) y 10 (store)
- [Identidad](identity.md) — Módulos que analizan y re-rankean memorias
- [Base de Datos](database.md) — ChromaDB, SQLite, Postgres persistence
- [Evaluación](evaluation.md) — Memory rollback tracking
- [Memory Lab](../dashboard/memory-lab.md) — Interfaz de gestión
- [Training Center](../dashboard/training-center.md) — Writing samples y corrections como memoria
- [Skill Manager](../dashboard/skill-manager.md) — Learn-topic alimenta semantic memory
