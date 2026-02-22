# Evaluation Dashboard — `/evaluation`

## Información General

El Evaluation Dashboard presenta los resultados de los 5 módulos de evaluación heurística que analizan cada interacción de Django. Estos módulos corren automáticamente después de cada respuesta del orquestador (paso 10 del pipeline) sin hacer llamadas LLM adicionales. La página permite inspeccionar scores de calidad, alineación con la persona, riesgos legales detectados, decisiones de negocio registradas, y operaciones de memoria con rollback. Desde aquí también se pueden aprobar/rechazar decisiones pendientes, deshacer operaciones de memoria, crear checkpoints, y purgar todos los datos de evaluación.

---

## Información Presentada

### Tab 1: Overview

#### Fila de KPIs (5 tarjetas)

| KPI | Icono | Color | Fuente | Valor |
|-----|-------|-------|--------|-------|
| **Avg Quality** | Star | Verde | `overview.quality.avg_composite` | Score compuesto en %, sub: "N interactions scored" |
| **Alignment** | Target | Azul | `overview.alignment.avg_overall` | Score de alineación en %, sub: "N evaluated" |
| **Legal Flags** | Scale | Rojo/Verde | `overview.legal_risk.requires_review` | Conteo de items que requieren revisión, sub: "N auto-blocked" |
| **Decisions** | Briefcase | Amarillo | `overview.decisions.total_decisions` | Total de decisiones detectadas, sub: "N pending approval" |
| **Memory Ops** | RotateCcw | Púrpura | `overview.rollback.total_operations` | Total de operaciones de memoria, sub: "N undoable" |

#### Quality Dimensions
Barras de score por cada dimensión del módulo de calidad:
- **Relevance**: Qué tan relevante es la respuesta al prompt
- **Coherence**: Coherencia interna de la respuesta
- **Completeness**: Cobertura del tema
- **Efficiency**: Eficiencia en el uso de tokens
- **Memory Utilization**: Qué tan bien aprovecha las memorias recalled

Colores: >=80% verde, >=60% amarillo, >=40% naranja, <40% rojo.

#### Grade Distribution
Distribución de grades A/B/C/D/F con badges de color y conteo.

#### Alignment Breakdown
Barras por dimensión: Values, Communication, Boundaries, Decisions. Si Identity Similarity > 0, muestra barra adicional.

#### Risk Distribution
Badges por nivel de riesgo con conteo (low/medium/high/critical).

#### Decision Status
Badges por estado de decisión (pending/approved/executed/rejected/deferred) con conteo.

Fuente: `GET /evaluation/overview`

### Tab 2: Quality

Lista de reportes de calidad expandibles (últimos 30). Cada tarjeta colapsada muestra:
- `GradeBadge` (A-F con color) + ID de interacción (primeros 12 chars) + composite %
- Expandido: 5 barras `ScoreBar` por dimensión + lista de flags (bullets amarillos)

Fuente: `GET /evaluation/quality/reports?limit=30`

### Tab 3: Alignment

Reportes de alineación expandibles (últimos 30). Cada tarjeta colapsada muestra:
- Score overall con color + ID + badge de conteo de violaciones
- Expandido:
  - 4 barras: Values, Communication, Boundaries, Decisions
  - **Identity Similarity** (con icono Fingerprint): `ScoreBar` + severity del drift (computado localmente: gap desde threshold 0.78 → none/low/medium/high) + policy action (none/log/flag/rewrite_request). Mostrado con `SeverityBadge` y `PolicyActionBadge`
  - Listas de violaciones (rojas) y observaciones (azules)

Fuente: `GET /evaluation/alignment/reports?limit=30`

### Tab 4: Legal Risk

Reportes de riesgo legal expandibles (últimos 30). Cada tarjeta muestra:
- `RiskBadge` + ID + badge "Review" (naranja, Eye icon) si `requires_review` + badge "Blocked" (rojo, XCircle) si `auto_blocked`
- Expandido: Cada flag con severity badge, categoría, descripción, y recomendación

Las 6 categorías de riesgo legal detectadas:
1. **Contractual**: Lenguaje contractual vinculante
2. **Financial**: Implicaciones económicas
3. **Confidentiality**: Posible divulgación de información sensible
4. **Liability**: Responsabilidad legal
5. **Regulatory**: Compliance regulatorio
6. **AI Disclosure**: Falta de divulgación de naturaleza AI

Fuente: `GET /evaluation/legal/reports?limit=30`

### Tab 5: Decisions

Decisiones de negocio detectadas (últimas 50). Cada tarjeta muestra:
- `StatusBadge` + `RiskBadge` + categoría + texto de decisión + Clock icon si pending
- Expandido: Grid con trigger, urgency, reversible (Yes/No), financial impact, rationale, alternatives

Fuente: `GET /evaluation/decisions?limit=50`

### Tab 6: Rollback

Visualización de operaciones de memoria y checkpoints.

#### Sección Checkpoints
Lista de checkpoints nombrados con icono Save, nombre y timestamp.

