# Cognitive Trace — `/trace`

## Información General

El Cognitive Trace viewer visualiza el pipeline de procesamiento interno del orquestador como un grafo dirigido interactivo usando React Flow. Cada vez que Django procesa un mensaje, el `TraceCollector` registra cada paso del pipeline como un nodo con inputs, outputs, métricas, y latencia. Esta página permite inspeccionar esos grafos, entender exactamente qué pasos ejecutó Django para generar una respuesta, ver los prompts exactos enviados al LLM, y ejecutar replays que generan nuevos traces. Es la herramienta principal de observabilidad y debugging del sistema.

---

## Información Presentada

### Sidebar (Panel izquierdo, 288px, colapsable)

#### Campo de Replay
- Input de texto + botón Send para enviar un mensaje que generará un nuevo trace
- Respuesta de Django mostrada debajo del input tras completar el replay

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

### Canvas Principal (React Flow)

Grafo dirigido interactivo que muestra el pipeline de procesamiento:

#### Barra de Stats (Panel superior central)
- Total de nodos
- Total de edges
- Latencia total (formateada como segundos o milisegundos)
- Badge de categoría
- Provider/modelo usado

#### Nodos del Grafo (32 tipos registrados, 32 canónicos del backend)

Cada nodo `TraceNodeComponent` muestra:
- **Header** (siempre visible): Icono del tipo + label + status icon + badge de latencia + chevron expandir/colapsar
- **Summary** (colapsado): `processingSummary` texto, máximo 2 líneas
- **Expandido**: Summary + Input (bloque `<pre>` cian) + Output (bloque `<pre>` esmeralda) + Metrics (badges ámbar para cada key/value) + Error (bloque rojo si aplica)

**Status icons de nodos**:
- Completed: CheckCircle2 verde
- Failed: XCircle rojo (+ borde rojo override)
- Running: Clock ámbar pulsante
- Skipped: SkipForward gris
- Pending: Clock gris

**Tabla completa de los 32 tipos de nodo registrados**:

| # | Tipo | Icono | Color Borde | Paso del Pipeline |
|---|------|-------|-------------|-------------------|
| 1 | `trace_input` | MessageSquare | blue-500 | Input del usuario |
| 2 | `classify` | GitBranch | purple-500 | Paso 1 — Clasificación |
| 3 | `decision_engine` | Scale | orange-500 | Paso 2 — Decision Engine |
| 4 | `planner` | ListChecks | sky-500 | Paso 3 — Planner |
| 5 | `memory_recall` | Database | cyan-500 | Paso 4 — Memory Recall |
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
| 16 | `identity_decision_modulation` | Fingerprint | pink-500 | Paso 3c (Phase 6C) |
| 17 | `identity_confidence` | Gauge | pink-400 | Paso 3d (Phase 6D) |
| 18 | `identity_autonomy` | SlidersHorizontal | pink-300 | Paso 3e (Phase 7A) |
| 19 | `identity_bias` | Palette | fuchsia-400 | Paso 3f (Phase 8A) |
| 20 | `identity_memory_bridge` | Link | rose-400 | Paso 2b (Phase 6A) |
| 21 | `identity_retrieval_weight` | ArrowUpDown | rose-300 | Paso 2c (Phase 7B) |
| 22 | `identity_context_weight` | PenTool | pink-500/40 | Paso 3b (Phase 6B) |
| 23 | `identity_prompt_inject` | Syringe | fuchsia-500/40 | Paso 3g (Phase 8B) |
| 24 | `identity_consolidation` | Layers | pink-600 | Paso 6d (Phase 7C) |
| 25 | `identity_drift` | ShieldAlert | red-400 | Paso 7c (Phase 5A) |
| 26 | `identity_policy` | Shield | red-500/40 | Paso 7d (Phase 5B) |
| 27 | `identity_feedback` | MessageCircle | orange-400 | Paso 7e (Phase 5C) |
| 28 | `identity_health_monitor` | HeartPulse | lime-500 | Paso 9a (Phase 9A) |
| 29 | `identity_health_regulation` | Activity | lime-400 | Paso 9b (Phase 9B) |
| 30 | `identity_evolution` | TrendingUp | amber-400 | Paso 9c (Phase 10A) |
| 31 | `identity_shadow` | FlaskConical | slate-400 | Paso 9d (Phase 10B) |
| 32 | `identity_version_candidate` | GitCommit | emerald-400 | Paso 9e (Phase 10C) |

**Nota**: El backend emite 32 tipos canónicos de nodo. La page remapea `input` → `trace_input` y `output` → `trace_output` para evitar estilos built-in de React Flow, resultando en 32 keys en el registro de `nodeTypes`.

