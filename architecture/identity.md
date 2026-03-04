# Sistema de Identidad — src/identity/

> [← Memoria](memory.md) · [Teleología →](teleology.md)

**Directorio**: `src/identity/` (6,080 líneas, 22 archivos)

---

## Sobre este documento

Este documento describe el **sistema más complejo de toda la arquitectura**: los 22 módulos y 17 fases que aseguran que Doe actúe, piense y se comunique **como Harold Vélez**. Si el pipeline es el corazón de Doe, la identidad es su alma — la capa que lo diferencia de cualquier otro chatbot y lo convierte en un representante fiel de una persona real.

### ¿Qué cubre este documento?

Documenta los **22 archivos** del módulo de identidad, organizados en 17 fases que se fueron construyendo progresivamente. Cubre desde la representación formal (`IdentityProfile`, un modelo Pydantic con Big Five, valores, estilo de comunicación y embedding baseline de 4096 dimensiones) hasta los módulos avanzados de evolución (que proponen cambios en la identidad basados en tendencias largas), simulación shadow (que evalúan qué pasaría si se aplicara un cambio), y control de versiones (que permite rollback seguro si algo sale mal).

### ¿Cuál es su función en la arquitectura?

El sistema de identidad es el **guardián de la fidelidad**. Su trabajo es asegurar que Doe NO derive — que no empiece a hablar de formas que Harold no reconocería como propias. Lo hace de múltiples maneras: evaluando cada decisión contra los valores de Harold, re-rankeando memorias según su afinidad con la identidad, inyectando preferencias de estilo en el prompt, monitoreando salud a largo plazo, y proponiendo evoluciones controladas cuando la evidencia lo justifica.

### ¿Cómo afecta al comportamiento de Doe?

**Profundamente y en cada interacción**:
- Las memorias que Doe recuerda están re-rankeadas por afinidad con Harold (Phase 7B)
- El contexto lleva anotaciones visibles de alineación (`[IDENTITY_ALIGNED]`) que influyen en el LLM (Phase 6B)
- El prompt del sistema incluye preferencias de estilo derivadas de la identidad (tono, asertividad, profundidad, creatividad) (Phase 8B)
- Si la respuesta se desvía mucho de la identidad, se detecta el drift y se registra (Phase 5A-5C)
- La confianza del sistema de identidad puede ajustar el umbral de gobernanza (Phase 7A): baja confianza = más estricto
- A largo plazo, si Doe consistentemente evoluciona, el sistema puede proponer una nueva versión de identidad (Phase 10A-10C) — siempre con aprobación humana

### ¿Cómo interactúa con las demás piezas?

El sistema de identidad tiene **tentacles en todo el pipeline** — más que cualquier otro módulo:
- **[Cognición](cognition.md)**: el `DecisionEngine` recibe el `IdentityProfile` en su constructor (read-only) durante el [Startup](startup.md)
- **[Pipeline](pipeline.md)**: 10+ pasos del Orchestrator son pasos de identidad (3c, 3d, 3e, 3f, 3g, 2b, 2c, 3b, 6d, 7c, 7d, 7e, 9a, 9b, 9c, 9d, 9e)
- **[Memoria](memory.md)**: 4 interacciones directas — Memory Bridge (afinidad), Retrieval Weighting (re-ranking), Context Weighting (anotación), Consolidation (importancia pre-storage)
- **[Evaluación](evaluation.md)**: el `AlignmentEvaluator` recibe el baseline embedding y calcula `identity_similarity` para cada respuesta
- **[Eventos](events.md)**: cada módulo de identidad emite su propio evento (ej: `identity.drift_checked`, `identity.confidence_computed`, `identity.evolution_analyzed`)
- **[Base de Datos](database.md)**: cada resultado se persiste como evaluación en Postgres, y los snapshots de versión se almacenan como `config_versions`
- **[Config](../config/README.md)**: `persona.yaml` es la fuente primaria del perfil, y `governance.yaml` define el mapeo severity→action para la política de identidad
- **[Dashboard](../dashboard/README.md)**: Identity Studio para editar la persona, Identity Governance para gestionar versiones/evolución/shadow/health

Los 4 principios de diseño fundamentales del sistema son: **stateless** (sin estado entre llamadas), **observacional por defecto** (la mayoría observa sin modificar), **progresivo** (cada fase se construye sobre la anterior), y **soberanía humana** (nada se auto-aplica, Harold siempre tiene la última palabra).

---

## Resumen

Representación formal, versionada y embedding-backed de la identidad del principal. El sistema más complejo del proyecto: 22 módulos, 17 fases, integrado en 17 pasos del [pipeline](pipeline.md).

### Principios de Diseño

