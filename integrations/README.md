# Integraciones

> [← Configuración](../config/README.md) · [Desarrollo →](../development/README.md)

---

## Sobre este documento

Este documento describe las **integraciones externas** de Doe — las piezas que lo conectan con el mundo fuera del dashboard. Actualmente la integración principal es el **Model Router** (la capa que gestiona múltiples proveedores LLM con fallback automático y circuit breaker).

### ¿Qué cubre este documento?

Documenta: el **Model Router** (530 líneas) con su cadena de fallback Gemini → Groq (2-level) + Claude on-demand, conteo real de tokens, estimación de costos, hot-reload, 2 perfiles, y circuit breaker por proveedor (CLOSED/OPEN/HALF_OPEN).

### ¿Cuál es su función en la arquitectura?

Las integraciones son los **puentes hacia afuera**:
- El **Model Router** es el único punto de salida hacia LLMs — ningún componente llama directamente a Gemini, Groq o Claude. Todo pasa por el router, que maneja fallback, retry, circuit breaking y conteo de tokens

### ¿Cómo afecta al comportamiento de Doe?

- El **Model Router** determina la *calidad* de las respuestas: Gemini (1M contexto, más capaz) vs Groq (rápido pero menos contexto). Si Gemini falla, Doe automáticamente pasa a Groq — sin que el usuario note la transición
- El **circuit breaker** evita hammering a proveedores caídos — cuando detecta fallos consecutivos, salta el proveedor hasta que se recupere

### ¿Cómo interactúa con las demás piezas?

- **[Pipeline](../architecture/pipeline.md) paso 7**: toda generación LLM pasa por el Model Router. El router es llamado por los [Agentes](../architecture/agents.md) durante la generación
- **[Config](../config/README.md)**: `models.json` define proveedores, fallback chain, asignaciones y perfiles. `.env` contiene las API keys
- **[Dashboard](../dashboard/model-manager.md)**: Model Manager permite cambiar perfiles y asignaciones, y testear proveedores — los cambios llegan al router via `PUT /models/*` y `reload_config()`
- **[Dashboard](../dashboard/command-center.md)**: el Command Center muestra LEDs de estado de proveedores (verde/rojo/amber según el circuit breaker)
- **[Base de Datos](../architecture/database.md)**: el callback de persistencia de tokens (conectado en [Startup](../architecture/startup.md)) guarda cada llamada LLM en la tabla `token_usage` de Postgres
- **[API](../api/README.md)**: las integraciones externas usan `POST /api/chat` como su interfaz con el backend — todo pasa por el pipeline completo

---

## Model Router — `src/router/` (~893 líneas)

### model_router.py (530 ln)

Cadena de fallback: **Gemini 2.5 Flash → Groq Llama 3.3 70B** (2-level) + **Claude Sonnet 4** (on-demand only, not in fallback chain)

| Feature | Detalle |
|---------|---------|
| `generate()` | `(prompt, role, system_prompt, temperature, max_tokens)` → `ModelResponse` |
| `generate_with_claude()` | `(prompt, system_prompt, temperature, max_tokens)` → `ModelResponse` — bypasses role routing, used by SkillForge |
| Token counting | Real por proveedor: Gemini `usage_metadata`, Groq `usage.total_tokens`, Claude `usage.input_tokens`+`output_tokens` |
| Costo | Gemini $0.15/1M, Groq $0.05/1M, Claude $3.0/1M |
| Hot-reload | `reload_config()` — recarga [models.json](../config/README.md) sin restart |
| Perfiles | balanced, max_quality |
| Persistence | Callback de persistencia de tokens wired al [startup](../architecture/startup.md) |

### circuit_breaker.py (153 ln)

Circuit breaker **por proveedor**:

| Estado | Significado |
|--------|-------------|
| **CLOSED** (verde) | Normal — requests pasan |
| **OPEN** (rojo) | Fallo detectado — requests se saltan este proveedor |
| **HALF_OPEN** (amber) | Prueba — un request de test para verificar recuperación |

- Threshold configurable de fallos consecutivos
- Timeout de recovery
- Visible en [Command Center](../dashboard/command-center.md) Health Bar (LEDs) y [Model Manager](../dashboard/model-manager.md)

---

## Temas Relacionados

- [Agentes](../architecture/agents.md) — Asignación de modelos
- [Pipeline](../architecture/pipeline.md) — Cómo se procesan mensajes via API
- [Model Manager](../dashboard/model-manager.md) — UI de gestión
- [Configuración](../config/README.md) — models.json, .env
