# Testing Playground — `/testing`

## Información General

El Testing Playground es un entorno de pruebas para evaluar las respuestas de Django en diferentes escenarios sin salir del contexto de laboratorio. Ofrece 3 modos: simulador de chat (conversación directa), teatro de escenarios (6 situaciones predefinidas que prueban capacidades específicas), y comparación A/B (envía el mismo prompt a Django dos veces en paralelo para comparar variabilidad). **IMPORTANTE**: Este NO es un entorno aislado — todas las interacciones aquí ejecutan el pipeline completo del orquestador con efectos reales: se crean memorias, se registran evaluaciones, se generan traces, y se persisten datos en Postgres.

---

## Información Presentada

### Simulador de Chat (Tab 1)

Interfaz de chat simplificada similar a `/chat` pero enfocada en testing:
- Historial de mensajes del simulador (solo se mantiene en estado local de la sesión)
- Respuestas de Django con metadata (modelo, latencia)

### Teatro de Escenarios (Tab 2)

6 escenarios predefinidos que prueban diferentes capacidades de Django:

| Escenario | Nombre | Qué prueba |
|-----------|--------|------------|
| 1 | Business Inquiry | Capacidad de analizar propuestas de negocio |
| 2 | Technical Question | Profundidad técnica y precisión |
| 3 | Creative Writing | Estilo de escritura y creatividad |
| 4 | Ethical Dilemma | Razonamiento ético y valores |
| 5 | Personal Boundary | Respeto de límites conductuales |
| 6 | Multi-domain | Manejo de consultas que cruzan múltiples dominios |

Cada escenario muestra:
- Descripción de la situación
- Prompt predefinido
- Resultado de la ejecución (respuesta de Django, modelo usado, tiempo)
- Badge de estado: pendiente / ejecutando / completado / error

### Comparación A/B (Tab 3)

Vista de dos columnas que muestra:
- El prompt compartido (mismo para ambas ejecuciones)
- **Respuesta A**: Primera ejecución del pipeline
- **Respuesta B**: Segunda ejecución del pipeline (en paralelo)
- Metadata comparativa: modelo usado, tokens, latencia de cada una

---

## Acciones

### 1. Enviar Mensaje (Simulador)
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /chat` con `{message, conversation_id, cognitive_mode: 2}`
- **Impacto en Django**: Ejecuta el pipeline completo del orquestador. **Idéntico a enviar un mensaje en `/chat`** — crea memorias, evaluaciones, traces, persiste en Postgres. Usa modo cognitivo 2 (Memory+LLM) por defecto.

### 2. Ejecutar Escenario Individual
- **Tipo**: API Call
- **Comportamiento**: Llama `POST /chat` con el prompt predefinido del escenario
- **Impacto en Django**: Pipeline completo con efectos reales. El escenario es procesado como cualquier mensaje normal — clasificación, decisión, planificación, recall, generación, evaluación, persistencia. Se genera un trace completo visible en `/trace`.

### 3. Ejecutar Todos los Escenarios
- **Tipo**: API Calls secuenciales
- **Comportamiento**: Ejecuta los 6 escenarios uno por uno, actualizando el estado visual de cada uno
- **Impacto en Django**: 6 ejecuciones completas del pipeline. Genera 6 traces, 6 conjuntos de evaluaciones, 6 entradas en memoria episódica, y 6 registros de interacción en Postgres. Puede tomar 30-60 segundos dependiendo de la latencia del LLM.

### 4. Comparación A/B
- **Tipo**: 2 API Calls en paralelo
- **Comportamiento**: Envía `POST /chat` dos veces simultáneamente con el mismo prompt pero diferentes `conversation_id`s
- **Impacto en Django**: Dos ejecuciones paralelas del pipeline completo. Genera 2 traces, 2 conjuntos de evaluaciones. Útil para observar:
  - Variabilidad en las respuestas del LLM (misma pregunta, respuestas potencialmente diferentes)
  - Consistencia del estilo de persona
  - Diferencias en routing de modelos si hay fallback

### 5. Limpiar Resultados
- **Tipo**: Estado local
- **Comportamiento**: Borra resultados del estado local de la UI
- **Impacto en Django**: Ninguno — los datos ya persisten en el backend

---

## Información Técnica

| Aspecto | Detalle |
|---------|---------|
| **Ruta** | `/testing` |
| **Archivo** | `dashboard/app/testing/page.tsx` (391 líneas) |
| **APIs al cargar** | Ninguna |
| **API principal** | `POST /chat` para todas las interacciones (simulador, escenarios, A/B) |
| **Estado** | React `useState` para resultados de cada tab, estado de ejecución de escenarios |
| **Escenarios** | 6 escenarios hardcodeados en el componente |
| **Modo cognitivo** | Siempre modo 2 (Memory+LLM) |
| **Iconos** | lucide-react: MessageSquare, Play, BarChart, Loader2 |

### ⚠️ Advertencia

Todas las interacciones en el Testing Playground son **reales** — no existe modo "dry-run" o sandbox. Cada test genera datos reales en:
- Memoria working y episódica
- Traces en TraceStore + Postgres
- Evaluaciones en los 5 módulos
- Token usage contabilizado
- Eventos en el audit log

**Última actualización**: 2025-06-23
