# Evaluación — src/evaluation/

> [← Teleología](teleology.md) · [Eventos & Trace →](events.md)

**Directorio**: `src/evaluation/` (2,151 líneas, 6 archivos)

---

## Sobre este documento

Este documento describe los **5 módulos de evaluación heurística** que califican cada respuesta de Django después de que es generada. No usan LLM — todo es regex, keyword matching y scoring determinístico. Esto significa que evaluar una respuesta cuesta exactamente $0 y toma milisegundos, lo cual permite evaluar **cada una de las interacciones** sin excepción.

### ¿Qué cubre este documento?

Documenta los 5 módulos: **Quality** (5 dimensiones de calidad con grade A-F), **Alignment** (alineación con la persona de Harold en 5 factores), **Legal Risk** (detección de 6 categorías de riesgo legal con 15+ patrones regex), **Decisions** (auto-detección de decisiones de negocio), y **Rollback** (tracking de operaciones de memoria con before/after para poder deshacer). Cada módulo tiene su tabla de dimensiones, métricas de almacenamiento, y formato de output.

### ¿Cuál es su función en la arquitectura?

La evaluación es el **sistema de control de calidad** de Django. Mientras que la [Identidad](identity.md) intenta *prevenir* desviaciones (antes y durante la generación), la evaluación *detecta* problemas después de que la respuesta ya existe. Juntos, forman un ciclo de feedback: la identidad influye en cómo se genera la respuesta, y la evaluación mide qué tan bien salió.

### ¿Cómo afecta al comportamiento de Django?

- **Quality Scorer**: no cambia la respuesta actual, pero sus grades alimentan los dashboards y permiten identificar patrones de degradación a largo plazo
- **Alignment Evaluator**: calcula `identity_similarity` comparando la respuesta contra el baseline embedding de Harold — esta métrica es usada por múltiples módulos de identidad como señal
- **Legal Risk**: detecta si Django hizo promesas contractuales, compartió información confidencial, o hizo claims financieros — crucial para proteger a Harold
- **Decision Registry**: auto-detecta cuand Django toma decisiones de negocio, registrando trigger, alternativas y trade-offs para auditoría
- **Memory Rollback**: permite deshacer operaciones de memoria si algo se almacenó incorrectamente — checkpoints con rollback seguro

### ¿Cómo interactúa con las demás piezas?

- **[Pipeline](pipeline.md) paso 10**: los 5 módulos se ejecutan en **paralelo** después de la generación LLM, antes de persistir
- **[Identidad](identity.md)**: el `AlignmentEvaluator` recibe el `baseline_embedding` del `IdentityProfile` durante el [Startup](startup.md) para calcular `identity_similarity`. Este score es consumido por Phase 5A (Enforcement), Phase 6D (Confidence), Phase 9A (Health Monitor)
- **[Base de Datos](database.md)**: todos los resultados se persisten via `PersistenceRepository.save_all_evaluations()` en la tabla `evaluations` de Postgres
- **[Eventos](events.md)**: cada evaluación genera un nodo de traza cognitiva de tipo `evaluation` en el TraceCollector
- **[Memory](memory.md)**: el `MemoryRollbackManager` intercepta operaciones de memoria (store, edit, delete) y guarda estado before/after. Sus callbacks se conectan durante el [Startup](startup.md)
- **[Dashboard](../dashboard/README.md)**: el Evaluation Dashboard visualiza todos los scores, trends y flags en 5 tabs separadas
- **[API](../api/README.md)**: 21 endpoints exponen los datos de evaluación (stats, reports, flagged, decisions, rollback)

---

## Resumen

5 módulos heurísticos que evalúan CADA interacción. **Cero llamadas LLM extra** — todo es regex, keyword matching, y scoring determinístico. Singletons en memoria (max 200-1000 records, perdidos en restart, respaldados en [Postgres](database.md)).

---

## Tabla de Módulos

