# Arquitectura General

> [← Volver al índice](../README.md)

---

## Sobre este documento

Este documento es el **mapa general de la arquitectura técnica** de ADLRA — la vista de alto nivel que muestra cómo todas las piezas del sistema encajan entre sí. Si los demás documentos de esta carpeta son los capítulos de un libro, este es el índice comentado que te dice qué hay en cada capítulo y por qué importa.

### ¿Qué cubre este documento?

Aquí encontrarás el **stack tecnológico completo** (qué lenguajes, frameworks y herramientas se usan), la **estructura de archivos del backend** (dónde vive cada módulo), las **métricas generales del proyecto** (líneas de código, archivos, endpoints) y un **diagrama de alto nivel** que muestra cómo fluye la información desde que un usuario escribe un mensaje hasta que Django responde.

### ¿Cuál es su función en la arquitectura?

Este documento sirve como **punto de orientación**. No entra en profundidad en ningún subsistema — eso lo hacen los documentos especializados. En cambio, te da el contexto necesario para entender dónde encaja cada pieza antes de sumergirte en los detalles. Es el documento que deberías leer primero después del [índice general](../README.md).

### ¿Cómo afecta al comportamiento de Django?

Indirectamente, todo. Este documento describe el ecosistema completo que hace posible que Django exista: el backend FastAPI que procesa sus pensamientos, los 5 agentes LLM que generan sus respuestas, los 22 módulos de identidad que lo mantienen fiel a Harold, los 4 niveles de memoria que le dan contexto, y el sistema teleológico que le da propósito. Sin alguna de estas piezas, Django sería un chatbot genérico en lugar de un delegado personalizado.

### ¿Cómo interactúa con las demás piezas?

Este documento **enlaza a todos los demás documentos de arquitectura**:
- El [Pipeline](pipeline.md) es el corazón — los 25+ pasos que procesan cada mensaje
- Los [Agentes](agents.md) son los ejecutores LLM — quiénes generan las respuestas
- La [Cognición](cognition.md) es el analizador determinístico — decide estrategia sin usar LLM
- La [Memoria](memory.md) es el contexto — 4 niveles desde conversación actual hasta conocimiento permanente
- La [Identidad](identity.md) es el guardián — 22 módulos que aseguran que Django suene como Harold
- La [Evaluación](evaluation.md) es el crítico — 5 módulos que califican cada respuesta
- Los [Eventos](events.md) son el sistema nervioso — transmiten lo que pasa a todo el sistema
- La [Base de Datos](database.md) es la persistencia — dónde se guarda todo
- El [Startup](startup.md) es el arranque — cómo se inicializa todo en el orden correcto
- La [Seguridad](security.md) es el escudo — filtra inputs maliciosos
- La [Teleología](teleology.md) es el propósito — metas, prioridades y recompensas

Cada uno de esos documentos explica una pieza en detalle, pero este documento te da la visión panorámica de cómo fluyen los datos entre ellas.

---

## Resumen Ejecutivo

ADLRA es un clon virtual autónomo diseñado para emular a Harold Vélez. El sistema consta de:

- **Backend**: FastAPI con [pipeline de 25+ pasos](pipeline.md), [5 agentes LLM](agents.md), [22 módulos de identidad](identity.md), [4 niveles de memoria](memory.md), [sistema teleológico](teleology.md)
- **Frontend**: [Dashboard Next.js 15](../dashboard/README.md) con 15 páginas de control
- **Integraciones**: [Bot de Discord](../integrations/README.md) para conversación natural
- **LLMs**: [Cadena Gemini → Groq → Ollama](../integrations/README.md) con circuit breaker
- **Costo operativo**: $0/mes (todos servicios en tier gratuito)

---

## Stack Tecnológico

| Capa | Tecnología | Detalles |
|------|-----------|----------|
| **Backend** | Python 3.11, FastAPI, uvicorn | Puerto 8000, `--reload` en dev |
| **Frontend** | Next.js 15 (App Router), React 19, TypeScript 5.7 | Puerto 3000, proxy `/api/*` → backend |
| **UI** | shadcn/ui, Tailwind CSS 3.4, lucide-react, CVA | Tema oscuro laboratorio |
| **Estado** | Zustand 5 | Persistencia localStorage para chat |
| **LLM** | Gemini 2.5 Flash, Groq Llama 3.3 70B, Ollama | [Cadena de fallback](../integrations/README.md) |
| **Vector DB** | ChromaDB (local) | Embedding all-MiniLM-L6-v2 (384-dim) |
| **SQL DB** | Neon Postgres (remoto) | [Ver database.md](database.md) |
| **Procedural DB** | SQLite | Correcciones, workflows |
| **Charts** | recharts, SVG inline | Visualizaciones [analytics](../dashboard/analytics.md) |
| **Flow Viz** | @xyflow/react v12 | [Trazas cognitivas](../dashboard/cognitive-trace.md) |
| **Discord** | discord.py, httpx | [Bot + webhook](../integrations/README.md) |

---

## Estructura de Archivos del Repositorio

