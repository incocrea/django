# Configuración del Proyecto

> [← Dashboard](../dashboard/README.md) · [Integraciones →](../integrations/README.md)

---

## Sobre este documento

Este documento describe los **6 archivos de configuración** que definen quién es Django, qué modelos LLM usa, qué habilidades tiene, qué límites no puede cruzar, qué variables de entorno necesita, y cómo se gestionan los servicios. Son los archivos que Harold modifica (directamente o via el dashboard) para moldear el comportamiento de Django sin tocar código.

### ¿Qué cubre este documento?

Documenta en detalle: **persona.yaml** (la identidad de Harold: Big Five, valores, estilo de comunicación, boundaries, expertise, autonomía), **models.json** (proveedores LLM, fallback chain, asignaciones por agente, 2 perfiles), **skills.json** (5 habilidades registradas con riesgo y dependencias), **governance.yaml** (niveles de autonomía 0-4, clasificación de riesgos, operaciones prohibidas, política de identidad, privacidad), **config.py** (variables de entorno via Pydantic BaseSettings), y **tasks.json** (VS Code tasks para gestión de servicios).

### ¿Cuál es su función en la arquitectura?

La configuración es la **capa declarativa** — define *qué* sin definir *cómo*. `persona.yaml` dice quién es Harold pero no cómo implementar la fidelidad; `governance.yaml` dice qué está prohibido pero no cómo bloquearlo. Los módulos de código (identidad, agentes, pipeline, gobernanza) leen estos archivos y los interpretan. Esto permite cambiar el comportamiento de Django sin modificar código — un cambio en persona.yaml se refleja en la próxima interacción.

### ¿Cómo afecta al comportamiento de Django?

**Todo empieza aquí**:
- `persona.yaml` define literalmente *quién es Django*: su personalidad, valores, estilo. Si cambias `openness` de 0.81 a 0.30, Django se vuelve más conservador y menos creativo
- `models.json` define *con qué cerebro piensa*: cambiar el perfil a `max_quality` hace que todo se procese con Gemini (máxima calidad, mayor costo)
- `governance.yaml` define *qué puede y qué no puede hacer*: las operaciones prohibidas son límites duros que ningún agente puede cruzar
- `skills.json` define *qué herramientas tiene*: deshabilitar `web-research` impide que Django busque en internet
- `.env` define *las credenciales*: sin `GEMINI_API_KEY`, Django solo puede usar Groq como fallback

### ¿Cómo interactúa con las demás piezas?

- **[Identidad](../architecture/identity.md)**: el `IdentityManager` lee `persona.yaml`, construye un `IdentityProfile` con hash SHA-256 y embedding baseline de 4096 dimensiones (Qwen3-Embedding-8B via EmbeddingRouter), lo versiona en Postgres, y lo inyecta en el `DecisionEngine`. `governance.yaml → identity_control` define el mapeo severity→action para la política de identidad
- **[Agentes](../architecture/agents.md)**: `persona.yaml` define el system prompt base de cada agente. `models.json` define qué modelo LLM usa cada uno y la cadena de fallback
- **[Model Router](../integrations/README.md)**: `models.json` es su fuente de verdad. Se puede recargar en caliente via `POST /router/reload`
- **[Pipeline](../architecture/pipeline.md)**: `persona.yaml` se recarga en caliente via `POST /persona/reload` → todos los agentes actualizan su prompt
- **[Dashboard](../dashboard/README.md)**: Identity Studio escribe directamente en `persona.yaml`. Model Manager escribe en `models.json`. Governance Console lee `governance.yaml`
- **[Startup](../architecture/startup.md)**: `config.py` (Pydantic BaseSettings) es lo primero que se carga. De ahí se derivan todas las demás configuraciones
- **[Tasks](../config/README.md)**: `tasks.json` define cómo se lanzan los servicios (backend, dashboard, watchdog, Discord bot) via VS Code

---

## Archivos de Configuración

| Archivo | Líneas | Propósito | Leído por |
|---------|--------|-----------|-----------|
| `configs/persona.yaml` | 89 | Identidad del principal | [IdentityManager](../architecture/identity.md), `IdentityCoreAgent`, endpoints persona |
| `configs/models.json` | 138 | Proveedores LLM, fallback, perfiles | [Model Router](../integrations/README.md), [Model Manager](../dashboard/model-manager.md) |
| `configs/skills.json` | 71 | Registro de habilidades | [Skill Manager](../dashboard/skill-manager.md) |
| `configs/governance.yaml` | 283 | Autonomía, riesgos, prohibiciones | [Governance Console](../dashboard/governance-console.md), [Identity Policy](../architecture/identity.md) |
| `agent/src/config.py` | 117 | Variables de entorno (Pydantic BaseSettings) | Todo el backend |
| `.vscode/tasks.json` | 78 | Tasks de servicio | VS Code |

---

## persona.yaml — Identidad del Principal

Define quién es Harold Vélez para el sistema.

