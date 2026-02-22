# Seguridad — src/security/

> [← Eventos](events.md) · [Base de Datos →](database.md)

**Directorio**: `src/security/` (429 líneas, 4 archivos)

---

## Sobre este documento

Este documento describe la **capa de protección de inputs** de Django — los 3 componentes que filtran, sanitizan y etiquetan el contenido antes de que entre al pipeline de procesamiento. Esta es la primera línea de defensa contra inyecciones de prompt, contenido malicioso y manipulación del sistema.

### ¿Qué cubre este documento?

Documenta 3 componentes: el `InputSanitizer` (detecta 3 tipos de amenaza: prompt injection, boundary violation, information extraction), el `ContentWrapper` (envúbelve el contenido del usuario con marcadores de boundary para que el LLM distinga entre instrucciones del sistema y contenido del usuario), y el `SecurityMiddleware` (se registra en posición PRE_CLASSIFY del pipeline para ejecutarse antes de cualquier procesamiento).

### ¿Cuál es su función en la arquitectura?

La seguridad es el **escudo perimetral**. Su trabajo es interceptar mensajes potencialmente peligrosos antes de que lleguen al LLM. Sin esta capa, un usuario malintencionado podría inyectar instrucciones que hagan que Django ignore su persona, revele información confidencial, o actúe fuera de sus límites de gobernanza.

### ¿Cómo afecta al comportamiento de Django?

**Detalle crítico**: actualmente la seguridad opera en **modo detect-only** — reporta amenazas detectadas pero **NO bloquea** ningún mensaje. Esto significa que:
- Las amenazas se registran como eventos y se incluyen en la traza cognitiva para auditoría
- El `ContentWrapper` añade boundary markers que ayudan al LLM a separar instrucciones de contenido
- Pero ningún mensaje se rechaza automáticamente todavía
- El paso a modo de bloqueo activo es una decisión pendiente que requiere calibración de falsos positivos

### ¿Cómo interactúa con las demás piezas?

- **[Pipeline](pipeline.md)**: el `SecurityMiddleware` se registra en la posición `PRE_CLASSIFY` — es el **primer filtro** que toca cada mensaje, antes incluso de la clasificación de tarea
- **[Eventos](events.md)**: las amenazas detectadas emiten eventos de seguridad y generan nodos en la traza cognitiva
- **[Startup](startup.md)**: los componentes de seguridad se registran como middleware del Orchestrator durante la inicialización
- **[Gobernanza](../dashboard/governance-console.md)**: las detecciones de seguridad alimentan el audit log visible en la consola de gobernanza
- **[API](../api/README.md)**: el endpoint `POST /chat` aplica sanitización antes de enviar al Orchestrator
- **[Cognición](cognition.md)**: el mensaje sanitizado (con boundary markers) es lo que recibe el `DecisionEngine` para clasificar
- **[Tests](../development/README.md)**: 3 archivos de test dedicados (`test_security.py`, `test_input_sanitizer.py`, `test_content_wrapper.py`) con ~515 líneas

---

## Componentes

| Archivo | Función | Líneas |
|---------|---------|--------|
| `input_sanitizer.py` | Detecta amenazas en input del usuario | — |
| `content_wrapper.py` | Marca contenido externo con boundary markers | — |
| `middleware.py` | `SecurityMiddleware` en posición PRE_CLASSIFY del [pipeline](pipeline.md) | 58 |

---

## Input Sanitizer

Detecta 3 categorías de amenaza:

| Amenaza | Descripción | Acción |
|---------|-------------|--------|
| **XSS** | Cross-site scripting patterns | Detect-only (Phase 11) |
| **SQL Injection** | Patrones de inyección SQL | Detect-only |
| **Prompt Injection** | Intentos de manipular el sistema prompt | Detect-only |

> **Modo actual**: Detect-only — reporta amenazas pero NO bloquea. Emite `security.threat_detected` via [EventBus](events.md).

---

## Content Wrapper

Marca contenido externo (web research, user uploads) con boundary markers de seguridad para que el LLM distinga entre contenido confiable y externo en el prompt.

---

## Security Middleware

Registrado en la posición `PRE_CLASSIFY` (prioridad 10) del [middleware registry](pipeline.md).

Se ejecuta ANTES de cualquier procesamiento del mensaje, como primer filtro en el pipeline.

---

## Integración

```
User message → SecurityMiddleware (PRE_CLASSIFY)
  ├── InputSanitizer.sanitize()
  │     ├── threat detected? → emit security.threat_detected event
  │     └── log threat details
  └── Continue to pipeline step 1 (Classify)
```

Visible en: [Command Center](../dashboard/command-center.md) ActivityFeed (eventos de seguridad), [Governance Console](../dashboard/governance-console.md) audit log.

---

## Temas Relacionados

- [Pipeline](pipeline.md) — Middleware position PRE_CLASSIFY
- [Eventos](events.md) — `security.threat_detected` events
- [Gobernanza](../dashboard/governance-console.md) — Audit log
- [Configuración](../config/README.md) — governance.yaml privacy rules
