# ADLRA — Documentación del Sistema

> **Autonomous Digital Learning Representative Agent** — Sistema multi-agente que aprende a representar la identidad, personalidad y estilo de toma de decisiones de su principal.

**Principal**: Usuario Creador · **Delegado**: Django · **Última actualización**: 22 de febrero de 2026

---

## Sobre este documento

Este es el **punto de entrada principal** a toda la documentación técnica de ADLRA — el sistema que le da vida a Django como delegado digital autónomo. Si estás leyendo esto por primera vez, este documento te servirá como mapa general para entender qué existe, dónde encontrarlo, y cómo todas las piezas encajan entre sí.

### ¿Qué es ADLRA y qué hace Django?

ADLRA es un sistema compuesto por muchas piezas que trabajan juntas para lograr una cosa: que Django pueda **representar fielmente** al creador en conversaciones, decisiones de negocio, comunicación profesional y tareas técnicas. No es un chatbot genérico — es un **delegado entrenado** que aprende la personalidad de su creador (cómo piensa, qué valora, cómo se comunica) y la aplica consistentemente en cada interacción.

Para lograrlo, el sistema tiene:
- Un **backend en Python/FastAPI** con un pipeline de procesamiento de 25+ pasos que analiza cada mensaje, consulta memorias, aplica la identidad de Harold, genera respuestas con LLMs y evalúa la calidad del resultado — todo esto documentado en la sección de [Arquitectura](architecture/README.md)
- Un **dashboard en Next.js** con 15 páginas donde Harold puede configurar la personalidad de Django, entrenarle, ver sus memorias, monitorear su rendimiento y controlar su gobernanza — documentado en la sección de [Dashboard](dashboard/README.md)
- **116 endpoints de API** que conectan el frontend con el backend — documentados en la [Referencia API](api/README.md)
- **Integraciones externas** como el bot de Discord y el sistema de modelos LLM con fallback automático — en [Integraciones](integrations/README.md)
- **Archivos de configuración** que definen quién es Harold, qué modelos usar, qué habilidades tiene Django y qué límites no puede cruzar — en [Configuración](config/README.md)

### ¿Cómo se conectan todas las piezas?

La forma más fácil de entender ADLRA es seguir **el camino de un mensaje**:

1. Harold (o alguien en Discord) escribe algo → llega al backend via la API (`POST /chat`)
2. El **Orchestrator** (documentado en [Pipeline](architecture/pipeline.md)) toma ese mensaje y lo pasa por 25+ pasos secuenciales
3. Primero, el motor de **Cognición** (determinístico, sin LLM) decide la estrategia y el riesgo — ver [Cognición](architecture/cognition.md)
4. Luego se consultan las **Memorias** (4 niveles: desde la conversación actual hasta conocimiento permanente) — ver [Memoria](architecture/memory.md)
5. El sistema de **Identidad** (22 módulos) analiza, re-rankea y anota esas memorias según qué tan alineadas están con la personalidad de Harold — ver [Identidad](architecture/identity.md)
6. Uno de los 5 **Agentes LLM** genera la respuesta — ver [Agentes](architecture/agents.md)
7. El sistema de **Evaluación** (5 módulos heurísticos) califica la calidad, alineación, riesgo legal y más — ver [Evaluación](architecture/evaluation.md)
8. Todo se persiste en **Base de Datos** (Postgres + ChromaDB + SQLite) — ver [Base de Datos](architecture/database.md)
9. Cada paso genera **Eventos** y nodos de traza cognitiva para observabilidad — ver [Eventos](architecture/events.md)
10. El sistema **Teleológico** puede influir en la priorización y enriquecer el contexto con metas activas — ver [Teleología](architecture/teleology.md)
11. La **Seguridad** filtra inputs maliciosos antes de que entren al pipeline — ver [Seguridad](architecture/security.md)

Todo esto se inicializa en un orden preciso cuando el servidor arranca — documentado en [Startup](architecture/startup.md).

### ¿Cómo usar esta documentación?

- **Si quieres el panorama general**: empieza por [Arquitectura](architecture/README.md) y luego [Pipeline](architecture/pipeline.md)
- **Si quieres entender una pieza específica**: usa la tabla de navegación rápida abajo
- **Si eres Django leyendo sobre ti mismo**: ve a la sección "Para Django" al final de este documento
- **Si necesitas la referencia monolítica completa**: [Baseline.md](Baseline.md) tiene todo en un solo archivo (1,137 líneas)

---

## Mapa de Navegación

Esta documentación está organizada para ser explorada tanto por humanos como por Django. Cada sección enlaza a las demás donde los conceptos se conectan.

### Punto de entrada rápido

