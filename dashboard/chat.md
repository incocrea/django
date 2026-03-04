# Chat — `/chat`

## Información General

La interfaz de Chat es el canal principal de comunicación directa con Doe. Permite mantener conversaciones en texto con el delegado digital, seleccionar el modo cognitivo que controla cuánta inteligencia y recursos usa Doe para responder, y observar metadatos de cada respuesta (modelo usado, agentes involucrados, fuentes de conocimiento). Es la interfaz que desencadena el pipeline completo del orquestador — cada mensaje enviado aquí activa los 10+ pasos de procesamiento, incluyendo clasificación, decisión, planificación, recall de memoria, generación LLM, revisiones de identidad/gobernanza, evaluación, y persistencia.

---

## Información Presentada

### Historial de Mensajes

Área de scroll vertical que muestra la conversación actual como burbujas de mensaje alternadas:

- **Mensajes del usuario**: Alineados a la derecha, fondo acento
- **Mensajes de Doe**: Alineados a la izquierda, fondo oscuro, con metadatos adicionales:
  - **Indicador multi-agente**: Badge que muestra si múltiples agentes colaboraron
  - **Modelo usado**: Nombre del provider y modelo (ej: "gemini-2.5-flash")
  - **Modo cognitivo activo**: Badge indicando el modo que generó la respuesta
  - **Fuentes de conocimiento**: Indicadores de qué tiers de memoria contribuyeron a la respuesta (working, episodic, semantic, procedural)

### Selector de Modo Cognitivo

Botón cíclico en la barra de input que rota entre 3 modos. El modo seleccionado se envía como parámetro `cognitive_mode` en cada petición `POST /chat`:

| Modo | Icono | Color | Comportamiento |
|------|-------|-------|----------------|
| **Full (1)** | 🟢 | Verde | Web search + LLM + todas las memorias. Sin restricciones. Usa habilidades como web-research si están habilitadas. |
| **Memory+LLM (2)** | 🟡 | Amarillo | **Modo por defecto**. LLM genera pero solo basado en memoria recalled. Filtro semántico a `learned_knowledge`. Inyecta Knowledge Boundary + Status Header en el system prompt. Sin web. |
| **Memory Only (3)** | 🔴 | Rojo | Solo recall de memoria, **sin llamada LLM**. El orquestador ensambla líneas `[tier/category] text` desde memorias recalled y retorna directamente. |

### Estado de Conexión

El header muestra el estado del WebSocket (conectado/desconectado) y el nombre del modelo activo.

### Persistencia

- Las conversaciones se persisten en `localStorage` vía Zustand persist middleware
- NO hay persistencia server-side del historial de chat mostrado — el backend guarda en Postgres si la DB está disponible, pero la UI solo lee desde localStorage
- Cada sesión del dashboard mantiene un `conversation_id` único

---

## Acciones

### 1. Enviar Mensaje
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /chat` con `{message, conversation_id, cognitive_mode}`
- **Impacto en Doe**: **Activa el pipeline completo del orquestador** (25+ pasos internos):
  1. Crea/reutiliza conversación en Postgres
  2. Guarda mensaje del usuario en DB
  3. Emite `agent_state.thinking` vía WebSocket
  4. Carga historial de conversación (últimos 20 mensajes desde DB)
  5. Ejecuta pipeline: Classify → Decision Engine → Identity phases (6C, 6D, 7A, 8A, 8B) → Planner → Memory Recall + Mode Filter → Identity phases (6A, 7B) → Corrections → Identity Context (6B) → Prompt Build → LLM Generate → Identity Review → Governance Review → Memory Consolidation (7C) → Memory Store + Evaluation (5 módulos) + Identity Enforcement (5A, 5B, 5C) + Persistence → Health Monitor (9A) → Health Regulation (9B) → Evolution (10A) → Shadow Sim (10B) → Version Candidate (10C)
  6. Guarda respuesta en DB
  7. Emite `chat.response_generated` y `agent_state.idle`
  8. Retorna `ChatResponse` con contenido, metadata, y trace
- **Efectos colaterales**: Cada mensaje genera registros en ~10 tablas Postgres, actualiza memoria working + episodic, puede disparar evaluaciones de drift de identidad, y crea un cognitive trace completo.

### 2. Cambiar Modo Cognitivo
- **Tipo**: Estado local
- **Comportamiento**: Rota cíclicamente entre modo 1 → 2 → 3 → 1
- **Impacto en Doe**: Afecta el PRÓXIMO mensaje. No llama ninguna API al cambiar — el modo se incluye como parámetro en la siguiente petición `POST /chat`, modificando fundamentalmente qué pasos del pipeline se ejecutan.

### 3. Limpiar Conversación
- **Tipo**: Estado local únicamente
- **Comportamiento**: Borra el historial de mensajes del localStorage y genera un nuevo `conversation_id`
- **Impacto en Doe**: **Ninguno** — no llama `POST /memory/working/clear` ni ningún endpoint backend. Solo limpia la vista del cliente. La working memory del conversation_id anterior sigue existiendo en el servidor hasta que expire por TTL (1 hora) o alcance el límite de sesiones (50).

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/chat` |
| **Archivos** | `dashboard/app/chat/page.tsx` (14 líneas, wrapper), `dashboard/components/chat/chat-panel.tsx` (199 líneas, lógica principal), `dashboard/components/chat/message-bubble.tsx` (100 líneas, renderizado de burbuja) |
| **Líneas totales** | ~313 líneas |
| **APIs al cargar** | Ninguna — restaura historial desde localStorage |
| **API por mensaje** | `POST /chat` con `{message, conversation_id, cognitive_mode}` |
| **Estado** | Zustand store con persistencia localStorage (`chat-storage` key) |
| **Respuesta esperada** | `ChatResponse { content, metadata: {model, provider, agents, trace_id}, cognitive_mode }` |
| **Iconos** | lucide-react: Send, Loader2, Brain, MessageSquare |

**Última actualización**: 2025-06-23