| Sección | Contenido |
|---------|-----------|
| `principal_name` | "Harold Vélez" |
| `personality` (Big Five) | openness 0.81, conscientiousness 0.74, extraversion 0.84, agreeableness 0.75, neuroticism 0.23 |
| `values` | efficiency, quality, innovation, integrity, reliability, respect |
| `communication` | formality 0.35, directness 0.77, verbosity 0.3, emotional_expression 0.68, humor 0.91, empathy 0.82 |
| `decision_making` | risk_tolerance moderate, analysis_depth thorough, stakeholder_weight high, data_vs_intuition 0.7 |
| `boundaries` | 5 reglas duras |
| `expertise` | Primary: fullstack dev, AI/ML, project management. Secondary: 3 más |
| `autonomy` | current_level: 0 (Observer). Per-category: todas en 0 |

**Flujo de actualización**:
```
Dashboard Identity Studio → PUT /persona/* → escribe YAML
  → agent_crew.reload_persona() → todos los agentes actualizan system prompt
  → IdentityManager detecta hash change → rebuild IdentityProfile + baseline embedding
```

---

## models.json — Configuración de LLMs

| Sección | Contenido |
|---------|-----------|
| `providers` | Gemini (1M ctx), Groq (128K ctx) |
| `fallback_chain` | gemini → groq |
| `agent_assignments` | identity_core→Gemini, business→Groq, communication→Gemini, technical→Groq, governance→Gemini |
| `profiles` | balanced (default), max_quality (all Gemini) |
| `embedding` | Qwen3-Embedding-8B via Ollama (4096-dim), managed by EmbeddingRouter |

**Flujo**: [Model Manager](../dashboard/model-manager.md) → `PUT /models/assignment` o `PUT /models/profile` → escribe JSON → `model_router.reload_config()`.

---

## skills.json — Registro de Habilidades

| Skill ID | Tipo | Riesgo | Habilitado | Dependencias |
|----------|------|--------|------------|-------------|
| `web-research` | tool | low | sí | — |
| `email-draft` | tool | low | sí | identity-core |
| `document-gen` | tool | low | sí | identity-core |
| `learn-topic` | tool | low | sí | web-research, identity-core |
| `repo-explorer` | tool | low | sí | identity-core |

---

## governance.yaml — Constitución del Agente

| Sección | Contenido |
|---------|-----------|
| `autonomy_levels` (0-4) | Observer → Assistant → Delegate → Autonomous → Trusted |
| `risk_levels` (4) | low (auto@L1), medium (auto@L2), high (approval, auto@L3), critical (2FA, auto@L4) |
| `forbidden` (7 reglas) | No bypass identity, no ocultar naturaleza AI, no auto-modificar governance, no deshabilitar audit |
| `identity_control` | Mapeo severity → action para [Identity Policy](../architecture/identity.md) |
| `self_modification` | File permission matrix, safety checks |
| `monitoring` | Drift detection (threshold 0.85), performance tracking |
| `privacy` | Procesamiento local requerido para muestras/correcciones; nunca enviar datos de identidad a cloud |

---

## config.py — Variables de Entorno

Pydantic `BaseSettings`, carga desde `.env` via `@lru_cache`.

| Grupo | Variables | Defaults |
|-------|----------|----------|
| App | `app_name`, `app_version`, `environment`, `debug` | "Django", "2.0.0", "development", True |
| API | `api_host`, `api_port` | "0.0.0.0", 8000 |
| LLM | `gemini_api_key`, `groq_api_key`, `ollama_base_url` | None, None, localhost:11434 |
| DB | `database_url`, `chroma_persist_directory` | None, "./chroma_data" |
| Web Research | `tavily_api_key` | None |
| Governance | `default_autonomy_level`, `principal_name` | 0, "Principal" |
| Embedding | `embedding_model`, `embedding_dimensions` | "qwen3-embedding", 4096 |

Properties booleanas: `has_gemini`, `has_groq`, `has_tavily`, `has_database`, `has_r2`, `has_supabase`.

---

## .vscode/tasks.json — Gestión de Servicios

| Task | Comando | instanceLimit |
|------|---------|---------------|
| `Backend: FastAPI Server` | `uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --reload` | 1 |
| `Dashboard: Next.js Dev Server` | `Remove-Item .next; npx next dev --port 3000` | 1 |
| `Health Watchdog` | `python -m src.watchdog` | 1 |
| `Discord: Django Bot` | `venv/Scripts/python.exe scripts/discord_bot.py` | 1 |
| `Start All Services` | dependsOn las 4 anteriores (parallel) | — |

> **Regla crítica**: Todos los servicios se inician via VS Code Tasks, NUNCA con terminales background.

---

## Temas Relacionados

- [Identidad](../architecture/identity.md) — Consume persona.yaml
- [Agentes](../architecture/agents.md) — Consume models.json, persona.yaml
- [Integrations](../integrations/README.md) — Model Router, Discord bot
- [Gobernanza](../dashboard/governance-console.md) — Consume governance.yaml
- [Startup](../architecture/startup.md) — Secuencia de carga de configuración