| Módulo | Archivo | Líneas | Qué Mide |
|--------|---------|--------|----------|
| **Quality** | `quality_scorer.py` | 383 | 5 dimensiones → grade A-F compuesto |
| **Alignment** | `alignment_evaluator.py` | 383 | Alineación con persona del principal |
| **Legal Risk** | `legal_risk.py` | 319 | Riesgo legal en 6 categorías |
| **Decisions** | `decision_registry.py` | 358 | Detecta y registra decisiones de negocio |
| **Rollback** | `memory_rollback.py` | 353 | Trackea operaciones de memoria |

---

## Quality Scorer

5 dimensiones evaluadas por respuesta:

| Dimensión | Qué mide |
|-----------|----------|
| Relevance | ¿La respuesta es relevante a la pregunta? |
| Coherence | ¿Es coherente internamente? |
| Completeness | ¿Cubre todos los aspectos? |
| Efficiency | ¿Es concisa sin sacrificar calidad? |
| Memory Utilization | ¿Usa bien el contexto de memoria? |

Grade compuesto: **A** (excelente) → **F** (fallo).

---

## Alignment Evaluator

Evalúa qué tan bien se alinea la respuesta con la persona del principal:

| Factor | Descripción |
|--------|-------------|
| Value Keywords | ¿Menciona valores del principal? |
| Communication Style | ¿Coincide con estilo de comunicación? |
| Boundary Compliance | ¿Respeta los límites establecidos? |
| Decision-Making | ¿Sigue patrones de decisión del principal? |
| Identity Similarity | Cosine similarity vs baseline embedding (via [identity](identity.md)) |

> **Nota**: `identity_similarity` se trackea pero NO se pondera en `overall_score` todavía — pendiente implementación.

---

## Legal Risk Assessment

15+ regex patterns en 6 categorías:

| Categoría | Ejemplos |
|-----------|----------|
| Contractual | Compromisos vinculantes, términos de contrato |
| Financial | Montos, precios, garantías financieras |
| Confidentiality | NDAs, información clasificada |
| Liability | Responsabilidad legal, disclaimers |
| Regulatory | Compliance normativo, regulaciones |
| AI Disclosure | Naturaleza AI, limitaciones |

---

## Decision Registry

Detecta decisiones de negocio automáticamente:

| Campo | Descripción |
|-------|-------------|
| Trigger | Qué disparó la decisión |
| Rationale | Razonamiento |
| Alternatives | Alternativas consideradas |
| Trade-offs | Compromisos |
| Stakeholders | Afectados |
| Status | pending / approved / rejected / deferred |

---

## Memory Rollback Manager

Trackea CADA operación de memoria con estado before/after:

| Feature | Descripción |
|---------|-------------|
| Operation log | Tipo (add/update/delete), tier, contenido before/after |
| Named checkpoints | Snapshots con nombre para rollback |
| Undo | Restaurar memoria desde before-state |
| Persistence callback | Wired a [Postgres](database.md) `memory_operations` at startup |

---

## Integración en el Pipeline

Las 5 evaluaciones corren en el **paso 10** del [pipeline](pipeline.md), después de Memory Store:

```
Paso 10: Memory Store
  ├── Quality Scorer → QualityReport
  ├── Alignment Evaluator → AlignmentReport (+ identity_similarity)
  ├── Legal Risk → LegalAssessment
  ├── Decision Registry → Decision (si detecta)
  └── Memory Rollback → OperationRecord
  └── Persist all to Postgres (evaluations table)
```

---

## Dashboard

Página: [Evaluation Dashboard](../dashboard/evaluation-dashboard.md) (`/evaluation`)

6 tabs: Overview / Quality / Alignment / Legal Risk / Decisions / Rollback.

**API**: 21 endpoints en `/evaluation/*` → ver [API](../api/README.md)

---

## Temas Relacionados

- [Pipeline](pipeline.md) — Paso 10 donde ejecutan
- [Identidad](identity.md) — identity_similarity en Alignment
- [Memoria](memory.md) — Rollback tracking de operaciones
- [Base de Datos](database.md) — Table `evaluations`
- [Evaluation Dashboard](../dashboard/evaluation-dashboard.md) — Interfaz visual
- [Teleología](teleology.md) — Quality score alimenta rewards