#### Sección Operations
Lista de operaciones de memoria con:
- Badge de acción: store (verde), delete (rojo), modify (amarillo), otro (azul)
- Tier label, target ID (primeros 16 chars), timestamp
- Badge "Undone" para operaciones ya revertidas
- Botón Undo para operaciones no revertidas

Fuentes: `GET /evaluation/rollback/log?limit=50` + `GET /evaluation/rollback/checkpoints`

---

## Acciones

### 1. Approve Decision
- **Tipo**: API Call
- **Comportamiento**: Botón verde en decisiones con status "pending" y `requires_approval: true`. Llama `PUT /evaluation/decisions/{id}/status` con `{status: "approved", approved_by: "principal"}`
- **Impacto en Django**: Actualiza el estado de la decisión en el `DecisionRegistry` a "approved". Esto puede afectar el comportamiento futuro del orquestador si implementa lógica condicional basada en decisiones aprobadas (actualmente informativo).

### 2. Reject Decision
- **Tipo**: API Call
- **Comportamiento**: Botón rojo. Llama `PUT /evaluation/decisions/{id}/status` con `{status: "rejected", approved_by: "principal"}`
- **Impacto en Django**: Marca la decisión como rechazada en el `DecisionRegistry`.

### 3. Defer Decision
- **Tipo**: API Call
- **Comportamiento**: Botón gris. Llama `PUT /evaluation/decisions/{id}/status` con `{status: "deferred", approved_by: "principal"}`
- **Impacto en Django**: Pospone la decisión. Sigue apareciendo en la lista pero con badge "deferred".

### 4. Undo Memory Operation
- **Tipo**: API Call
- **Comportamiento**: Botón naranja Undo2 en operaciones no revertidas. Llama `POST /evaluation/rollback/undo/{operationId}`
- **Impacto en Django**: **Restaura la memoria a su estado anterior** desde el snapshot before-state:
  - Si fue un `delete`: Re-crea la memoria en ChromaDB/SQLite con el contenido original
  - Si fue un `modify`: Revierte al contenido anterior
  - Si fue un `store`: Elimina la memoria creada
  
  Esto afecta directamente el recall del paso 4 del pipeline — las futuras respuestas de Django tendrán acceso a memorias restauradas o perderán acceso a memorias revertidas.

### 5. Create Checkpoint
- **Tipo**: API Call
- **Comportamiento**: Botón Save. Llama `POST /evaluation/rollback/checkpoint` con nombre auto-generado `checkpoint-{timestamp}`
- **Impacto en Django**: Crea un checkpoint nombrado en el `MemoryRollbackManager`. Marca el punto actual en el log de operaciones. (Nota: actualmente el rollback to checkpoint no está expuesto en la UI).

### 6. Delete All Evaluation Data
- **Tipo**: API Call con confirmación
- **Comportamiento**: Botón rojo Trash2 en header → confirmar en `ConfirmDialog`. Llama `DELETE /evaluation/data`
- **Impacto en Django**: **Purga permanente** de todos los datos de evaluación:
  - Limpia las 5 stores en memoria (quality, alignment, legal, decisions, rollback)
  - Elimina los registros de evaluación de Postgres
  - Resetea todos los contadores y métricas a cero
  - **No afecta** memorias, traces, configuración, ni el comportamiento de Django — solo datos de evaluación históricos

### 7. Refresh
- **Tipo**: API Call
- **Comportamiento**: Re-llama overview + datos del tab activo
- **Impacto en Django**: Solo lectura

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/evaluation` |
| **Archivo** | `dashboard/app/evaluation/page.tsx` (874 líneas) |
| **APIs al cargar** | `GET /evaluation/overview` (siempre) + datos de tab activo al cambiar |
| **APIs por tab** | Quality: `GET /evaluation/quality/reports?limit=30`, Alignment: `GET /evaluation/alignment/reports?limit=30`, Legal: `GET /evaluation/legal/reports?limit=30`, Decisions: `GET /evaluation/decisions?limit=50`, Rollback: `GET /evaluation/rollback/log?limit=50` + `GET /evaluation/rollback/checkpoints` |
| **APIs de escritura** | `PUT /evaluation/decisions/{id}/status`, `POST /evaluation/rollback/undo/{id}`, `POST /evaluation/rollback/checkpoint`, `DELETE /evaluation/data` |
| **Total endpoints** | ~21 |
| **Carga lazy** | Tab-specific data se carga on-demand al cambiar de tab |
| **Componentes helper** | 8: `GradeBadge`, `RiskBadge`, `StatusBadge`, `DriftBadge`, `SeverityBadge`, `PolicyActionBadge`, `ScoreBar`, `KpiCard` |
| **Estado** | React `useState` para overview, datos de cada tab, tab activo, expansión de tarjetas |
| **Backend modules** | `src/evaluation/quality_scorer.py` (456 ln), `alignment_evaluator.py` (506 ln), `legal_risk.py` (359 ln), `decision_registry.py` (407 ln), `memory_rollback.py` (408 ln) |
| **Iconos** | lucide-react: Star, Target, Scale, Briefcase, RotateCcw, Trash2, RefreshCw, Eye, XCircle, Clock, Fingerprint, Save, Undo2, ChevronDown, ChevronUp |

**Última actualización**: 2025-06-23
