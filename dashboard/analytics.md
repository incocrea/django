# Analytics Dashboard — `/analytics`

## Información General

El Analytics Dashboard muestra métricas operacionales agregadas sobre el funcionamiento de Django: distribución de eventos por tipo, agente y riesgo, tendencias de fidelidad de identidad, tasa de autonomía, y uso de tokens. Es una vista de solo lectura (excepto por la acción de borrar datos) que permite al principal monitorear patrones de uso, costos, y la consistencia del comportamiento de Django a lo largo del tiempo.

---

## Información Presentada

### Fila de KPIs (4 tarjetas)

| KPI | Icono | Color | Fuente | Valor |
|-----|-------|-------|--------|-------|
| **Total Events** | Activity | Cian | `GET /analytics/overview` → `total_events` | Número total de eventos registrados |
| **Identity Fidelity** | Target | Verde | `GET /analytics/identity-fidelity` → `current_score` | Porcentaje de fidelidad actual + "Baseline: X%" debajo. **Nota**: Este es un score heurístico basado en conteo de correcciones (`100 - corrections × 2`), NO similarity de embeddings real |
| **Autonomy Rate** | Zap | Amarillo | `GET /analytics/autonomy` → `autonomy_rate` | Porcentaje de tareas auto-completadas vs escaladas. Sub-texto: "X auto / Y escalated" |
| **Est. Tokens Used** | Brain | Púrpura | `GET /analytics/token-usage` → `estimated_tokens` | Tokens totales formateados como `X.XK`. Sub-texto: "N conversations" |

### Gráfico de Fidelidad Over Time

Bar chart de la serie temporal de fidelidad (`fidelity.series`). Cada barra representa un día:
- **Verde**: Score >= 90%
- **Cian**: Score >= 85%
- **Amarillo**: Score < 85%

Línea base (baseline) dibujada como referencia horizontal con label debajo. Rango normalizado 70-100%, altura 140px.

### Events by Type (Top 8)

Gráfico de barras verticales (`MiniBar` component) mostrando los 8 tipos de evento más frecuentes. Cada barra es proporcional al conteo máximo.

Fuente: `GET /analytics/overview` → `events_by_type`

### Events by Agent (Top 8)

Gráfico de barras verticales en azul mostrando actividad por agente.

Fuente: `GET /analytics/overview` → `events_by_agent`

### Events by Risk

Barras horizontales de progreso por nivel de riesgo con colores:
- **Low**: Verde
- **Info**: Azul
- **Medium**: Amarillo
- **High**: Naranja
- **Critical**: Rojo

Cada barra muestra el porcentaje respecto al total y el conteo absoluto a la derecha.

Fuente: `GET /analytics/overview` → `events_by_risk`

---

## Acciones

### 1. Clear All Data (Borrar Todos los Datos)
- **Tipo**: API Call con confirmación
- **Comportamiento**: Botón rojo con icono Trash2. Requiere confirmar en `ConfirmDialog` (variant "danger")
- **Impacto en Django**: Ejecuta en paralelo:
  - `DELETE /analytics/events` — Elimina todos los registros de eventos de Postgres y el EventBus en memoria
  - `DELETE /analytics/tokens` — Elimina todos los registros de uso de tokens de Postgres
  
  Después de la eliminación, todos los contadores y gráficos se reinician a cero. **Los datos eliminados NO son recuperables**. No afecta memorias, traces, evaluaciones, ni la configuración de Django — solo datos estadísticos de analytics.

### 2. Refresh
- **Tipo**: API Call
- **Comportamiento**: Re-llama los 4 endpoints de lectura
- **Impacto en Django**: Solo lectura

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/analytics` |
| **Archivo** | `dashboard/app/analytics/page.tsx` (330 líneas) |
| **APIs al cargar** | `GET /analytics/overview`, `GET /analytics/identity-fidelity`, `GET /analytics/autonomy`, `GET /analytics/token-usage` |
| **APIs de escritura** | `DELETE /analytics/events`, `DELETE /analytics/tokens` |
| **Total endpoints** | 6 (4 lectura + 2 borrado) |
| **Estado** | React `useState` para overview, fidelity, autonomy, tokenUsage |
| **Componentes helper** | `SparkLine` (SVG polyline, 120x32px — definido pero no usado), `MiniBar` (gráfico de barras vertical, max 8 entries) |
| **Iconos** | lucide-react: BarChart3, Activity, Target, Zap, Brain, Trash2, RefreshCw, TrendingUp, Users, Shield |

**Última actualización**: 2025-06-23
