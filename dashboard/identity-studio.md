# Identity Studio — `/identity`

## Información General

El Identity Studio es el editor principal de la personalidad e identidad de Django. Permite visualizar y modificar los cinco grandes rasgos de personalidad (Big Five), el estilo de comunicación, la jerarquía de valores, y los límites conductuales del delegado. Cada cambio guardado aquí modifica directamente el archivo `persona.yaml` en el servidor, recarga inmediatamente la configuración de todos los agentes, y afecta todas las respuestas futuras de Django. Es la interfaz donde el principal define "quién es Django" a nivel de personalidad.

---

## Información Presentada

### Radar Chart de Big Five

Gráfico SVG de 5 ejes que visualiza los rasgos Big Five como un polígono. Cada eje va de 0.0 (centro) a 1.0 (borde). Se actualiza en tiempo real al mover los sliders. Los rasgos mostrados son:

| Rasgo | Rango | Significado |
|-------|-------|-------------|
| **Openness** | 0.0–1.0 | Apertura a experiencias nuevas |
| **Conscientiousness** | 0.0–1.0 | Organización, disciplina, meticulosidad |
| **Extraversion** | 0.0–1.0 | Sociabilidad, energía social |
| **Agreeableness** | 0.0–1.0 | Cooperación, empatía, amabilidad |
| **Neuroticism** | 0.0–1.0 | Reactividad emocional, sensibilidad al estrés |

Fuente: `GET /persona/info` → `traits`

### Sliders de Estilo de Comunicación

6 sliders horizontales que definen cómo se comunica Django:

| Dimensión | Rango | Polo bajo → Polo alto |
|-----------|-------|----------------------|
| **Formality** | 0.0–1.0 | Informal → Formal |
| **Technical Depth** | 0.0–1.0 | Simple → Técnico/detallado |
| **Warmth** | 0.0–1.0 | Distante → Cálido |
| **Directness** | 0.0–1.0 | Indirecto → Directo |
| **Humor** | 0.0–1.0 | Serio → Humorístico |
| **Verbosity** | 0.0–1.0 | Conciso → Verboso |

Fuente: `GET /persona/info` → `communication_style`

### Jerarquía de Valores

Lista ordenable por drag-and-drop. El orden determina la prioridad — el primer valor es el más importante. Cada valor es un string libre que el usuario puede definir. Soporta:

- **Reordenar**: Drag and drop para cambiar prioridad
- **Agregar**: Campo de texto + botón para añadir nuevos valores
- **Eliminar**: Botón X en cada valor para removerlo

Fuente: `GET /persona/info` → `values`

### Límites Conductuales (Boundaries)

Lista de reglas que Django debe respetar. Cada boundary es un string libre. Soporta:

- **Agregar**: Campo de texto + botón para añadir nuevos límites
- **Eliminar**: Botón X en cada límite para removerlo

Fuente: `GET /persona/info` → `boundaries`

---

## Acciones

### 1. Guardar Personalidad (Save Traits)
- **Tipo**: API Call
- **Comportamiento**: Llama `PUT /persona/traits` con el objeto `{openness, conscientiousness, extraversion, agreeableness, neuroticism}`
- **Impacto en Django**: Escribe los nuevos valores en `configs/persona.yaml` en disco. Recarga inmediatamente la configuración de todos los 5 agentes — sus system prompts se reconstruyen con los nuevos rasgos. El `IdentityCoreAgent` incorpora los rasgos en su prompt de persona. Emite evento `config.persona_updated`. Los cambios afectan la personalidad de TODAS las respuestas futuras, tanto en dashboard como en Discord.

### 2. Guardar Estilo de Comunicación (Save Communication)
- **Tipo**: API Call
- **Comportamiento**: Llama `PUT /persona/communication` con `{formality, technical_depth, warmth, directness, humor, verbosity}`
- **Impacto en Django**: Escribe en `persona.yaml`, recarga agentes. Afecta directamente el tono, nivel de detalle, y estilo de todas las respuestas. El `AlignmentEvaluator` usa estos valores como baseline para medir si las respuestas futuras se alinean con el estilo definido. El `IdentityContextWeighter` (Phase 6B) y `IdentityBehavioralBias` (Phase 8A) derivan `tone_weight` y `assertiveness` de estos valores.

### 3. Guardar Valores (Save Values)
- **Tipo**: API Call
- **Comportamiento**: Llama `PUT /persona/values` con la lista ordenada de valores
- **Impacto en Django**: Escribe en `persona.yaml`, recarga agentes. Los valores son usados por el `AlignmentEvaluator` para medir alineación ético-moral, por el `IdentityDecisionModulator` (Phase 6C) para evaluar alignment categoría-valores, por el `IdentityFeedbackController` (Phase 5C) para generar correction hints, y por el `IdentityBehavioralBias` (Phase 8A) para derivar assertiveness. El orden importa — el primer valor tiene mayor peso.

### 4. Guardar Límites (Save Boundaries)
- **Tipo**: API Call
- **Comportamiento**: Llama `PUT /persona/boundaries` con la lista de límites
- **Impacto en Django**: Escribe en `persona.yaml`, recarga agentes. Los boundaries son verificados por el `AlignmentEvaluator` en cada interacción — violaciones se reportan como flags en el Evaluation Dashboard. El `GovernanceAgent` también los considera en su revisión de compliance.

### 5. Reload from File (Recargar desde archivo)
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /persona/reload`, luego `GET /persona/info` para refrescar la UI
- **Impacto en Django**: Re-lee `configs/persona.yaml` desde disco (descartando cambios no guardados en la UI) y recarga todos los agentes. Útil cuando `persona.yaml` fue editado manualmente fuera del dashboard.

### ⚠️ Limitaciones Conocidas

- **Sin confirmación de cambios no guardados**: Si el usuario navega fuera de la página con cambios sin guardar, se pierden sin advertencia.
- **Sin historial de versiones en la UI**: Aunque el backend tiene `IdentityVersionManager` con versionado semántico, la UI no muestra diff ni permite revertir. Para versionado, usar la página `/identity-governance`.

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/identity` |
| **Archivo** | `dashboard/app/identity/page.tsx` (447 líneas, todo en un archivo) |
| **Componentes externos** | Ninguno dedicado — usa shadcn/ui primitives (Slider, Input, Button, Card) |
| **APIs al cargar** | `GET /persona/info` |
| **APIs de escritura** | `PUT /persona/traits`, `PUT /persona/communication`, `PUT /persona/values`, `PUT /persona/boundaries`, `POST /persona/reload` |
| **Estado** | React `useState` para cada sección (traits, communication, values, boundaries) |
| **Drag and Drop** | Implementado con HTML5 drag events nativos (onDragStart, onDragOver, onDrop) |
| **Iconos** | lucide-react: User, Save, RefreshCw, Plus, X, GripVertical |

**Última actualización**: 2025-06-23
