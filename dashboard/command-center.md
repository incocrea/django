# Command Center — `/`

## Información General

El Command Center es la página principal del dashboard y sirve como centro de control operacional de Django. Proporciona una vista consolidada en tiempo real del estado completo del sistema: salud de los servicios, métricas clave de rendimiento (KPIs), estado de los agentes, actividad reciente, configuración de la persona, y estado del router de modelos. Es la primera pantalla que ve el usuario al acceder al dashboard y desde aquí puede navegar rápidamente a cualquier módulo del sistema mediante acciones directas.

---

## Información Presentada

### Barra de Salud del Sistema (7 LEDs)

Indicadores visuales en la parte superior que muestran el estado de cada componente crítico:

| LED | Fuente de Datos | Verde (online) | Ámbar (degraded) | Rojo (offline) |
|-----|-----------------|----------------|-------------------|----------------|
| **Database** | `GET /health` → `database` | `status: "connected"` | — | Cualquier otro valor |
| **LLM** | `GET /health` → `llm` | `status: "connected"` | — | Cualquier otro valor |
| **WebSocket** | `GET /health` → `websocket` | `status: "active"` | — | Cualquier otro valor |
| **Router** | `GET /router/status` → `providers` | Algún provider `available: true` | — | Ninguno disponible |
| **Governance** | `GET /system-status` → `governance` | `emergency_stopped: false` | — | `emergency_stopped: true` |
| **Storage** | `GET /system-status` → `memory_stats` | Cualquier dato presente | — | Error al obtener stats |
| **Security** | Siempre verde (placeholder) | Siempre | — | — |

### Tarjetas KPI (8 métricas)

| KPI | Icono | Fuente | Valor mostrado |
|-----|-------|--------|----------------|
| **Conversations** | MessageSquare | `system-status → conversations_count` | Número total de conversaciones |
| **Fidelity** | Target | `system-status → training.total_corrections` | `max(0, 100 - corrections × 2)%` (heurístico) |
| **Autonomy Level** | Zap | `system-status → governance.current_level` | `Level X` (0-4) |
| **Training Sessions** | BookOpen | `system-status → training.total_sessions` | Número total de sesiones |
| **Active Skills** | Wrench | `system-status → skills` | `X/Y active` (habilitadas/total) |
| **Est. Cost** | DollarSign | `system-status → token_usage` | `$X.XX` (cálculo: Gemini $0.15/1M + Groq $0.05/1M) |
| **Memory Items** | Database | `system-status → memory_stats` | Suma de todos los tiers |
| **Uptime** | Clock | `system-status → uptime_seconds` | Formateado como `Xh Ym` o `Xm Ys` |

### Anillo de Estado de Agentes

Visualización circular SVG que muestra cuántos agentes están en cada estado. Los estados se actualizan en tiempo real vía WebSocket (eventos `agent_state.*`):

| Estado | Color | Significado |
|--------|-------|-------------|
| `idle` | Verde | Agente disponible, sin actividad |
| `thinking` | Azul | Agente procesando/razonando |
| `acting` | Ámbar | Agente ejecutando una acción |
| `reviewing` | Púrpura | Agente revisando output |
| `waiting` | Gris | Agente esperando input |
| `error` | Rojo | Agente en estado de error |
| `learning` | Cian | Agente en proceso de aprendizaje |

### Tarjeta de Persona

Muestra la configuración activa de la persona cargada desde `persona.yaml`:

- **Nombre del principal** y versión de identidad
- **Big Five traits**: 5 barras horizontales (Openness, Conscientiousness, Extraversion, Agreeableness, Neuroticism) con valores 0-100%
- **Values**: Lista de los valores configurados del principal

Fuente: `GET /persona/info`

### Tarjeta del Router

Muestra el estado del Model Router y su cadena de fallback:

- **Cadena de fallback**: Gemini → Groq con indicadores de disponibilidad (2-level)
- **Estado de circuit breakers**: Contadores de fallos por provider
- **Perfil activo**: El profile seleccionado (balanced, max_quality)

Fuente: `GET /router/status`

### Feed de Actividad

Lista cronológica de los eventos más recientes del sistema (últimos 50). Dos formatos de presentación:

