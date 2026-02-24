# Skill Manager — `/skills`

## Información General

El Skill Manager permite visualizar, activar y desactivar las habilidades (skills) disponibles de Django, y ejecutar herramientas de aprendizaje. Las habilidades son capacidades discretas que Django puede usar durante el procesamiento de mensajes — como búsqueda web, generación de documentos, o aprendizaje de temas. Habilitar o deshabilitar una skill aquí determina si el orquestador puede invocarla durante el pipeline de procesamiento.

---

## Información Presentada

### Grid de Habilidades

Tarjetas para cada skill registrada en `configs/skills.json`. Actualmente 5 skills:

| Skill | Descripción | Estado default |
|-------|-------------|----------------|
| **web-research** | Búsqueda web vía Tavily API (single search + deep research) | Habilitada |
| **email-draft** | Generación de borradores de email usando CommunicationAgent | Habilitada |
| **document-gen** | Generación de documentos usando CommunicationAgent | Habilitada |
| **learn-topic** | Aprendizaje de temas: web search → LLM summarize → chunk → ChromaDB | Habilitada |
| **repo-explorer** | Exploración de repositorios locales/GitHub + docs propios de Django | Habilitada |

Cada tarjeta muestra:
- Nombre y descripción de la skill
- Badge de estado (verde = habilitada, gris = deshabilitada)
- Botón de toggle
- Estadísticas de uso (número de veces invocada)

Fuente: `GET /skills`

### Buscador de Skills

Campo de texto que filtra las skills visibles por nombre o descripción. Filtrado puramente client-side.

### Herramienta Learn Topic

Panel dedicado (visible cuando learn-topic está habilitada) con:
- Campo de texto para el tema a aprender
- Selector de profundidad con 3 niveles:
  - **1 (basic)**: Búsqueda superficial, resumen corto
  - **2 (moderate)**: Múltiples fuentes, resumen detallado
  - **3 (deep)**: Investigación profunda con sub-queries, resumen extenso
- Resultado del aprendizaje una vez completado

---

## Acciones

### 1. Toggle Skill (Activar/Desactivar)
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /skills/{skill_id}/toggle`
- **Impacto en Django**: Cambia el estado enabled/disabled de la skill en el `SkillRegistry` en memoria. Persiste el cambio en `configs/skills.json`. Cuando una skill está deshabilitada:
  - **web-research**: El orquestador no podrá hacer búsquedas web incluso en modo cognitivo 1 (Full). Las búsquedas simplemente se omiten.
  - **learn-topic**: Los comandos de chat "aprende sobre X" no se detectarán en el paso 0.5 del pipeline. El endpoint API seguirá disponible pero retornará error.
  - **repo-explorer**: Los comandos "lee el archivo X" o "explora el repo Y" no se detectarán en el paso 0.7 del pipeline.
  - **email-draft / document-gen**: Las herramientas de generación de documentos/emails no estarán disponibles para el orquestador.

### 2. Learn Topic (Aprender Tema)
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /skills/learn-topic` con `{topic, depth: 1|2|3}`
- **Impacto en Django**: Ejecuta un pipeline de aprendizaje autónomo:
  1. **Búsqueda web**: Usa Tavily API para buscar información sobre el tema (depth controla cuántas sub-queries)
  2. **Resumen LLM**: Envía los resultados de la búsqueda al LLM para generar un resumen estructurado
  3. **Chunking**: Divide el resumen en segmentos de ~500 palabras
  4. **Almacenamiento**: Guarda los chunks en ChromaDB como memoria semántica con `category=learned_knowledge`
  
  Después de este proceso, Django "sabe" sobre el tema — la información estará disponible en el recall de memoria (paso 4 del pipeline) para todas las conversaciones futuras. En modos cognitivos 2 y 3, solo las memorias con categoría `learned_knowledge` pasan el filtro semántico, así que este mecanismo es la forma principal de expandir el conocimiento de Django en esos modos.

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/skills` |
| **Archivo** | `dashboard/app/skills/page.tsx` (300 líneas) |
| **APIs al cargar** | `GET /skills`, `GET /skills/dynamic` |
| **APIs de escritura** | `POST /skills/{id}/toggle`, `POST /skills/learn-topic`, `POST /skills/dynamic/create`, `DELETE /skills/dynamic/{name}`, `POST /skills/dynamic/{name}/test`, `POST /skills/dynamic/{name}/reload` |
| **Config file** | `configs/skills.json` (5 skills registradas) |
| **Estado** | React `useState` para lista de skills, filtro de búsqueda, resultado learn-topic |
| **Backend modules** | `src/skills/registry.py`, `src/skills/learn_topic.py` (428 ln), `src/skills/web_research.py`, `src/skills/tools.py`, `src/skills/repo_explorer.py` (1,107 ln), `src/skills/skill_generator.py` (555 ln), `src/skills/dynamic_loader.py` (400 ln) |
| **Iconos** | lucide-react: Wrench, Power, Search, BookOpen, Loader2 |

---

## SkillForge — Dynamic Skill Creation

### Arquitectura

SkillForge permite que Django cree dinámicamente nuevos skills a partir de descripciones en lenguaje natural.

| Componente | Archivo | Función |
|-----------|---------|---------|
| `SkillGenerator` | `skill_generator.py` (555 ln) | Pipeline LLM-driven: parse requirement → Claude prompt → generate code → AST validate → install → smoke test |
| `DynamicSkillLoader` | `dynamic_loader.py` (400 ln) | Lifecycle manager: install, uninstall, load_all, reload |
| `ASTValidator` | `ast_validator.py` (353 ln) | Import allowlist/blocklist + structural validation |
| `SkillFileManager` | `fs_manager.py` (259 ln) | Sandboxed file I/O in `src/skills/dynamic/` |
| `BaseSkill` | `base_skill.py` (176 ln) | ABC that all dynamic skills must extend |
| `SkillAuthGate` | `skill_auth.py` (217 ln) | Access level enforcement (PUBLIC/PRINCIPAL/SYSTEM) |

### API Endpoints (5)

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/skills/dynamic` | Lista dynamic skills cargados |
| `POST` | `/skills/dynamic/create` | Crear skill desde descripción en lenguaje natural (Claude) |
| `DELETE` | `/skills/dynamic/{name}` | Desinstalar dynamic skill |
| `POST` | `/skills/dynamic/{name}/test` | Smoke test de dynamic skill |
| `POST` | `/skills/dynamic/{name}/reload` | Recargar dynamic skill desde disco |

### Pipeline de Creación

1. **Parse Requirement** — Extrae nombre, descripción, tags del lenguaje natural
2. **Build Prompt** — Comprehensive BaseSkill template con reglas de imports, código template, reglas críticas
3. **Generate Code** — Claude genera Python code
4. **AST Validate** — Verifica imports, estructura, forbidden calls
5. **Retry** — Hasta 2 reintentos con feedback de errores (temperature 0.2)
6. **Install** — DynamicSkillLoader: write → import → instantiate → register centroid
7. **Smoke Test** — Advisory: validate() + execute() con mensaje test

**Última actualización**: 2025-06-23
