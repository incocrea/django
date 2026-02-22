# Governance Console — `/governance`

## Información General

La Governance Console es el centro de control de las reglas, límites, y políticas que gobiernan el comportamiento de Django. Presenta la configuración completa de gobernanza (niveles de autonomía, clasificación de riesgo, operaciones prohibidas, controles de emergencia, privacidad), el registro de auditoría de eventos del sistema, la cola de aprobaciones pendientes, una matriz estática de límites por nivel de autonomía, y el middleware del pipeline. Desde aquí se puede activar/desactivar el Emergency Stop que bloquea completamente el procesamiento de Django.

---

## Información Presentada

### Tab 1: Overview (Configuración de Gobernanza)

Renderiza el contenido completo de `configs/governance.yaml` en secciones colapsables:

#### Niveles de Autonomía
Grid de 5 tarjetas (niveles 0-4), cada una con:
- **Número de nivel** y **nombre** (Observer, Assistant, Collaborator, Delegate, Trusted)
- **Descripción** del nivel
- **Capabilities** (hasta 3 por nivel): lo que Django puede hacer autónomamente en ese nivel

#### Clasificación de Riesgo
Tarjetas por nivel de riesgo con badges de color:
- **Low** (verde): Ejemplos y si requiere aprobación
- **Medium** (amarillo): Ejemplos y requerimiento de aprobación
- **High** (naranja): Ejemplos y requerimiento de aprobación
- **Critical** (rojo): Ejemplos y requerimiento de aprobación

#### Operaciones Prohibidas
Lista con iconos XCircle rojos de operaciones que Django **nunca** puede ejecutar, independientemente del nivel de autonomía. Marcadas como "HARDCODED" — no configurables por el usuario.

#### Controles de Emergencia
Tarjeta con borde rojo que muestra:
- Estado del kill switch
- Configuración de auto-rollback
- Grace period configurado

#### Privacidad de Datos
Pares clave-valor de las políticas de privacidad de `governance.yaml`. Valores booleanos renderizados como Yes/No.

Fuente: `GET /governance/config`

### Tab 2: Audit Log

Tabla scrollable (max 600px) con header sticky. Últimos 100 eventos del sistema:

| Columna | Contenido |
|---------|-----------|
| **Time** | Timestamp formateado |
| **Event** | Tipo de evento, con badge rojo "Security" para eventos `security.*` |
| **Agent** | Rol del agente que generó el evento |
| **Action** | Descripción de la acción (max 300px, truncada) |
| **Risk** | Badge de riesgo (low/medium/high/critical) |

Fuente: `GET /governance/audit-log?limit=100`

### Tab 3: Approvals

Lista de items pendientes de aprobación con contador en el tab label. Cada item muestra:
- Badge de riesgo + descripción de la acción + timestamp
- Rol del agente + tipo de evento
- 3 botones de acción: Approve, Reject, Request Changes

**⚠️ Gap conocido**: Los botones de aprobación solo **descartan el item de la vista local** (agregan el índice a un Set `dismissed`). **No hacen ninguna llamada API** — no hay endpoint backend para procesar aprobaciones. Es una funcionalidad visual placeholder.

Fuente: `GET /governance/approvals`

### Tab 4: Boundaries (Matriz de Escalación)

Tabla estática (hardcodeada en el componente) que muestra qué nivel de autorización necesita cada tipo de acción según el nivel de autonomía:

| Acción | Level 0-1 | Level 2-3 | Level 4 |
|--------|-----------|-----------|---------|
| Email Drafting | Approval | Notify | Auto |
| Email Sending | Forbidden | Approval | Approval |
| Financial (Low) | Approval | Auto | Auto |
| Financial (High) | Forbidden | Approval | Approval |
| Legal Agreements | Forbidden | Forbidden | Approval |
| Public Statements | Forbidden | Approval | Notify |
| New Relationships | Approval | Notify | Auto |

Celdas con color: rojo (forbidden), amarillo (approval), azul (notify), verde (auto).

### Tab 5: Middleware

Lista de middlewares registrados en el pipeline del orquestador. Cada entrada muestra:
- Nombre, posición, prioridad, descripción
- Botón toggle (verde "Enabled" / rojo "Disabled")

Fuente: `GET /middleware`

---

## Acciones

### 1. Emergency Stop
- **Tipo**: API Call con confirmación
- **Comportamiento**: Botón rojo con sombra brillante en el header. Requiere confirmar en `ConfirmDialog` (variant "danger")
- **Impacto en Django**: Llama `POST /governance/emergency-stop`. Establece `_emergency_stopped = True` en el orquestador. **BLOQUEA COMPLETAMENTE TODO el procesamiento** de Django — cualquier mensaje (desde chat, Discord, API) será rechazado en el paso 0 del pipeline. Emite evento `governance.emergency_stop_activated` al audit log. Estado solo en memoria (se resetea al reiniciar el servidor). Indicador visual "STOPPED" pulsante aparece en el header.

### 2. Emergency Resume
- **Tipo**: API Call con confirmación
- **Comportamiento**: Botón verde con CheckCircle. Solo visible cuando Emergency Stop está activo. Requiere confirmar en `ConfirmDialog` (variant "info")
- **Impacto en Django**: Llama `POST /governance/emergency-resume`. Restablece `_emergency_stopped = False`. Django vuelve a procesar mensajes normalmente. Emite evento `governance.emergency_stop_deactivated`.

### 3. Toggle Middleware
- **Tipo**: API Call
- **Comportamiento**: Botón toggle en cada entrada de middleware
- **Impacto en Django**: Llama `POST /middleware/{name}/toggle`. Habilita o deshabilita un paso de middleware en el pipeline del orquestador. Los middlewares disabled se saltan durante el procesamiento.

### 4. Approval Actions (Approve/Reject/Request Changes)
- **Tipo**: Estado local únicamente
- **Comportamiento**: Descarta el item de la lista visual y muestra un toast de 3 segundos
- **Impacto en Django**: **NINGUNO** — no existe endpoint backend para procesar aprobaciones. Es un placeholder visual. Los items vuelven a aparecer al recargar la página.

### 5. Refresh
- **Tipo**: API Call
- **Comportamiento**: Re-llama los 5 endpoints
- **Impacto en Django**: Solo lectura

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/governance` |
| **Archivo** | `dashboard/app/governance/page.tsx` (572 líneas) |
| **APIs al cargar** | `GET /governance/emergency-status`, `GET /governance/config`, `GET /governance/audit-log?limit=100`, `GET /governance/approvals`, `GET /middleware` |
| **APIs de escritura** | `POST /governance/emergency-stop`, `POST /governance/emergency-resume`, `POST /middleware/{name}/toggle` |
| **Total endpoints** | 8 (5 lectura + 3 escritura) |
| **Config file** | `configs/governance.yaml` (leído por GET /governance/config) |
| **Estado** | React `useState` para cada bloque de datos, Set para dismissed approvals |
| **Componentes helper** | `RiskBadge` (color por nivel), `ConfirmDialog` (modal de confirmación) |
| **Iconos** | lucide-react: Shield, RefreshCw, XCircle, CheckCircle, AlertTriangle, Eye, Clock |

**Última actualización**: 2025-06-23