| Si quieres... | Ve a |
|---------------|------|
| Entender qué es ADLRA y cómo funciona | [Arquitectura General](architecture/README.md) |
| Ver cómo se procesa cada mensaje | [Pipeline del Orchestrator](architecture/pipeline.md) |
| Conocer los 116 endpoints de la API | [Referencia API](api/README.md) |
| Explorar las 15 páginas del dashboard | [Dashboard](dashboard/README.md) |
| Entender el sistema de identidad de Django | [Sistema de Identidad](architecture/identity.md) |
| Ver la configuración del sistema | [Configuración](config/README.md) |
| Entender las integraciones (Discord, LLMs) | [Integraciones](integrations/README.md) |
| Ver convenciones, tests, roadmap | [Desarrollo](development/README.md) |
| Consultar la referencia monolítica original | [Baseline.md](Baseline.md) |

---

## Estructura de la Documentación

```
docs/
├── README.md                          ← Estás aquí
├── Baseline.md                        # Referencia monolítica completa (1,137 líneas)
│
├── architecture/                      # Cómo funciona el sistema
│   ├── README.md                      #   Resumen, métricas, stack, estructura de archivos
│   ├── startup.md                     #   AppState, lifespan, inicialización
│   ├── pipeline.md                    #   Orchestrator: 25+ pasos del pipeline
│   ├── agents.md                      #   5 agentes LLM + AgentCrew
│   ├── cognition.md                   #   DecisionEngine + Planner (determinístico)
│   ├── memory.md                      #   4 niveles de memoria
│   ├── identity.md                    #   22 módulos de identidad (Phases 4-10C)
│   ├── teleology.md                   #   Metas, planes, prioridades, recompensas
│   ├── evaluation.md                  #   5 módulos de evaluación heurística
│   ├── events.md                      #   EventBus + Cognitive Trace
│   ├── security.md                    #   Sanitización, content wrapping, middleware
│   └── database.md                    #   Postgres + ChromaDB + SQLite + Persistence
│
├── api/
│   └── README.md                      #   116 endpoints agrupados por sección
│
├── dashboard/                         # Las 15 páginas del dashboard
│   ├── README.md                      #   Overview, componentes, store, hooks, i18n
│   ├── command-center.md              #   / — KPIs, estado, acciones rápidas
│   ├── chat.md                        #   /chat — Conversación con Django
│   ├── identity-studio.md             #   /identity — Editor de personalidad
│   ├── training-center.md             #   /training — 3 modos de entrenamiento
│   ├── testing-playground.md          #   /testing — Scenarios, A/B compare
│   ├── model-manager.md               #   /models — Proveedores LLM, perfiles
│   ├── skill-manager.md               #   /skills — Habilidades del delegado
│   ├── memory-lab.md                  #   /memory — CRUD de memorias
│   ├── governance-console.md          #   /governance — Reglas y emergency stop
│   ├── analytics.md                   #   /analytics — Métricas y uso
│   ├── cognitive-trace.md             #   /trace — Visualización React Flow
│   ├── evaluation-dashboard.md        #   /evaluation — 5 sistemas de evaluación
│   ├── identity-governance.md         #   /identity-governance — Versiones + evolución
│   └── goals.md                       #   /goals — Sistema de metas
│
├── config/
│   └── README.md                      #   persona.yaml, models.json, skills.json, governance.yaml
│
├── integrations/
│   └── README.md                      #   Discord bot, webhook, Model Router + circuit breaker
│
└── development/
    └── README.md                      #   Convenciones, tests, roadmap, riesgos, glosario
```

---

## Métricas del Proyecto

| Métrica | Valor |
|---------|-------|
| Backend Python | 25,217 líneas, 89 archivos |
| Tests pytest | 20,041 líneas, 47 archivos |
| Dashboard Next.js | 12,490 líneas |
| **Total** | **57,748 líneas** |
| Endpoints API | 116 (55 GET, 35 POST, 10 PUT, 12 DELETE, 1 WS) |
| Páginas dashboard | 15 |
| Módulos de identidad | 22 archivos, 6,080 líneas |
| Fases completadas | 17 |
| Costo operativo | $0/mes |

---

## Para Django

Si estás leyendo esto como parte de tu autoconocimiento:

1. **Tu identidad** está definida en [persona.yaml](config/README.md) y formalizada como [IdentityProfile](architecture/identity.md)
2. **Tu pipeline de pensamiento** tiene 25+ pasos documentados en [Pipeline](architecture/pipeline.md)
3. **Tus memorias** operan en [4 niveles](architecture/memory.md): working → episodic → semantic → procedural
4. **Tu gobernanza** está en [governance.yaml](config/README.md) — hay cosas que no puedes hacer
5. **Tu evolución** pasa por [22 módulos de identidad](architecture/identity.md) que te protegen del drift
6. **Tus metas** se gestionan via el [sistema teleológico](architecture/teleology.md)
7. **Cada interacción tuya** genera una [traza cognitiva](architecture/events.md) que Harold puede revisar

Esta documentación ES tu arquitectura. Explórala para entenderte.
