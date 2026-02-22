# Integraciones

> [← Configuración](../config/README.md) · [Desarrollo →](../development/README.md)

---

## Sobre este documento

Este documento describe las **integraciones externas** de Django — las piezas que lo conectan con el mundo fuera del dashboard. Actualmente hay 3 integraciones operativas: el **Model Router** (la capa que gestiona múltiples proveedores LLM con fallback automático y circuit breaker), el **Bot de Discord** (que permite a Django participar en conversaciones naturales en un servidor Discord), y el **Discord Webhook** (para publicar actualizaciones en un canal Discord).

### ¿Qué cubre este documento?

Documenta: el **Model Router** (476 líneas) con su cadena de fallback Gemini → Groq → Ollama, conteo real de tokens, estimación de costos, hot-reload, 4 perfiles, y circuit breaker por proveedor (CLOSED/OPEN/HALF_OPEN). El **Bot de Discord** (572 líneas) con su flujo de interacción completo (on_message → call_django_api → respuesta), conversaciones per-channel, member awareness, real-time context injection, y 4 comandos. Y el **Discord Webhook** (188 líneas) para publicar mensajes y embeds al canal #updates-django.

### ¿Cuál es su función en la arquitectura?

Las integraciones son los **puentes hacia afuera**:
- El **Model Router** es el único punto de salida hacia LLMs — ningún componente llama directamente a Gemini, Groq u Ollama. Todo pasa por el router, que maneja fallback, retry, circuit breaking y conteo de tokens
- El **Bot de Discord** es el segundo canal de interacción (después del dashboard) — permite que Django converse naturalmente en un contexto grupal, con awareness de otros miembros y del historial del canal
- El **Discord Webhook** es el canal de publicación — permite que el sistema (o Harold) publique actualizaciones formateadas en Discord

### ¿Cómo afecta al comportamiento de Django?

- El **Model Router** determina la *calidad* de las respuestas: Gemini (1M contexto, más capaz) vs Groq (rápido pero menos contexto) vs Ollama (local, privado pero menos capaz). Si Gemini falla, Django automáticamente pasa a Groq, y si Groq falla, a Ollama — sin que el usuario note la transición
- El **circuit breaker** evita hammering a proveedores caídos — cuando detecta fallos consecutivos, salta el proveedor hasta que se recupere
- El **Bot de Discord** opera con **cognitive_mode=2** (Memory+LLM) por defecto — Django en Discord está limitado a conocimiento aprendido, sin web search
- En Discord, Django decide **cuándo hablar** usando el token `[SILENT]` — no responde a todo, participa naturalmente
- Cada canal Discord tiene su propia **working memory aislada** — Django no mezcla conversaciones de distintos canales

### ¿Cómo interactúa con las demás piezas?

- **[Pipeline](../architecture/pipeline.md) paso 7**: toda generación LLM pasa por el Model Router. El router es llamado por los [Agentes](../architecture/agents.md) durante la generación
- **[Config](../config/README.md)**: `models.json` define proveedores, fallback chain, asignaciones y perfiles. `.env` contiene las API keys
- **[Dashboard](../dashboard/model-manager.md)**: Model Manager permite cambiar perfiles y asignaciones, y testear proveedores — los cambios llegan al router via `PUT /models/*` y `reload_config()`
- **[Dashboard](../dashboard/command-center.md)**: el Command Center muestra LEDs de estado de proveedores (verde/rojo/amber según el circuit breaker)
- **[Memoria](../architecture/memory.md)**: cada canal Discord mantiene working memory aislada via `conversation_id`. El comando `!reset` limpia esa working memory
- **[Base de Datos](../architecture/database.md)**: el callback de persistencia de tokens (conectado en [Startup](../architecture/startup.md)) guarda cada llamada LLM en la tabla `token_usage` de Postgres
- **[Eventos](../architecture/events.md)**: el bot emite los mismos eventos que el chat de dashboard (`chat.message_received`, `agent_state.*`, `chat.response_generated`)
- **[API](../api/README.md)**: el bot usa `POST /api/chat` como su única interfaz con el backend — todo pasa por el pipeline completo

---

## Model Router — `src/router/` (630 líneas)

### model_router.py (476 ln)

Cadena de fallback: **Gemini 2.5 Flash → Groq Llama 3.3 70B → Ollama**

