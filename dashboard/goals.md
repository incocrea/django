# Goal Manager — `/goals`

## Información General

El Goal Manager implementa un sistema teleológico de gestión de objetivos para Django. Permite definir, priorizar, y gestionar objetivos a diferentes niveles (táctico, operacional, estratégico, misión), detectar conflictos entre objetivos activos, y monitorear señales de recompensa asociadas a las interacciones. Este módulo es parte del sistema de gobernanza — los objetivos informan el comportamiento de Django al proporcionar contexto sobre las prioridades del principal.

La página organiza su funcionalidad en 3 tabs: Goals (lista y gestión), Conflicts (detección automática de conflictos entre objetivos), y Rewards (señales de recompensa por interacción).

---

## Información Presentada

### Tab 1: Goals (Target icon)

#### Lista de Objetivos
Cada objetivo se muestra como un Card con:

| Elemento | Descripción |
|----------|-------------|
| **Title** | Nombre del objetivo en texto semibold |
| **Type badge** | Tipo del objetivo con colores: mission (púrpura), strategic (índigo), operational (cyan), tactical (teal) |
| **Status badge** | Estado actual con colores y borde: created (azul), active (verde), paused (amarillo), completed (esmeralda), failed (rojo), cancelled (gris), blocked (naranja) |
| **Description** | Texto descriptivo debajo del título |
| **Priority** | Porcentaje (0-100%) — prioridad relativa del objetivo |
| **Alignment** | Porcentaje (0-100%) — alineación con la identidad del principal (identity_alignment) |

#### Botones por Objetivo

| Botón | Visible cuando | Icono | Color |
|-------|---------------|-------|-------|
| **Activate** | `status === "created"` | Play | Verde |
| **Complete** | `status === "active"` | CheckCircle | Esmeralda |
| **Cancel** | `status ∈ {created, active, paused}` | XCircle | Rojo |
| **Delete** | Siempre (todos los estados) | Trash2 | Rojo |

Fuente: `GET /goals`

### Tab 2: Conflicts (AlertTriangle icon)

#### Lista de Conflictos
Cada conflicto detectado muestra:
- **Conflict Type**: Tipo del conflicto en MAYÚSCULAS (ej: RESOURCE, PRIORITY, TEMPORAL)
- **Severity**: Porcentaje de severidad (0-100%)
- **Description**: Descripción textual del conflicto

Los conflictos se detectan automáticamente al cargar la página comparando los objetivos activos entre sí.

Fuente: `GET /goals/conflicts/detect`

### Tab 3: Rewards (BarChart3 icon)

#### Lista de Señales de Recompensa
Cada señal muestra en una fila horizontal:
- **Interaction ID**: Primeros 8 caracteres del ID de interacción
- **Composite Reward**: Valor decimal monoespaciado (ej: 0.750)
- **Timestamp**: Fecha y hora truncada a segundos

Fuente: `GET /goals/rewards/recent?limit=20`

---

## Acciones

### 1. Create Goal
- **Tipo**: Formulario inline → API Call
- **Comportamiento**: Botón "Create Goal" (Plus icon) toggle formulario inline con:
  - **Title** (texto, requerido)
  - **Description** (textarea)
  - **Type** (select: Tactical, Operational, Strategic, Mission — default: tactical)
  
  Llama `POST /goals` con `{title, description, goal_type}`

- **Impacto en Django**: Crea un nuevo objetivo en el sistema teleológico del backend con:
  - `status: "created"` (no activo hasta activación explícita)
  - `priority` y `identity_alignment` calculados automáticamente por el backend
  - UUID `goal_id` generado
  - Disponible para el orquestador como contexto de prioridades del principal

### 2. Activate Goal
- **Tipo**: API Call directa
- **Comportamiento**: Botón Play (verde) en objetivos con status "created" → llama `POST /goals/{id}/activate`
- **Impacto en Django**: Cambia el estado a "active". Los objetivos activos son considerados por el sistema de gobernanza durante la evaluación de decisiones y pueden influir en el comportamiento del orquestador al proveer contexto sobre las prioridades actuales del principal.

### 3. Complete Goal
- **Tipo**: API Call directa
- **Comportamiento**: Botón CheckCircle (esmeralda) en objetivos con status "active" → llama `POST /goals/{id}/complete`
- **Impacto en Django**: Marca el objetivo como completado. Sale del conjunto de objetivos activos y deja de influir en el contexto de decisiones. Genera señal de recompensa positiva.

### 4. Cancel Goal
- **Tipo**: API Call directa
- **Comportamiento**: Botón XCircle (rojo) en objetivos con status created/active/paused → llama `POST /goals/{id}/cancel`
- **Impacto en Django**: Marca el objetivo como cancelado. Sale del conjunto de objetivos activos. No genera señal de recompensa.

### 5. Delete Goal
- **Tipo**: API Call directa (sin confirmación)
- **Comportamiento**: Botón Trash2 (rojo) — disponible en **todos los estados** → llama `DELETE /goals/{id}`
- **Impacto en Django**: ⚠️ **Eliminación permanente** del objetivo. Sin diálogo de confirmación. Si el objetivo estaba activo, se elimina inmediatamente del contexto de gobernanza. Los conflictos asociados se recalculan automáticamente.

### 6. Refresh (implícito)
- **Tipo**: Carga automática
- **Comportamiento**: Los 3 tabs se cargan en paralelo con `Promise.all([loadGoals(), loadConflicts(), loadRewards()])` al montar la página
- **Impacto en Django**: Solo lectura

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/goals` |
| **Archivo** | `dashboard/app/goals/page.tsx` (362 líneas) |
| **APIs de lectura** | `GET /goals`, `GET /goals/conflicts/detect`, `GET /goals/rewards/recent?limit=20` |
| **APIs de escritura** | `POST /goals`, `POST /goals/{id}/activate`, `POST /goals/{id}/complete`, `POST /goals/{id}/cancel`, `DELETE /goals/{id}` |
| **APIs definidas en api.ts pero NO usadas por la UI** | `GET /goals/{id}` (getGoal), `PUT /goals/{id}` (updateGoal), `POST /goals/{id}/governance` (evaluateGoalGovernance) |
| **Total endpoints usados** | 8 |
| **Estado** | React `useState` para goals, conflicts, rewards, activeTab, showCreate, formulario (newTitle, newDesc, newType) |
| **Carga** | Paralela al montar — `Promise.all()` con 3 llamadas API |
| **Tipos de objetivo** | 4: tactical, operational, strategic, mission |
| **Estados posibles** | 7: created, active, paused, completed, failed, cancelled, blocked |
| **Componentes** | Todo inline en page.tsx — no hay componentes extraídos |
| **Iconos** | lucide-react: Target, Plus, Play, CheckCircle, XCircle, Trash2, Shield, AlertTriangle, BarChart3 |
| **⚠️ Nota** | Delete no tiene diálogo de confirmación — eliminación directa |

**Última actualización**: 2025-06-23