- **Decision Cards**: Para eventos de tipo `agent.decision.*`, muestra categoría, estrategia, agente seleccionado, y nivel de riesgo con badges de color
- **Standard Events**: Para todos los demás, muestra tipo de evento, timestamp, y datos resumidos

Fuente: `GET /events/recent?limit=50` + actualizaciones en tiempo real vía WebSocket

### Mecanismo de Actualización

- **Polling automático**: Cada 5 segundos se re-llaman los endpoints de health, system-status, events, persona y router
- **WebSocket**: Eventos de estado de agentes (`agent_state.*`) se procesan en tiempo real sin polling

---

## Acciones

### 1. Start Chat
- **Tipo**: Navegación
- **Comportamiento**: Redirige a `/chat`
- **Impacto en Django**: Ninguno directo — solo navegación

### 2. Start Training
- **Tipo**: API Call + Navegación
- **Comportamiento**: Llama `POST /training/session/start` con modo `correction`, luego redirige a `/training`
- **Impacto en Django**: Inicia una nueva sesión de entrenamiento en `TrainingManager`. El backend registra la sesión como activa, emite evento `training.session_started`. Si ya hay una sesión activa, el backend retorna error.

### 3. Reload Persona
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /persona/reload`, luego `GET /persona/info` para actualizar la UI
- **Impacto en Django**: El backend re-lee `configs/persona.yaml` desde disco, reconstruye los prompts de todos los agentes con los nuevos datos de personalidad. Los cambios aplican inmediatamente a todas las respuestas subsiguientes. Emite evento `config.persona_reloaded`.

### 4. Reload Router
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /router/reload`, luego `GET /router/status` para actualizar la UI
- **Impacto en Django**: El backend re-lee `configs/models.json` desde disco, reconfigura el `ModelRouter` con los nuevos providers, assignments, y fallback chain. Puede cambiar qué modelo usa cada agente. Emite evento `config.router_reloaded`.

### 5. Identity Governance
- **Tipo**: Navegación
- **Comportamiento**: Redirige a `/identity-governance`
- **Impacto en Django**: Ninguno directo — solo navegación

### 6. Pending Approvals (condicional)
- **Tipo**: Navegación
- **Comportamiento**: Solo visible cuando hay aprobaciones pendientes. Redirige a `/governance`
- **Impacto en Django**: Ninguno directo — solo navegación

### 7. Emergency Stop / Resume
- **Tipo**: API Call con doble confirmación
- **Emergency Stop**: Requiere hacer clic dos veces (botón principal + confirmar en `ConfirmDialog`). Llama `POST /governance/emergency-stop`
- **Emergency Resume**: Mismo patrón de doble confirmación. Llama `POST /governance/emergency-resume`
- **Impacto en Django**:
  - **Stop**: Establece `_emergency_stopped = True` en el orquestador. **BLOQUEA TODO el procesamiento** — cualquier mensaje enviado al chat retornará error en el paso 0 del pipeline. Emite evento `governance.emergency_stop_activated` al audit log. Estado solo en memoria (se pierde al reiniciar).
  - **Resume**: Restablece `_emergency_stopped = False`. El orquestador vuelve a procesar mensajes normalmente. Emite evento `governance.emergency_stop_deactivated`.

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/` |
| **Archivo principal** | `dashboard/app/page.tsx` (166 líneas) |
| **Componentes** | 8 en `dashboard/components/command-center/`: `kpi-cards.tsx`, `health-bar.tsx`, `agent-ring.tsx`, `activity-feed.tsx`, `persona-card.tsx`, `router-card.tsx`, `quick-actions.tsx`, `system-health.tsx` |
| **Líneas totales** | ~1,170 líneas (página + componentes) |
| **APIs al cargar** | `GET /health`, `GET /system-status`, `GET /events/recent?limit=50`, `GET /persona/info`, `GET /router/status` |
| **Actualización** | Polling cada 5 segundos + WebSocket para estados de agentes |
| **Estado local** | React `useState` para cada bloque de datos |
| **Iconos** | lucide-react: Activity, MessageSquare, Target, Zap, BookOpen, Wrench, DollarSign, Database, Clock, RefreshCw, Play, AlertCircle, Shield, CheckCircle, XCircle |

**Última actualización**: 2025-06-23