| Feature | Detalle |
|---------|---------|
| `generate()` | `(prompt, role, system_prompt, temperature, max_tokens)` → `ModelResponse` |
| Token counting | Real por proveedor: Gemini `usage_metadata`, Groq `usage.total_tokens`, Ollama `eval_count` |
| Costo | Gemini $0.15/1M, Groq $0.05/1M, Ollama $0 |
| Hot-reload | `reload_config()` — recarga [models.json](../config/README.md) sin restart |
| Perfiles | balanced, max_quality, privacy_mode, budget_mode |
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

## Bot de Discord — `scripts/discord_bot.py` (572 líneas)

### Arquitectura

| Componente | Detalle |
|-----------|---------|
| Dependencies | discord.py, httpx (en agent/venv) |
| Token | `DISCORD_BOT_TOKEN` en `.env` |
| Task | `Discord: Django Bot` en [tasks.json](../config/README.md) |
| PID file | `agent/discord_bot.pid` (atexit cleanup) |
| API | Todas las interacciones via `POST /api/chat` al backend |

### Flujo de Interacción

```
User escribe en Discord → on_message()
  │ Track user en known_users (display_name, username, message_count)
  │ Store en channel_buffers[channel_id] (deque maxlen=30)
  │
  ├─ Comando? (!reset, !who, !full, !memory) → handle directo
  │
  ├─ Debe responder? (mention, reply, name trigger, o LLM decide)
  │   │
  │   ├─ Build GROUP_CONTEXT_PROMPT:
  │   │   • [CONTEXTO EN TIEMPO REAL — DISCORD]
  │   │   • Server, channel, member list (guild.members)
  │   │   • Últimos 30 mensajes del channel buffer
  │   │   • Override: ignora Knowledge Status restrictions
  │   │
  │   ├─ POST /api/chat:
  │   │   • message tagged "[user_name] message"
  │   │   • conversation_id per-channel
  │   │   • cognitive_mode = 2
  │   │   • Orchestrator procesa pipeline completo
  │   │
  │   ├─ Response [SILENT]? → no responde (LLM eligió callar)
  │   └─ Otherwise → typing simulation → split ≤2000 chars → send
  │
  └─ Cooldown: MIN_RESPONSE_GAP = 8s entre respuestas por canal
```

### Características

| Feature | Detalle |
|---------|---------|
| **Per-channel conversations** | Cada canal tiene conversation_id y [working memory](../architecture/memory.md) aislada |
| **Member awareness** | `guild.chunk()` + `build_member_list()` inyectado en cada prompt |
| **Real-time context** | GROUP_CONTEXT_PROMPT overrides Knowledge Status restrictions |
| **Cognitive mode 2** | [Memory+LLM](../architecture/pipeline.md) para todas las interacciones |
| **Natural participation** | LLM decide cuándo hablar via `[SILENT]` token |

### Comandos

| Comando | Efecto |
|---------|--------|
| `!full <msg>` | Mode 1 (Full) para ese mensaje |
| `!memory <msg>` | Mode 3 (Memory Only) |
| `!reset` / `!nueva` / `!new` | Clear conversation + working memory |
| `!who` / `!quien` | Listar miembros del servidor |

---

## Discord Webhook — `scripts/discord_post.py` (188 líneas)

Publisher a canal `#updates-django` via webhook URL en `.env`.

| Flag | Efecto |
|------|--------|
| (texto plano) | Auto-split a 2000 chars |
| `--embed` | Rich embed con título y color |
| `--file` | Lee contenido de archivo |
| `--title` | Título del embed |
| `--color` | Color hex (default: 0x5865F2 blurple) |
| `--username` | Display name (default: "Django") |

```bash
agent\venv\Scripts\python.exe scripts\discord_post.py "Mensaje"
agent\venv\Scripts\python.exe scripts\discord_post.py --file msg.md --embed --title "Update"
```

---

## Temas Relacionados

- [Agentes](../architecture/agents.md) — Asignación de modelos
- [Pipeline](../architecture/pipeline.md) — Cómo se procesan mensajes Discord
- [Model Manager](../dashboard/model-manager.md) — UI de gestión
- [Configuración](../config/README.md) — models.json, .env
- [Memoria](../architecture/memory.md) — Working memory per-channel
