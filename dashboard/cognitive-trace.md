# Cognitive Trace — `/trace`

## Información General

El Cognitive Trace viewer visualiza el pipeline de procesamiento interno del orquestador como un pipeline horizontal de 8 columnas usando CSS Grid. Cada vez que Doe procesa un mensaje, el `TraceCollector` registra cada paso del pipeline como un nodo con inputs, outputs, métricas, y latencia. Esta página permite inspeccionar esos pipelines, entender exactamente qué pasos ejecutó Doe para generar una respuesta, ver los prompts exactos enviados al LLM, y ejecutar replays que generan nuevos traces. Es la herramienta principal de observabilidad y debugging del sistema.

---

## Información Presentada

### Sidebar (Panel izquierdo, 288px, colapsable)

#### Campo de Replay
- Input de texto + botón Send para enviar un mensaje que generará un nuevo trace
- Respuesta de Doe mostrada debajo del input tras completar el replay

#### Lista de Traces
Lista scrollable de los últimos 50 traces generados. Cada item muestra:
- **Mensaje del usuario**: Primeros 40 caracteres del input original
- **Node count**: Badge con número de nodos en el trace
- **Latencia total**: Tiempo total del pipeline
- **Category**: Badge de categoría (CONVERSATION, BUSINESS, TECHNICAL, etc.)
- **Provider**: Nombre del modelo/provider usado
- **Timestamp**: Fecha/hora de creación
- **Botón Delete** (aparece al hover): X para eliminar ese trace individual

El trace seleccionado se resalta con borde de acento.

### Pipeline View (CSS Grid 8 columnas)

Visualizacón horizontal del pipeline de procesamiento organizado en **8 columnas funcionales** — cada columna representa un grupo con nodos apilados verticalmente y accordion expand/collapse:

#### Columnas (8 bloques funcionales)

Los nodos se organizan en columnas mediante CSS Grid `repeat(8, minmax(180px, 1fr))`. Cada columna corresponde a un `parentId` del backend:

| # | Grupo | Color | Nodos que contiene |
|---|-------|-------|-------------------|
| 0 | Pre-Pipeline | Indigo (#6366f1) | input, branch |
| 1 | Classification | Cyan (#06b6d4) | classify, decision_engine |
| 2 | Identity Pre-Gen | Purple (#a855f7) | identity_decision_modulation, identity_confidence, identity_autonomy, identity_bias |
| 3 | Planning & Skills | Amber (#f59e0b) | planner, skill_execution |
| 4 | Context Assembly | Emerald (#10b981) | memory_recall, identity_memory_bridge, identity_retrieval_weight, correction, identity_context_weight, identity_prompt_inject, prompt_build, compaction |
| 5 | LLM Generation | Red (#ef4444) | llm_generate, multi_agent, identity_review, governance_review |
| 6 | Persistence & Eval | Blue (#3b82f6) | identity_consolidation, memory_store, evaluation, identity_drift, identity_policy, identity_feedback, output |
| 7 | Post-Pipeline | Pink (#ec4899) | identity_health_monitor, identity_health_regulation, identity_evolution, identity_shadow, identity_version_candidate |

Cada columna muestra: **header** con barra de color + label + conteo de nodos + latencia agregada. Columnas sin nodos se muestran en gris con texto "empty". **Flechas inter-columna** (→) separan las columnas visualmente.

**Nodos con LLM**: Badge violeta 🧠 (Brain icon) en el header del nodo cuando `uses_llm=true`. El backend detecta esto via `metadata.llm_details` o `metrics.provider`.

**Nodos con Embedding**: Badge 📐 EMB en el header del nodo cuando `uses_embedding=true`. Indica uso de EmbeddingRouter (Qwen3-Embedding-8B).

**Nodos de Skill**: Badge 🔧 SKILL en sub-nodos de skill execution. Indica pasos de ejecución de habilidades reportados via `SkillReport.to_trace_nodes()`. Incluye skills nativas (`learn-topic`, `repo-explorer`) y skills dinámicas autogeneradas por SkillForge cuando retornan `SkillResult(report=...)`.

**Nota sobre `skill_execution`**: Aunque usa `node_type="llm_generate"`, un override por `node_id` lo asigna al grupo Planning & Skills (no Generation).

#### Barra de Stats (Panel superior central)
- Total de nodos
- Total de edges
- Latencia total (formateada como segundos o milisegundos)
- Badge de categoría
- Provider/modelo usado

#### Nodos del Grafo (33 tipos registrados, 33 canónicos del backend)

Cada nodo `PipelineNode` muestra:
- **Header** (siempre visible): Barra de color izquierda (color de columna) + icono del tipo + label + status icon + badge de latencia + 🧠 LLM badge (si aplica) + chevron expandir/colapsar
- **Summary** (colapsado): `processingSummary` texto, máximo 2 líneas
- **LLM Details Panel** (expandido, si el nodo usa LLM): Sección violeta con badges de provider coloreados (Gemini=azul, Groq=naranja), tokens usados, latencia, estimación de costo ($0.15/1M Gemini, $0.05/1M Groq), indicador de fallback. Soporta arrays para nodos multi-agente.
- **Expandido**: Summary + LLM Details (si aplica) + Input (bloque `<pre>` cian) + Output (bloque `<pre>` esmeralda) + Metrics (badges ámbar para cada key/value) + Error (bloque rojo si aplica)
- **Accordion**: Click en header toglea expand/collapse

**Status icons de nodos**:
- Completed: CheckCircle2 verde
- Failed: XCircle rojo (+ borde rojo override)
- Running: Clock ámbar pulsante
- Skipped: SkipForward gris
- Pending: Clock gris

**Tabla completa de los 33 tipos de nodo registrados**:

| # | Tipo | Icono | Color Borde | Paso del Pipeline |
|---|------|-------|-------------|-------------------|
| 1 | `trace_input` | MessageSquare | blue-500 | Input del usuario |
| 2 | `classify` | GitBranch | purple-500 | Paso 1 — Clasificación |
| 3 | `decision_engine` | Scale | orange-500 | Paso 2 — Decision Engine |
| 4 | `planner` | ListChecks | sky-500 | Paso 3 — Planner |
| 5 | `skill_execution` | Wrench | amber-500 | Paso 1d — Skill Execution |
| 6 | `memory_recall` | Database | cyan-500 | Paso 4 — Memory Recall |
| 6 | `correction` | FileText | amber-500 | Paso 5 — Correction Injection |
| 7 | `prompt_build` | FileText | indigo-500 | Paso 6 — Prompt Build |
| 8 | `llm_generate` | Brain | emerald-500 | Paso 7 — LLM Generate |
| 9 | `identity_review` | User | rose-500 | Paso 8 — Identity Review |
| 10 | `governance_review` | Shield | yellow-500 | Paso 9 — Governance Review |
| 11 | `memory_store` | Database | teal-500 | Paso 10 — Memory Store |
| 12 | `evaluation` | Zap | violet-500 | Paso 10 — Evaluation |
| 13 | `trace_output` | ArrowDown | green-500 | Output final |
| 14 | `branch` | AlertTriangle | red-500 | Bifurcación |
| 15 | `multi_agent` | Users | fuchsia-500 | Multi-agente |
| 16 | `compaction` | Minimize2 | slate-500 | Memory Compaction |
| 17 | `identity_decision_modulation` | Fingerprint | pink-500 | Paso 3c (Phase 6C) |
| 18 | `identity_confidence` | Gauge | pink-400 | Paso 3d (Phase 6D) |
| 19 | `identity_autonomy` | SlidersHorizontal | pink-300 | Paso 3e (Phase 7A) |
| 20 | `identity_bias` | Palette | fuchsia-400 | Paso 3f (Phase 8A) |
| 21 | `identity_memory_bridge` | Link | rose-400 | Paso 2b (Phase 6A) |
| 22 | `identity_retrieval_weight` | ArrowUpDown | rose-300 | Paso 2c (Phase 7B) |
| 23 | `identity_context_weight` | PenTool | pink-500/40 | Paso 3b (Phase 6B) |
| 24 | `identity_prompt_inject` | Syringe | fuchsia-500/40 | Paso 3g (Phase 8B) |
| 25 | `identity_consolidation` | Layers | pink-600 | Paso 6d (Phase 7C) |
| 26 | `identity_drift` | ShieldAlert | red-400 | Paso 7c (Phase 5A) |
| 27 | `identity_policy` | Shield | red-500/40 | Paso 7d (Phase 5B) |
| 28 | `identity_feedback` | MessageCircle | orange-400 | Paso 7e (Phase 5C) |
| 29 | `identity_health_monitor` | HeartPulse | lime-500 | Paso 9a (Phase 9A) |
| 30 | `identity_health_regulation` | Activity | lime-400 | Paso 9b (Phase 9B) |
| 31 | `identity_evolution` | TrendingUp | amber-400 | Paso 9c (Phase 10A) |
| 32 | `identity_shadow` | FlaskConical | slate-400 | Paso 9d (Phase 10B) |
| 33 | `identity_version_candidate` | GitCommit | emerald-400 | Paso 9e (Phase 10C) |

**Nota**: El backend emite 33 tipos canónicos de nodo. `PipelineView` agrupa los nodos por `parentId` en columnas y `PipelineNode` renderiza cada uno con su icono, color y secciones expandibles. Las constantes de tipos, iconos y colores están centralizadas en `trace-constants.ts`.

#### Flechas Inter-Columna
- Flechas `→` entre columnas adyacentes que contienen nodos
- Centradas verticalmente en el espacio entre columnas
- Color: slate-600

#### Columnas Vacías
- Columnas sin nodos se muestran con opacidad reducida (40%)
- Header en gris con texto "empty" en itálica
- No se ocultan — las 8 columnas siempre son visibles para mantener contexto del pipeline completo

### Botón View Prompt (Panel superior derecho)

Botón índigo brillante con icono Eye. Solo aparece cuando el nodo `prompt_build` tiene `refined_prompt` o `user_prompt` en su output.

### Modal de Prompt Viewer

Overlay con backdrop blur que muestra el prompt completo:
- **Header**: Icono Eye, título, badge con conteo de caracteres, botón Copy (con animación check), botón Close
- **Tabs**: "User Prompt" (ámbar) y "System Prompt" (índigo)
- **Contenido**: Bloque `<pre>` monospace con el texto completo del prompt, scrollable

---

## Acciones

### 1. Seleccionar Trace
- **Tipo**: API Call
- **Comportamiento**: Click en un trace de la sidebar. Llama `GET /trace/{interactionId}`
- **Impacto en Doe**: **Solo lectura** — carga el grafo del trace seleccionado en el canvas.

### 2. Replay Message
- **Tipo**: API Call
- **Comportamiento**: Escribir un mensaje en el campo de replay y presionar Enter o el botón Send. Llama `POST /trace/replay` con `{message, conversation_id}`
- **Impacto en Doe**: **Ejecuta el pipeline completo del orquestador** — idéntico a enviar un mensaje en `/chat`. Genera un nuevo trace, crea memorias, evaluaciones, persiste en Postgres. La diferencia es que el nuevo trace se carga automáticamente en el canvas y la respuesta se muestra directamente debajo del input. Refresca la lista de traces.

### 3. Delete Single Trace
- **Tipo**: API Call con confirmación
- **Comportamiento**: Hover sobre trace → botón X → confirmar en `ConfirmDialog`. Llama `DELETE /trace/{interactionId}`
- **Impacto en Doe**: Elimina el trace del `TraceStore` en memoria. Si el trace eliminado estaba seleccionado, limpia el canvas. **No elimina** la interacción, memorias, ni evaluaciones asociadas — solo el trace graph.

### 4. Delete All Traces
- **Tipo**: API Call con confirmación
- **Comportamiento**: Botón Trash2 en header de lista → confirmar. Llama `DELETE /trace`
- **Impacto en Doe**: Elimina TODOS los traces del `TraceStore` en memoria. Limpia canvas y lista. No afecta datos en Postgres (interactions, evaluations, etc.), solo los graphs en memoria.

### 5. Load Latest
- **Tipo**: API Call
- **Comportamiento**: Llama `GET /trace/latest/graph`
- **Impacto en Doe**: Solo lectura — carga el trace más reciente

### 6. View Prompt
- **Tipo**: Estado local
- **Comportamiento**: Abre el modal de prompt viewer. Extrae `refined_prompt` y `user_prompt` del output del nodo `prompt_build`
- **Impacto en Doe**: Ninguno — operación puramente visual

### 7. Copy Prompt
- **Tipo**: Clipboard
- **Comportamiento**: Copia el texto del prompt activo al portapapeles
- **Impacto en Doe**: Ninguno

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/trace` |
| **Archivos** | `dashboard/app/trace/page.tsx` (~567 ln), `dashboard/components/trace/pipeline-view.tsx` (~189 ln), `dashboard/components/trace/pipeline-node.tsx` (~381 ln), `dashboard/components/trace/trace-constants.ts` (~285 ln) |
| **Archivos legacy** | `trace-node.tsx` (512 ln), `group-node.tsx` (57 ln) — dead code, kept for reference |
| **Líneas totales** | ~1,422 líneas (page + pipeline-view + pipeline-node + trace-constants) |
| **Librería** | CSS Grid nativo (no @xyflow/react) |
| **APIs al cargar** | `GET /trace/list?limit=50`, `GET /trace/latest/graph` |
| **APIs de escritura** | `POST /trace/replay`, `DELETE /trace/{id}`, `DELETE /trace` |
| **Total endpoints** | 6 |
| **Node types registrados** | 33 (16 core + 17 identity), organizados en 8 columnas funcionales |
| **Columnas** | 8 (pre_pipeline, classification, identity_pre, planning, context, generation, persistence, post_pipeline) |
| **Layout** | CSS Grid horizontal `repeat(8, minmax(180px, 1fr))` — columnas siempre visibles, vacías en gris |
| **LLM Indicator** | Badge violeta 🧠 en header del nodo cuando `uses_llm=true` (detectado via `metadata.llm_details` o `metrics.provider`). Badge 📐 EMB para embedding (detectado via `metadata.embedding_details` o `uses_embedding`). Badge 🔧 SKILL para sub-nodos de skill |
| **Memoization** | `PipelineNode` wrapped en `React.memo` |
| **LLM Details** | Panel dedicado con badges de provider coloreados, tokens, latencia, costo estimado |
| **EMB Details** | Panel dedicado (cyan) con provider, modelo, dimensiones, items embebidos, latencia — visible al expandir nodo con badge 📐 EMB |
| **Estado** | React `useState` para graphData, traceList, selectedTraceId, replayInput |
| **Constantes** | `trace-constants.ts`: NODE_ICONS (37), NODE_COLORS (37), GROUP_META (8), GROUP_ORDER, NODE_TYPE_GROUP, EmbeddingDetail interface, estimateCost() |

**Última actualización**: 2025-06-24
