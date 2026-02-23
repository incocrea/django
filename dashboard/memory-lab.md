# Memory Lab — `/memory`

## Información General

El Memory Lab es la interfaz para inspeccionar, buscar, crear, editar y eliminar los contenidos de la memoria de Django. El sistema de memoria tiene 4 tiers (Working, Episodic, Semantic, Procedural), cada uno con diferente propósito, persistencia y almacenamiento. Desde aquí el principal puede ver estadísticas de cada tier, buscar memorias por texto y tier, almacenar nuevos conocimientos manualmente en memoria semántica, editar memorias existentes, y eliminar memorias individuales o en bloque. Las operaciones aquí afectan directamente qué información recuerda Django en futuras conversaciones.

---

## Información Presentada

### Tarjetas de Estadísticas (4 tiers)

| Tier | Almacenamiento | Qué contiene | Persistencia |
|------|---------------|--------------|--------------|
| **Working** | OrderedDict en memoria (por conversation_id) | Buffer de conversación activa (últimos 20 mensajes por conversación, max 50 sesiones, LRU eviction, 1h TTL) | Efímera — se pierde al reiniciar |
| **Episodic** | ChromaDB `episodic_memory` | Logs de interacciones pasadas, searchable por similitud semántica | Disco (`chroma_data/`) |
| **Semantic** | ChromaDB `semantic_memory` | Base de conocimiento, writing samples, learned_knowledge, RAG | Disco (`chroma_data/`) |
| **Procedural** | SQLite `procedural.db` | Workflows, patrones, correcciones de entrenamiento | Disco (`data/`) |

Cada tarjeta muestra el número total de items almacenados en ese tier.

Fuente: `GET /memory/stats`

### Resultados de Búsqueda

Lista de memorias que coinciden con la query de búsqueda. Cada resultado muestra:
- **Contenido**: Texto de la memoria (texto completo)
- **Tier**: De qué tier proviene (episodic/semantic)
- **Distancia**: Score de similitud semántica con la query (menor = más similar)
- **Metadata**: Información adicional (categoría, timestamp, source)
- **ID**: Identificador único en ChromaDB

### Formulario de Almacenamiento

Campos para crear una nueva memoria semántica manualmente:
- **Content**: Texto del conocimiento a almacenar
- **Category**: Categoría opcional (ej: `learned_knowledge`, `writing_sample`)
- **Source**: Fuente del conocimiento (ej: "manual entry", "external document")

---

## Acciones

### 1. Buscar Memorias
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /memory/search` con `{query, tier?: "episodic"|"semantic", limit?: number}`
- **Impacto en Django**: **Solo lectura** — ejecuta búsqueda por similitud semántica (cosine similarity) en ChromaDB. No modifica ningún dato. El filtro de tier permite buscar solo en episodic, solo en semantic, o en ambos. Límite configurable (default 10).

### 2. Almacenar Memoria Semántica
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /memory/semantic/store` con `{content, category?, source?}`
- **Impacto en Django**: **Crea una nueva entrada** en ChromaDB `semantic_memory`. ChromaDB computa automáticamente el embedding del contenido (Qwen3-Embedding-8B, 4096-dim via EmbeddingRouter). Esta memoria estará disponible en el recall del paso 4 del pipeline para todas las conversaciones futuras. Si la categoría es `learned_knowledge`, será visible en modos cognitivos 2 y 3. Si es otra categoría, solo será visible en modo 1 (Full).

### 3. Editar Memoria Semántica
- **Tipo**: API Call
- **Comportamiento**: Llama `PUT /memory/semantic/{id}` con `{content, metadata?}`
- **Impacto en Django**: Modifica el contenido de una memoria existente en ChromaDB. Se **recomputa el embedding** — la proximidad semántica de esta memoria a futuras queries puede cambiar significativamente. El `MemoryRollbackManager` registra el estado anterior (before-state) para posible undo.

### 4. Eliminar Memoria Semántica
- **Tipo**: API Call
- **Comportamiento**: Llama `DELETE /memory/semantic/{id}`
- **Impacto en Django**: Elimina permanentemente la memoria de ChromaDB. Django ya no recordará esta información. El `MemoryRollbackManager` registra la operación con el contenido completo eliminado para posible undo desde el Evaluation Dashboard.

### 5. Eliminar Memoria Episódica
- **Tipo**: API Call
- **Comportamiento**: Llama `DELETE /memory/episodic/{id}`
- **Impacto en Django**: Elimina un log de interacción pasada de ChromaDB. Afecta qué contexto histórico recuerda Django. Operación registrada por `MemoryRollbackManager`.

### 6. Eliminar en Bloque (Bulk Delete)
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /memory/bulk-delete` con `{ids: string[], tier: "semantic"|"episodic"}`
- **Impacto en Django**: Elimina múltiples memorias de un solo tier en una operación. Cada eliminación se registra individualmente en `MemoryRollbackManager`. Útil para limpiar datos de testing o memorias obsoletas.

### 7. Limpiar Working Memory
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /memory/working/clear` con `{conversation_id?}`
- **Impacto en Django**: Si se especifica `conversation_id`, limpia solo el buffer de working memory de esa conversación. Si no, limpia todos los buffers. El efecto es que Django "olvida" el contexto de la conversación activa — la próxima respuesta no tendrá referencia a mensajes anteriores de esa sesión.

### Mecanismo de Undo

Las operaciones destructivas (delete, edit) quedan registradas en `MemoryRollbackManager` con snapshots del estado anterior. El undo se ejecuta desde el **Evaluation Dashboard** (`/evaluation` → tab Rollback), no desde esta página.

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/memory` |
| **Archivo** | `dashboard/app/memory/page.tsx` (692 líneas) |
| **APIs al cargar** | `GET /memory/stats` |
| **APIs de lectura** | `POST /memory/search` |
| **APIs de escritura** | `POST /memory/semantic/store`, `PUT /memory/semantic/{id}`, `DELETE /memory/semantic/{id}`, `DELETE /memory/episodic/{id}`, `POST /memory/bulk-delete`, `POST /memory/working/clear` |
| **Total endpoints usados** | 8 |
| **Estado** | React `useState` para stats, resultados de búsqueda, formulario, modo edición |
| **Auto-refresh** | Re-busca automáticamente cuando cambia el filtro de tier |
| **Backend module** | `src/memory/manager.py` (~810 líneas), ChromaDB local en `chroma_data/`, SQLite en `data/procedural.db` |
| **Iconos** | lucide-react: Database, Search, Plus, Edit, Trash2, RefreshCw |

**Última actualización**: 2025-06-23
