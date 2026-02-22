# Training Center — `/training`

## Información General

El Training Center es la interfaz para entrenar a Django y enseñarle cómo debe comportarse. Soporta 3 modos de entrenamiento: correcciones directas (donde se indica explícitamente qué debería haber dicho Django), conversación libre (donde Django extrae rasgos de personalidad automáticamente de la conversación), e entrevista guiada (cuestionario estructurado de 20+ preguntas que construye un perfil de personalidad). También permite subir muestras de escritura del principal y gestionar correcciones y sugerencias almacenadas. Cada interacción de entrenamiento modifica permanentemente el comportamiento futuro de Django a través de sus sistemas de memoria.

---

## Información Presentada

### Estado de Entrenamiento

Panel superior que muestra:
- **Sesión activa**: Indicador de si hay una sesión de entrenamiento en curso, su modo, y duración
- **Total de sesiones**: Número histórico de sesiones completadas
- **Total de correcciones**: Número de correcciones almacenadas en memoria procedural
- **Total de muestras**: Número de writing samples procesadas

Fuente: `GET /training/status`

### Historial de Sesiones

Lista de sesiones pasadas con fecha, modo, duración, y número de interacciones.

Fuente: `GET /training/history`

### Correcciones Almacenadas

Tabla de correcciones existentes en memoria procedural. Cada corrección muestra:
- **Situación**: El contexto original
- **Respuesta original**: Lo que Django dijo
- **Corrección**: Lo que debería haber dicho
- **Fecha de creación**: Timestamp

Fuente: `GET /training/corrections`

### Cuestionario de Entrevista Guiada

Cuando se activa el modo de entrevista, muestra una serie de 20+ preguntas predefinidas diseñadas para extraer rasgos de personalidad del principal:

Fuente: `GET /training/interview/questions`

---

## Acciones

### 1. Iniciar Sesión — Modo Corrección
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/session/start` con `{mode: "correction"}`
- **Impacto en Django**: Crea una sesión de entrenamiento activa en `TrainingManager`. Emite evento `training.session_started`. Solo puede haber una sesión activa a la vez.

### 2. Enviar Corrección
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/correction` con `{situation, original_response, corrected_response, explanation?}`
- **Impacto en Django**: **Almacena permanentemente** la corrección en SQLite `procedural.db` vía `ProceduralMemory`. A partir de este momento, esta corrección se **inyecta en el paso 5 (Correction Injection) de CADA interacción futura** del orquestador. Django consultará estas correcciones antes de generar cada respuesta, ajustando su comportamiento para no repetir el error. Las correcciones nunca expiran — persisten indefinidamente.

### 3. Iniciar Sesión — Conversación Libre
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/session/start` con `{mode: "free_conversation"}`
- **Impacto en Django**: Crea sesión activa. Los mensajes se envían via `POST /training/exchange`.

### 4. Enviar Mensaje (Conversación Libre)
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/exchange` con `{message, conversation_id}`
- **Impacto en Django**: Django responde al mensaje Y simultáneamente analiza el contenido con el LLM para **extraer rasgos de personalidad implícitos** del principal. Los rasgos extraídos se almacenan en ChromaDB como memoria semántica, enriqueciendo el perfil de identidad con datos derivados de la conversación natural. Cada intercambio puede revelar patrones de comunicación, preferencias, y valores del principal.

### 5. Iniciar Sesión — Entrevista Guiada
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/session/start` con `{mode: "guided_interview"}`
- **Impacto en Django**: Carga las preguntas predefinidas y comienza el cuestionario estructurado.

### 6. Responder Pregunta de Entrevista
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/interview/answer` con `{question_id, answer}`
- **Impacto en Django**: La respuesta es analizada por el LLM para **extraer rasgos de personalidad**. Los rasgos extraídos se almacenan en ChromaDB como memoria semántica. Las preguntas están diseñadas para cubrir aspectos específicos de la personalidad (valores, estilo de comunicación, toma de decisiones, expertise).

### 7. Subir Muestras de Escritura
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/upload-samples` con archivos de texto
- **Impacto en Django**: El backend **chunquea** cada archivo en segmentos de ~500 palabras y los almacena en ChromaDB como memoria semántica con categoría `writing_sample`. Estos chunks son usados por:
  - `AlignmentEvaluator`: Para comparar el estilo de Django vs el estilo real del principal
  - `IdentityManager.build_profile()`: Los top 3 writing samples se incluyen en el texto de identidad para computar el `baseline_embedding`
  - `IdentityMemoryBridge` (Phase 6A): Las muestras contribuyen al recall de memoria con afinidad de identidad

### 8. Finalizar Sesión
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /training/session/end`
- **Impacto en Django**: Cierra la sesión activa, registra duración y estadísticas. Emite evento `training.session_ended`.

### 9. Editar Corrección
- **Tipo**: API Call
- **Comportamiento**: Llama `PUT /training/corrections/{id}` con datos actualizados
- **Impacto en Django**: Modifica la corrección existente en SQLite. Los cambios se reflejan inmediatamente en las inyecciones futuras del paso 5 del pipeline.

### 10. Eliminar Corrección
- **Tipo**: API Call
- **Comportamiento**: Llama `DELETE /training/corrections/{id}`
- **Impacto en Django**: Elimina permanentemente la corrección de SQLite. Django dejará de recibir esa corrección en el paso 5 de interacciones futuras.

### 11. Eliminar Sugerencias
- **Tipo**: API Call
- **Comportamiento**: Llama `DELETE /training/suggestions`
- **Impacto en Django**: Limpia sugerencias generadas automáticamente por el LLM durante sesiones de conversación libre.

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/training` |
| **Archivo** | `dashboard/app/training/page.tsx` (1,125 líneas, todo en un archivo) |
| **APIs al cargar** | `GET /training/status`, `GET /training/history`, `GET /training/corrections` |
| **APIs de escritura** | `POST /training/session/start`, `POST /training/session/end`, `POST /training/correction`, `POST /training/exchange`, `POST /training/upload-samples`, `GET /training/interview/questions`, `POST /training/interview/answer`, `PUT /training/corrections/{id}`, `DELETE /training/corrections/{id}`, `DELETE /training/suggestions` |
| **Total endpoints usados** | 14 distintos |
| **Estado** | React `useState` para sesión activa, modo, historial, correcciones, preguntas |
| **Upload** | HTML5 File API con FormData para writing samples |
| **Almacenamiento destino** | Correcciones → SQLite (procedural.db), Escritura/Rasgos → ChromaDB (semantic) |

**Última actualización**: 2025-06-23
