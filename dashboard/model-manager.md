# Model Manager — `/models`

## Información General

El Model Manager controla qué modelos de lenguaje (LLM) usa Django, cómo están asignados a cada agente, y qué perfil de configuración está activo. Permite ver el estado de los providers (Gemini, Groq, Ollama), probar la conexión a cada uno, cambiar de perfil con un clic, y reasignar qué modelo usa cada agente individualmente. Los cambios aquí afectan inmediatamente qué proveedor y modelo procesa las solicitudes de cada agente — modificando calidad, velocidad, costo, y privacidad de las respuestas.

---

## Información Presentada

### Estado de Providers

Tarjetas para cada provider configurado (Gemini, Groq, Ollama) que muestran:

| Dato | Descripción |
|------|-------------|
| **Nombre** | gemini, groq, ollama |
| **Estado** | LED verde (available) o gris (unavailable) basado en `GET /router/status` |
| **Modelo activo** | Nombre del modelo configurado (ej: gemini-2.5-flash, llama-3.3-70b-versatile) |
| **Circuit breaker** | Estado del circuit breaker: failures count, estado (closed/open/half-open) |
| **Fallback position** | Posición en la cadena de fallback (1°, 2°, 3°) |

Fuente: `GET /router/status`

### Cadena de Fallback

Visualización de la cadena de fallback configurada:
```
Gemini (1°) → Groq (2°) → Ollama (3°)
```
Cuando un provider falla, el router automáticamente intenta el siguiente en la cadena.

### Perfiles de Configuración

4 perfiles predefinidos con sus configuraciones:

| Perfil | Asignación | Caso de uso |
|--------|-----------|-------------|
| **balanced** | Gemini para identity_core/communication/governance, Groq para business/technical | Balance costo-calidad (default) |
| **max_quality** | Todo Gemini | Máxima calidad, mayor costo |
| **privacy_mode** | Todo Ollama | 100% local, sin datos a la nube, menor calidad |
| **budget_mode** | Todo Ollama llama3.1:8b | Mínimo costo ($0), modelo más pequeño |

El perfil activo se muestra con un indicador visual.

Fuente: `GET /models/config`

### Asignaciones de Agentes

Tabla que muestra qué modelo/provider tiene asignado cada uno de los 5 agentes:

| Agente | Provider por defecto |
|--------|---------------------|
| Identity Core | Gemini |
| Business | Groq |
| Communication | Gemini |
| Technical | Groq |
| Governance | Gemini |

### Routing por Tipo de Tarea

Tabla decorativa que muestra cómo se rutean los diferentes tipos de tarea a los agentes (basado en la clasificación del paso 1 del pipeline):

| Categoría | Agente asignado |
|-----------|----------------|
| CONVERSATION | Identity Core |
| BUSINESS | Business |
| TECHNICAL | Technical |
| CREATIVE | Communication |
| RESEARCH | Technical |
| ANALYSIS | Business |

---

## Acciones

### 1. Cambiar Perfil
- **Tipo**: API Call
- **Comportamiento**: Llama `PUT /models/profile` con `{profile: "balanced|max_quality|privacy_mode|budget_mode"}`
- **Impacto en Django**: Reconfigura **instantáneamente** las asignaciones de TODOS los agentes al preset del perfil seleccionado. Escribe en `configs/models.json`. La PRÓXIMA llamada LLM de cualquier agente usará el nuevo provider/modelo. Si se cambia a `privacy_mode`, todo el tráfico LLM pasa a ser local (Ollama) — ningún dato sale de la máquina. Si se cambia a `max_quality`, todo va a Gemini — mayor gasto de tokens pero mejor calidad.

### 2. Cambiar Asignación Individual
- **Tipo**: API Call
- **Comportamiento**: Seleccionar un provider diferente en el dropdown de un agente específico. Llama `PUT /models/assignment` con `{agent_role, provider}`
- **Impacto en Django**: Cambia el provider del agente especificado sin afectar los demás. Escribe en `configs/models.json`. Solo afecta al agente modificado — por ejemplo, poner Identity Core en Ollama y mantener el resto en Gemini para proteger la privacidad del core de identidad mientras se usa calidad cloud para lo demás.

### 3. Probar Modelo
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /models/test` con `{provider}`
- **Impacto en Django**: Envía un prompt de prueba simple al provider seleccionado y mide latencia. **No genera efectos colaterales** — no crea memorias, traces, ni evaluaciones. Es una llamada LLM directa sin pasar por el orquestador. Útil para verificar que un provider está online antes de cambiar la asignación.

### 4. Recargar desde Archivo
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /router/reload`
- **Impacto en Django**: Re-lee `configs/models.json` desde disco y reconfigura el router. Útil si el archivo fue editado manualmente. Emite evento `config.router_reloaded`.

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/models` |
| **Archivo** | `dashboard/app/models/page.tsx` (403 líneas) |
| **APIs al cargar** | `GET /models/config`, `GET /router/status` |
| **APIs de escritura** | `PUT /models/profile`, `PUT /models/assignment`, `POST /models/test`, `POST /router/reload` |
| **Config file** | `configs/models.json` (modificado por PUT endpoints) |
| **Estado** | React `useState` para config, status, perfil seleccionado |
| **Iconos** | lucide-react: Cpu, RefreshCw, Play, CheckCircle, XCircle, AlertCircle |

**Última actualización**: 2025-06-23