#### Edges
- **Tipo**: `smoothstep` para todos
- **Color por tipo**: `conditional` → ámbar, `fallback` → rojo, default → cian
- **Estilo**: Stroke cian rgba(34,211,238,0.4), width 2, ArrowClosed marker

#### Background y Controles
- **Background**: Grid de puntos cian al 5% opacidad
- **Zoom**: Min 0.3, max 1.5
- **MiniMap**: Nodos coloreados por tipo (13 colores explícitos + slate fallback)
- **Controles**: Zoom/fit (bottom-left) con estilo lab theme

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
- **Impacto en Django**: **Solo lectura** — carga el grafo del trace seleccionado en el canvas.

### 2. Replay Message
- **Tipo**: API Call
- **Comportamiento**: Escribir un mensaje en el campo de replay y presionar Enter o el botón Send. Llama `POST /trace/replay` con `{message, conversation_id}`
- **Impacto en Django**: **Ejecuta el pipeline completo del orquestador** — idéntico a enviar un mensaje en `/chat`. Genera un nuevo trace, crea memorias, evaluaciones, persiste en Postgres. La diferencia es que el nuevo trace se carga automáticamente en el canvas y la respuesta se muestra directamente debajo del input. Refresca la lista de traces.

### 3. Delete Single Trace
- **Tipo**: API Call con confirmación
- **Comportamiento**: Hover sobre trace → botón X → confirmar en `ConfirmDialog`. Llama `DELETE /trace/{interactionId}`
- **Impacto en Django**: Elimina el trace del `TraceStore` en memoria. Si el trace eliminado estaba seleccionado, limpia el canvas. **No elimina** la interacción, memorias, ni evaluaciones asociadas — solo el trace graph.

### 4. Delete All Traces
- **Tipo**: API Call con confirmación
- **Comportamiento**: Botón Trash2 en header de lista → confirmar. Llama `DELETE /trace`
- **Impacto en Django**: Elimina TODOS los traces del `TraceStore` en memoria. Limpia canvas y lista. No afecta datos en Postgres (interactions, evaluations, etc.), solo los graphs en memoria.

### 5. Load Latest
- **Tipo**: API Call
- **Comportamiento**: Llama `GET /trace/latest/graph`
- **Impacto en Django**: Solo lectura — carga el trace más reciente

### 6. View Prompt
- **Tipo**: Estado local
- **Comportamiento**: Abre el modal de prompt viewer. Extrae `refined_prompt` y `user_prompt` del output del nodo `prompt_build`
- **Impacto en Django**: Ninguno — operación puramente visual

### 7. Copy Prompt
- **Tipo**: Clipboard
- **Comportamiento**: Copia el texto del prompt activo al portapapeles
- **Impacto en Django**: Ninguno

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/trace` |
| **Archivos** | `dashboard/app/trace/page.tsx` (735 líneas), `dashboard/components/trace/trace-node.tsx` (427 líneas) |
| **Líneas totales** | ~1,162 líneas (page + node component) |
| **Librería** | `@xyflow/react` (React Flow v12) |
| **APIs al cargar** | `GET /trace/list?limit=50`, `GET /trace/latest/graph` |
| **APIs de escritura** | `POST /trace/replay`, `DELETE /trace/{id}`, `DELETE /trace` |
| **Total endpoints** | 6 |
| **Node types registrados** | 32 (15 core + 17 identity) |
| **Edge type** | smoothstep para todos |
| **Layout** | Pre-computado server-side — posiciones vienen de la API |
| **Memoization** | `TraceNodeComponent` wrapped en `React.memo` |
| **Estado** | React `useState` para graphMeta, nodes, edges, traceList, selectedTraceId, replayInput |
| **Iconos page** | lucide-react: Activity, Play, RefreshCw, Clock, Loader2, Zap, Box, ArrowRight, Send, Layers, ChevronRight, Eye, X, Copy, Check, Trash2 |
| **Iconos node** | lucide-react: MessageSquare, GitBranch, Scale, ListChecks, Database, FileText, Brain, User, Shield, Zap, ArrowDown, AlertTriangle, Users, Fingerprint, Gauge, SlidersHorizontal, Palette, Link, ArrowUpDown, PenTool, Syringe, Layers, ShieldAlert, MessageCircle, HeartPulse, Activity, TrendingUp, FlaskConical, GitCommit, CheckCircle2, XCircle, Clock, SkipForward, ChevronDown, ChevronUp |

**Última actualización**: 2025-06-23
