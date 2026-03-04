# Admin: Revenue Dashboard — `/admin/revenue`

## Información General

Página de administración exclusiva para el **owner** de la plataforma. Muestra métricas de revenue generado por el consumo de tokens de todos los tenants, desglosado por proveedor LLM y por tier.

---

## Información Presentada

### KPI Cards (4 tarjetas)

| Card | Valor | Descripción |
|------|-------|-------------|
| **Total Revenue** | USD formateado | Ingresos totales de la plataforma |
| **Total Commission** | USD formateado | Total de comisiones cobradas a tenants |
| **Total Consumed** | Tokens formateados | Total de tokens consumidos por todos los tenants |
| **Total Purchased** | Tokens formateados | Total de tokens comprados por todos los tenants |

### Revenue by Provider

Tabla con métricas por proveedor LLM:

| Columna | Descripción |
|---------|-------------|
| **Provider** | Nombre del proveedor (gemini, groq, claude, ollama) |
| **Consumed** | Tokens totales consumidos en ese proveedor |
| **Cost** | Costo total en USD |
| **Commission** | Comisiones totales generadas |

### Revenue by Tier

Tabla con métricas por nivel de tenant:

| Columna | Descripción |
|---------|-------------|
| **Tier** | free, basic, pro, enterprise |
| **Tenants** | Cantidad de tenants en ese tier |
| **Consumed** | Tokens totales consumidos por tenants de ese tier |
| **Commission** | Comisiones totales generadas por ese tier |

---

## Endpoints Usados

| Método | Endpoint | Propósito |
|--------|----------|-----------|
| GET | `/billing/revenue` | Revenue completo (owner only) |

---

## Control de Acceso

- Solo accesible cuando `authStore.isOwner() === true`
- Redirige a `/` si el usuario no es owner