1. **Stateless modules** — Cada módulo es sin estado, sin constructor params, sin dependencia al orchestrator
2. **Observational by default** — La mayoría solo observa y reporta; no modifica datos
3. **Progressive layering** — Cada fase se construyó sobre la anterior
4. **Human sovereignty** — Las mutaciones de identidad siempre requieren aprobación humana

---

## Tabla de Módulos

| Archivo | Líneas | Fase | Clase | Función |
|---------|--------|------|-------|---------|
| `schema.py` | ~105 | 4 | `IdentityProfile` | Pydantic model — version, Big Five, values, comm_style, baseline_embedding 4096-dim (Qwen3-Embedding-8B via EmbeddingRouter), drift_threshold, content_hash (SHA-256) |
| `embedding.py` | ~233 | 4 | — | `build_identity_text()` + `compute_baseline_embedding()` (Qwen3-Embedding-8B via EmbeddingRouter, fallback all-MiniLM-L6-v2) + `cosine_similarity()` |
| `versioning.py` | ~215 | 4 | `IdentityVersionManager` | Semantic versioning, SHA-256 hashing, persistence via config_versions |
| `manager.py` | ~416 | 4 | `IdentityManager` | Singleton — load/save/rebuild profile. DB first, fallback YAML. Short-circuit si hash unchanged |
| `enforcement.py` | ~90 | 5A | `IdentityEnforcer` | Evalúa similarity vs drift_threshold → `{status, similarity, threshold, severity}` |
| `policy.py` | ~165 | 5B | `IdentityPolicyEngine` | Clasifica severidad de drift → acción (none/log/flag/rewrite_request/block) via [governance.yaml](../config/README.md) |
| `feedback.py` | ~110 | 5C | `IdentityFeedbackController` | Genera hints de corrección cuando action==rewrite_request AND severity==high |
| `memory_bridge.py` | ~165 | 6A | `IdentityMemoryBridge` | Cosine similarity per recalled memory vs baseline embedding |
| `context_weighting.py` | ~200 | 6B | `IdentityContextWeighter` | Tags `[IDENTITY_ALIGNED]` / `[LOW_IDENTITY_ALIGNMENT]` en líneas de contexto |
| `decision_modulation.py` | ~333 | 6C | `IdentityDecisionModulator` | Evalúa decision–identity alignment (4 factores) |
| `confidence.py` | ~264 | 6D | `IdentityConfidenceEngine` | Agrega señales → confidence score (0-1) + autonomy_modifier (+1/0/-1) |
| `autonomy_modulation.py` | ~175 | 7A | `IdentityAutonomyModulator` | Ajusta governance threshold según confidence level |
| `retrieval_weighting.py` | ~210 | 7B | `IdentityRetrievalWeighter` | Re-rank: weighted_score = 0.8×semantic + 0.2×identity affinity |
| `consolidation_weighting.py` | ~210 | 7C | `IdentityConsolidationWeighter` | Ajusta memory importance pre-storage (factor 0.75-1.25) |
| `behavioral_bias.py` | ~500 | 8A | `IdentityBehavioralBias` | Genera `recommended_planner_mode` + `style_bias` (tone, assertiveness, depth, creativity) |
| `prompt_integration.py` | ~195 | 8B | `IdentityPromptIntegrator` | Inyecta `[IDENTITY STYLE PREFERENCES]` block en system prompt |
| `health_monitor.py` | ~320 | 9A | `IdentityHealthMonitor` | Sliding window 50 interacciones, instability_index (0-1), health classification |
| `health_regulation.py` | ~260 | 9B | `IdentityHealthRegulator` | Ajusta governance threshold + identity weight según health status |
| `evolution.py` | ~600 | 10A | `IdentityEvolutionEngine` | Análisis de evolución a largo plazo (200 interacciones, OLS regression) |
| `shadow_simulation.py` | ~528 | 10B | `IdentityShadowSimulator` | Simulación paralela no-mutante de candidato de identidad |
| `version_control.py` | ~710 | 10C | `IdentityVersionControl` | Snapshots inmutables, apply/rollback controlados, SHA-256 verification |

---

## Flujo de Identidad en el Pipeline

```
  PASO 2b ── Memory Bridge ──────────── afinidad per-memory
       │
  PASO 2c ── Retrieval Weighting ────── re-ranking identitario
       │
  PASO 3b ── Context Weighting ──────── tags [ALIGNED]/[LOW] en contexto
       │
  PASO 3c ── Decision Modulation ────── alignment decisión-identidad
       │
  PASO 3d ── Confidence Engine ──────── confidence score 0-1
       │
  PASO 3e ── Autonomy Modulation ────── ajuste governance threshold
       │
  PASO 3f ── Behavioral Bias ────────── style_bias + planner_mode
       │
  PASO 3g ── Prompt Integration ─────── inyección en system prompt
       │
  PASO 6d ── Consolidation Weight ───── ajuste importance pre-storage
       │
  PASO 7c ── Enforcement ────────────── drift check (similarity vs threshold)
       │
  PASO 7d ── Policy ─────────────────── severity → action mapping
       │
  PASO 7e ── Feedback ───────────────── correction hints (condicional)
       │
  PASO 9a ── Health Monitor ─────────── monitoreo longitudinal
       │
  PASO 9b ── Health Regulation ──────── ajuste adaptativo
       │
  PASO 9c ── Evolution ──────────────── propuesta de evolución
       │
  PASO 9d ── Shadow Simulation ──────── simulación paralela
       │
  PASO 9e ── Version Control ────────── candidato de versión
```

