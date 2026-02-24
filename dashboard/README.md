# Dashboard — Next.js 15

> [← API](../api/README.md) · [Configuración →](../config/README.md)

**Directorio**: `dashboard/` (23,249 líneas en 59 archivos)

---

## Stack

| Tecnología | Versión | Uso |
|-----------|---------|-----|
| Next.js | 15 (App Router) | Framework |
| React | 19 | UI Library |
| TypeScript | 5.7 | Tipos |
| Tailwind CSS | 3.4 | Estilos |
| shadcn/ui | — | Primitivos UI |
| Zustand | 5 | Estado global |
| recharts | — | Gráficas |
| @xyflow/react | v12 | Legacy (no longer used in trace page) |
| lucide-react | — | Iconos |

---

## 15 Páginas

| Ruta | Página | Descripción | Doc |
|------|--------|-------------|-----|
| `/` | Command Center | KPIs, agent status, health, actividad | [→](command-center.md) |
| `/chat` | Chat | Conversación con el delegado | [→](chat.md) |
| `/docs` | Documentation Viewer | 3-panel: file tree, markdown, ToC | [→](docs.md) |
| `/identity` | Identity Studio | Editor de personalidad Big Five | [→](identity-studio.md) |
| `/training` | Training Center | Correcciones + carga con dedup semántica | [→](training-center.md) |
| `/testing` | Testing Playground | Escenarios, A/B compare | [→](testing-playground.md) |
| `/models` | Model Manager | Proveedores LLM, perfiles | [→](model-manager.md) |
| `/skills` | Skill Manager | Skills toggle, learn-topic, repo-explorer | [→](skill-manager.md) |
| `/memory` | Memory Lab | Explorar/editar memorias | [→](memory-lab.md) |
| `/governance` | Governance Console | Reglas, audit, emergency stop | [→](governance-console.md) |
| `/analytics` | Analytics | Métricas de rendimiento | [→](analytics.md) |
| `/trace` | Cognitive Trace | Pipeline horizontal CSS Grid 8 columnas | [→](cognitive-trace.md) |
| `/evaluation` | Evaluation Dashboard | 5 módulos de evaluación | [→](evaluation-dashboard.md) |
| `/identity-governance` | Identity Governance | Versiones, evolución, shadow, health | [→](identity-governance.md) |
| `/goals` | Goals | Sistema teleológico | [→](goals.md) |

---

## Componentes

### Layout (3)

| Componente | Archivo | Función |
|-----------|---------|---------|
| `ClientShell` | `client-shell.tsx` (23 ln) | Root: `useWebSocket()`, `useHealth(5000)`, `useInitialData()`, wraps `I18nProvider` + `TooltipProvider` |
| `Header` | `header.tsx` (82 ln) | Barra superior: principal name, language switcher, DB LED, WS status, agent state badge |
| `Sidebar` | `sidebar.tsx` (159 ln) | 240px: logo, status LED, 15 nav items (flat list, no group headers) |

### Command Center (7)

| Componente | Líneas | Función |
|-----------|--------|---------|
| `KPICard` | 75 | Card con valor, trend, icono |
| `AgentStatusRing` | 87 | Ring 112×112 animado (7 estados) |
| `HealthBar` | 153 | 7 LEDs del sistema |
| `ActivityFeed` | 349 | Timeline de eventos expandibles |
| `PersonaCard` | 92 | Big Five mini-bars + valores |
| `QuickActions` | 165 | 7 action buttons |
| `RouterCard` | 117 | Fallback chain + circuit breakers |

### Chat (2)

| Componente | Líneas | Función |
|-----------|--------|---------|
| `ChatPanel` | 199 | Input, send, mode toggle, clear |
| `MessageBubble` | 100 | Avatar, content, footer (model, mode, sources) |

### Trace (5)

