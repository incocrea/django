# Training Center — `/training`

## Información General

El Training Center es la interfaz para entrenar a Django y enseñarle cómo debe comportarse. Soporta correcciones directas (donde se indica explícitamente qué debería haber dicho Django) y carga de archivos con deduplicación semántica (JSON estructurado con writing_sample, style_rule, personality_trait que pasan por un pipeline de parsing, deduplicación contra memoria existente, y almacenamiento en ChromaDB). También permite gestionar correcciones y sugerencias almacenadas. Cada interacción de entrenamiento modifica permanentemente el comportamiento futuro de Django a través de sus sistemas de memoria.

---

## Información Presentada

### Estado de Entrenamiento

Panel superior con 2 cards de estadísticas:
- **Total de correcciones**: Número de correcciones almacenadas en memoria procedural
- **Agentes entrenados**: Número de agentes que han recibido correcciones

Fuente: `GET /training/status`

### Botón "Ejemplo JSON"

Botón en la zona superior que abre un modal con ejemplos completos del formato JSON para cada tipo de entrenamiento (personality_trait, style_rule, writing_sample).

### Correcciones Almacenadas

Tabla de correcciones existentes en memoria procedural. Cada corrección muestra:
- **Situación**: El contexto original
- **Respuesta original**: Lo que Django dijo
- **Corrección**: Lo que debería haber dicho
- **Fecha de creación**: Timestamp

Fuente: `GET /training/corrections`

---

## Acciones

### 1. Enviar Corrección
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/correction` con `{situation, original_response, corrected_response, explanation?}`
- **Impacto en Django**: **Almacena permanentemente** la corrección en SQLite `procedural.db` vía `ProceduralMemory`. A partir de este momento, esta corrección se **inyecta en el paso 5 (Correction Injection) de CADA interacción futura** del orquestador. Django consultará estas correcciones antes de generar cada respuesta, ajustando su comportamiento para no repetir el error. Las correcciones nunca expiran — persisten indefinidamente.

### 2. Subir Archivo de Entrenamiento (JSON con Deduplicación Semántica)
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/upload-samples` con archivos JSON estructurados
- **Impacto en Django**: El backend procesa el archivo a través de un pipeline de 3 etapas:
  1. **Parser** (`training_parser.py`): Extrae items estructurados del JSON — soporta categorías `writing_sample`, `style_rule`, y `personality_trait` (acepta campo `"category"` o `"type"`)
  2. **Deduplicador** (`deduplicator.py`): Compara cada item contra la memoria semántica existente via cosine similarity (EmbeddingRouter). Umbrales por categoría — items duplicados se descartan
  3. **Processor** (`processor.py`): Almacena items únicos en ChromaDB semantic memory con sus categorías correspondientes
  - Las muestras alimentan `AlignmentEvaluator`, `IdentityManager.build_profile()`, y `IdentityMemoryBridge` (Phase 6A)

### 3. Editar Corrección
- **Tipo**: API Call
- **Comportamiento**: Llama `PUT /training/corrections/{id}` con datos actualizados
- **Impacto en Django**: Modifica la corrección existente en SQLite. Los cambios se reflejan inmediatamente en las inyecciones futuras del paso 5 del pipeline.

### 4. Eliminar Corrección
- **Tipo**: API Call
- **Comportamiento**: Llama `DELETE /training/corrections/{id}`
- **Impacto en Django**: Elimina permanentemente la corrección de SQLite. Django dejará de recibir esa corrección en el paso 5 de interacciones futuras.

### 5. Eliminar Sugerencias
- **Tipo**: API Call
- **Comportamiento**: Llama `DELETE /training/suggestions`
- **Impacto en Django**: Limpia sugerencias generadas automáticamente.

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/training` |
| **Archivo** | `dashboard/app/training/page.tsx` (741 líneas) |
| **APIs al cargar** | `GET /training/status`, `GET /training/corrections` |
| **APIs de escritura** | `POST /training/session/start`, `POST /training/session/end`, `POST /training/correction`, `POST /training/upload-samples`, `PUT /training/corrections/{id}`, `DELETE /training/corrections/{id}`, `DELETE /training/suggestions` |
| **Total endpoints usados** | 14 distintos (backend mantiene endpoints legacy de free conversation e interview) |
| **Estado** | React `useState` para sesión activa, modo, historial, correcciones |
| **Upload** | HTML5 File API con FormData — JSON estructurado con deduplicación semántica |
| **Almacenamiento destino** | Correcciones → SQLite (procedural.db), Training data → ChromaDB (semantic) |

### Backend: Módulo Training (5 archivos, ~1,522 líneas)

| Archivo | Líneas | Función |
|---------|--------|--------|
| `manager.py` | 520 | TrainingManager — gestión de sesiones, CRUD de correcciones, upload de writing samples |
| `training_parser.py` | 406 | Parser JSON para datos de entrenamiento estructurados (writing_sample, style_rule, personality_trait). Acepta claves `"category"` y `"type"` |
| `deduplicator.py` | 224 | Motor de deduplicación semántica — umbrales de similitud coseno por categoría vía EmbeddingRouter |
| `processor.py` | 371 | Pipeline end-to-end — parse → deduplicar → almacenar en ChromaDB |
| `__init__.py` | — | Package exports |

**Última actualización**: 2026-02-23