---

## Capas de Identidad (4)

| # | Capa | Estado | Descripción |
|---|------|--------|-------------|
| 1 | **Prompt Engineering** | ✅ | [persona.yaml](../config/README.md) → system prompt |
| 2 | **RAG Knowledge Base** | ✅ | ChromaDB semantic → [memoria](memory.md) |
| 3 | **Human Feedback Training** | ✅ | Correcciones en [procedural memory](memory.md) via [Training](../dashboard/training-center.md) |
| 4 | **QLoRA Fine-Tuning** | 🔲 | PEFT + Unsloth para modelo privado (futuro) |

---

## IdentityProfile — schema.py

Modelo Pydantic central:

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `version` | str | Semántico `vX.Y.Z` |
| `principal_name` | str | "Harold Vélez" |
| `big_five` | dict | 5 floats 0.0-1.0 (OCEAN) |
| `values` | list[str] | Lista ordenada de valores |
| `communication_style` | dict | 6 floats (formality, directness, etc.) |
| `boundaries` | list[str] | Reglas duras |
| `writing_style` | dict | Estilo de escritura |
| `expertise` | dict | Áreas de experiencia |
| `decision_making` | dict | Patrones de decisión |
| `baseline_embedding` | list[float] | Vector 4096-dim (Qwen3-Embedding-8B via EmbeddingRouter) |
| `drift_threshold` | float | 0.78 default |
| `content_hash` | str | SHA-256 del contenido |

Properties: `has_baseline`, `to_persistable()`.

---

## Confidence Engine — Detalles

Agrega 4 señales con pesos:

| Señal | Peso | Fuente |
|-------|------|--------|
| `similarity` | 0.30 | Enforcement (paso 7c) |
| `memory_affinity` | 0.20 | Memory Bridge (paso 2b) |
| `decision_alignment` | 0.25 | Decision Modulation (paso 3c) |
| `policy_severity` | 0.25 | Policy (paso 7d) |

Niveles: high (≥0.75, +1), medium (0.50-0.74, 0), low (<0.50, -1).

---

## Health Monitor — Clasificación

| Instability Index | Clasificación | Significado |
|-------------------|---------------|-------------|
| < 0.25 | `stable` | Identidad consistente |
| 0.25 - 0.49 | `monitor` | Observación recommended |
| 0.50 - 0.69 | `unstable` | Intervención sugerida |
| ≥ 0.70 | `critical` | Intervención urgente |

---

## Evolution Engine — Criterios

**Evolución se propone cuando**:
- sustained_high_confidence (últimos 10 > 0.75) AND
- (similarity_trend > 0 OR sustained_similarity_shift) AND
- drift_rate < 0.25 AND avg_confidence > 0.70

**Evolución se rechaza cuando**:
- avg_instability > 0.60 OR high_severity_rate > 0.20

**Version bump**:
- shift_magnitude < 0.05 → no evolution
- 0.05 - 0.10 → minor version
- > 0.10 → major version

Siempre `requires_human_approval: True`. UI: [Identity Governance](../dashboard/identity-governance.md)

---

## Gestión desde Dashboard

| Página | Funciones de Identidad |
|--------|----------------------|
| [Identity Studio](../dashboard/identity-studio.md) | Editar Big Five, comm style, values, boundaries |
| [Identity Governance](../dashboard/identity-governance.md) | Versiones, evolución, shadow simulation, health |
| [Evaluation Dashboard](../dashboard/evaluation-dashboard.md) | Alignment reports, drift severity, policy action badges |
| [Analytics](../dashboard/analytics.md) | Identity fidelity score |

---

## Temas Relacionados

- [Pipeline](pipeline.md) — Los 17 pasos de identidad integrados
- [Memoria](memory.md) — Memory Bridge y Retrieval Weighting
- [Cognición](cognition.md) — DecisionEngine con identity_profile
- [Evaluación](evaluation.md) — Alignment evaluator con baseline embedding
- [Configuración](../config/README.md) — persona.yaml, governance.yaml (identity_control)
- [Base de Datos](database.md) — config_versions, evaluations, identity signals