```
iame.lol/
├── agent/                                    # Backend Python FastAPI
│   ├── src/                                  # 25,217 líneas, 89 archivos
│   │   ├── agents/          (878 ln, 8 files) # → agents.md
│   │   ├── api/           (4,110 ln, 3 files) # → ../api/README.md
│   │   ├── cognition/       (349 ln, 3 files) # → cognition.md
│   │   ├── db/            (1,916 ln, 3 files) # → database.md
│   │   ├── evaluation/    (2,151 ln, 6 files) # → evaluation.md
│   │   ├── events/          (151 ln, 2 files) # → events.md
│   │   ├── flows/         (3,116 ln, 5 files) # → pipeline.md
│   │   ├── governance/         (1 file)       # Stub
│   │   ├── identity/      (6,080 ln, 22 files)# → identity.md
│   │   ├── memory/        (1,636 ln, 4 files) # → memory.md
│   │   ├── router/          (630 ln, 3 files) # → ../integrations/README.md
│   │   ├── security/        (429 ln, 4 files) # → security.md
│   │   ├── skills/        (1,364 ln, 6 files) # → ../dashboard/skill-manager.md
│   │   ├── teleology/    (2,294 ln, 11 files) # → teleology.md
│   │   ├── trace/           (~408 ln, 2 files) # → events.md
│   │   ├── training/        (521 ln, 2 files) # → ../dashboard/training-center.md
│   │   ├── config.py                  (117 ln)# → ../config/README.md
│   │   ├── service_logger.py          (263 ln)# Rotating file logger
│   │   └── watchdog.py               (136 ln)# Service health watchdog
│   ├── tests/                                 # → ../development/README.md
│   └── configs → ../configs                   # Symlink
├── dashboard/                                 # → ../dashboard/README.md
│   ├── app/                                   # 15 rutas (App Router)
│   ├── components/                            # 31 archivos en 7 directorios
│   ├── lib/                                   # api.ts, store.ts, hooks/, i18n/
│   └── ...
├── configs/                                   # → ../config/README.md
├── scripts/                                   # → ../integrations/README.md
├── .vscode/tasks.json                         # → ../config/README.md
└── docs/                                      # ← Esta documentación
```

---

## Diagrama de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────┐
│                        DASHBOARD (Next.js 15)                   │
│  15 páginas · Zustand store · WebSocket · api.ts (111 methods)  │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTP/WS :3000 → :8000
┌──────────────────────────────┴──────────────────────────────────┐
│                         FastAPI Backend                          │
│  ┌──────────┐  ┌───────────────────────────────────────────┐    │
│  │ Routes   │→ │           ORCHESTRATOR (25+ steps)        │    │
│  │ (116 ep) │  │  Classify → Cognition → Memory → LLM →   │    │
│  └──────────┘  │  Identity → Governance → Eval → Store     │    │
│                └──────────┬────────────────────────────────┘    │
│  ┌────────────────────────┼────────────────────────────────┐    │
│  │        │           │          │          │          │    │    │
│  │  5 Agents    4-Tier Mem   22 Identity  Teleology  Eval  │    │
│  │  (LLM)      (ChromaDB)   (Phases)     (Goals)    (5)   │    │
│  └────────────────────────────────────────────────────────┘    │
│                               │                                 │
│  ┌──────────────┐  ┌─────────┴─────────┐  ┌───────────────┐   │
│  │ Model Router │  │    EventBus       │  │  Persistence  │   │
│  │ (3 providers)│  │ (WS + DB + mem)   │  │  (Postgres)   │   │
│  └──────────────┘  └───────────────────┘  └───────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         │                                           │
    ┌────┴────┐                              ┌───────┴───────┐
    │ Discord │                              │ Neon Postgres  │
    │   Bot   │                              │ ChromaDB       │
    └─────────┘                              │ SQLite         │
                                             └───────────────┘
```

---

## Módulos del Backend

| Módulo | Doc | Líneas | Función Principal |
|--------|-----|--------|-------------------|
| Orchestrator | [pipeline.md](pipeline.md) | 2,716 | Pipeline central 25+ pasos |
| Agents | [agents.md](agents.md) | 878 | 5 agentes LLM especializados |
| Cognition | [cognition.md](cognition.md) | 349 | DecisionEngine + Planner (determinístico) |
| Memory | [memory.md](memory.md) | 1,636 | 4-tier memory system |
| Identity | [identity.md](identity.md) | 6,080 | 22 módulos de identidad |
| Teleology | [teleology.md](teleology.md) | 2,294 | Metas, planes, prioridades |
| Evaluation | [evaluation.md](evaluation.md) | 2,151 | 5 módulos heurísticos |
| Events + Trace | [events.md](events.md) | 552 | EventBus + TraceCollector |
| Security | [security.md](security.md) | 429 | Sanitización + middleware |
| Database | [database.md](database.md) | 1,916 | Postgres + ChromaDB + SQLite |
| Router | [../integrations/README.md](../integrations/README.md) | 630 | Model Router + Circuit Breaker |

---

## Temas Relacionados

- [Cómo arranca el sistema](startup.md) — AppState, lifespan, inicialización
- [API REST completa](../api/README.md) — 116 endpoints
- [Dashboard](../dashboard/README.md) — 15 páginas de control
- [Configuración](../config/README.md) — persona.yaml, models.json, governance.yaml
- [Desarrollo](../development/README.md) — Convenciones, tests, roadmap
