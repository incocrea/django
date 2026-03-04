# Admin: Tenant Management — `/admin/tenants`

## Información General

Página de administración exclusiva para el **owner** de la plataforma. Permite listar, crear y gestionar los tenants del sistema. Los tenants no-owner que intenten acceder son redirigidos automáticamente a `/`.

Cada tenant tiene un tier (free, basic, pro, enterprise) que determina su comisión por token, y un status (active, suspended, deleted) que controla su acceso.

---

## Información Presentada

### Stats Row

4 badges con conteo de tenants por tier:

| Badge | Descripción |
|-------|-------------|
| **Free** | Cantidad de tenants en tier free |
| **Basic** | Cantidad de tenants en tier basic |
| **Pro** | Cantidad de tenants en tier pro |
| **Enterprise** | Cantidad de tenants en tier enterprise |

### Create Tenant Form

| Campo | Tipo | Descripción |
|-------|------|-------------|
| **Name** | Text input | Nombre del tenant |
| **Email** | Email input | Email del tenant |
| **Tier** | Select | free / basic / pro / enterprise (default: free) |
| **Create** | Button | Crea tenant con UUID autogenerado |

### Tenant List

Tabla con todos los tenants:

| Columna | Descripción |
|---------|-------------|
| **Name** | Nombre del tenant |
| **Email** | Email del tenant |
| **Tier** | Dropdown editable — cambio en vivo del tier |
| **Status** | Dropdown editable — cambio en vivo del status |
| **Created** | Fecha de creación |

---

## Acciones del Usuario

| Acción | Disparador | Resultado |
|--------|-----------|-----------|
| **Create tenant** | Click "Create" con nombre y email | `api.createTenant({tenant_id, name, email, tier})` |
| **Change tier** | Select dropdown en fila | `api.updateTenantTier(id, newTier)` |
| **Change status** | Select dropdown en fila | `api.updateTenantStatus(id, newStatus)` |

---

## Endpoints Usados

| Método | Endpoint | Propósito |
|--------|----------|-----------|
| GET | `/tenants` | Listar todos los tenants (owner only) |
| POST | `/tenants` | Crear nuevo tenant |
| PUT | `/tenants/{id}/tier` | Cambiar tier |
| PUT | `/tenants/{id}/status` | Cambiar status |

---

## Control de Acceso

- Solo accesible cuando `authStore.isOwner() === true`
- Redirige a `/` si el usuario no es owner
- Items de navegación (sidebar) solo visibles para owner