| Componente | Líneas | Función |
|-----------|--------|--------|
| `PipelineView` | 189 | CSS Grid 8-columnas horizontal: headers coloreados, flechas inter-columna, columnas vacías en gris |
| `PipelineNode` | 366 | Nodo accordion: icono, status, latencia, 🧠 LLM badge, 📐 EMB badge, secciones expandibles (LLM details, Embedding details, input, output, metrics) |
| `trace-constants` | 275 | Constantes compartidas: NODE_ICONS (37), NODE_COLORS (37), GROUP_META (8), estimateCost(), getNodeGroup(), EmbeddingDetail interface |
| `TraceNodeComponent` | 512 | Legacy — nodo React Flow custom (dead code, kept for reference) |
| `GroupNodeComponent` | 57 | Legacy — contenedor visual de sub-flow group (dead code, kept for reference) |

### Identity Governance (5)

| Componente | Líneas | Función |
|-----------|--------|---------|
| `VersionsTab` | 508 | CRUD versiones + activate/rollback |
| `EvolutionTab` | 273 | Approve/reject candidates |
| `ShadowTab` | 267 | Risk grades, diffs |
| `HealthTab` | 360 | Health signals timeline |
| `CreateSnapshotModal` | 283 | Modal: version, tags, label, notes |

### UI Primitives (10)

Badge, Button, Card, ConfirmDialog, Input, Progress, ScrollArea, Separator, Tabs, Tooltip — todos shadcn/ui o custom con CVA.

---

## Estado Global — Zustand Store (159 líneas)

| Grupo | Estado | Persistido |
|-------|--------|------------|
| Health | `health: HealthResponse` | No |
| System | `systemStatus: SystemStatus` | No |
| Agent | `agentState` (7 estados) | No |
| Chat | `messages[]`, `conversationId`, `isStreaming`, `cognitiveMode` | **Sí** (localStorage `iame-chat`) |
| Events | `events[]` (max 100) | No |
| Persona | `persona: PersonaInfo` | No |
| Router | `routerStatus` | No |
| WebSocket | `wsConnected` | No |

---

## Hooks

| Hook | Función |
|------|---------|
| `useHealth(5000)` | Polling: `api.health()` + `api.systemStatus()` cada 5s |
| `useInitialData()` | One-shot: carga events, persona, routerStatus en paralelo |
| `useWebSocket()` | WS `ws://localhost:8000/api/ws` — auto-reconnect 3s, keepalive 30s |

---

## i18n

631 keys en 17 secciones. EN + ES fully symmetric.
- Provider: `I18nProvider` en `client-shell.tsx`
- Hook: `useTranslation()` → `{ locale, setLocale, t(key) }`
- Persiste locale en localStorage

---

## Tema Visual

| Zona | Color |
|------|-------|
| Background | `#0a0e1a` |
| Surface | `#111827` |
| Card | `#1a2035` |
| Border | `#1e293b` |
| Text | `#e2e8f0` |
| Text dim | `#94a3b8` |
| Accent primary | `#6366f1` |
| Accent glow | `#818cf8` |
| Status green | `#22c55e` |
| Status amber | `#f59e0b` |
| Status red | `#ef4444` |

Fonts: Inter (sans) + JetBrains Mono (mono). Animaciones: `pulse-led`, `slide-in`, `fade-up`.

---

## API Client — `lib/api.ts` (1,279 líneas)

~111 métodos que mapean 1:1 con los [endpoints del backend](../api/README.md). Patrón:

```typescript
export const api = {
  chat: (msg, convId, mode) =>
    fetchAPI<ChatResponse>("/chat", {
      method: "POST",
      body: JSON.stringify({ message: msg, conversation_id: convId, cognitive_mode: mode }),
    }),
};
```

---

## Temas Relacionados

- [API](../api/README.md) — Los 120 endpoints que consume
- [Arquitectura](../architecture/README.md) — Backend que alimenta el dashboard
- [Configuración](../config/README.md) — tailwind.config, next.config
- [Eventos](../architecture/events.md) — WebSocket updates
