# Multi-Tenant Architecture

> [← Seguridad](security.md) · [Base de Datos →](database.md)

---

## Sobre este documento

Este documento describe la **arquitectura multi-tenant** de ADLRA — el sistema que permite que múltiples inquilinos (tenants) utilicen la misma instancia del backend de forma aislada, con facturación independiente y control de acceso basado en API keys y JWT.

### ¿Qué cubre este documento?

Documenta el **módulo de tenancy** (`src/tenant/`), que incluye: middleware de autenticación, servicio de auth (JWT + API keys), billing de tokens, repositorio de datos de tenant, modelos de dominio, y contexto de tenant por request. También cubre las **rutas API** dedicadas (`routers/tenant.py` y `routers/auth.py`) y la **integración con el pipeline** existente.

### ¿Cuál es su función en la arquitectura?

El módulo multi-tenant es la **capa de aislamiento y facturación** del sistema. Permite que el dueño de la plataforma (principal) opere su instancia para sí mismo Y para clientes externos, cada uno con su propio saldo de tokens, historial de transacciones y claves de API, sin mezclar datos ni acceso.

### ¿Cómo interactúa con las demás piezas?

- **[Pipeline](pipeline.md)**: Cada interacción del orchestrator tiene un `tenant_id` que se propaga a todas las capas.
- **[Base de Datos](database.md)**: 5 tablas dedicadas a tenancy + columna `tenant_id` en las 16 tablas existentes.
- **[Persistence](database.md)**: Todos los métodos de `PersistenceRepository` aceptan `tenant_id` con auto-resolución desde el contexto.
- **[Model Router](../integrations/README.md)**: Cada llamada LLM se factura al tenant activo.
- **[Dashboard](../dashboard/README.md)**: 4 nuevas páginas (Billing, API Keys, Admin Tenants, Admin Revenue) + auth flow.

---

## Visión General

```
┌──────────────┐     ┌──────────────────────────────┐
│   Dashboard   │────▶│   TenantMiddleware            │
│   API Client  │     │   (JWT / ApiKey / Open)       │
└──────────────┘     └──────────┬───────────────────┘                 │
                                 ▼
                    ┌───────────────────────┐
                    │   TenantContext        │
                    │   (contextvars per     │
                    │    request)            │
                    └──────────┬────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │  Orchestrator │ │  Persistence │ │  Billing     │
     │  (tenant_id)  │ │  (auto-      │ │  (prepaid    │
     │               │ │   resolve)   │ │   tokens)    │
     └──────────────┘ └──────────────┘ └──────────────┘
```

---

## Módulo `src/tenant/` (5 archivos, ~1,200 líneas)

| Archivo | Líneas | Responsabilidad |
|---------|--------|-----------------|
| `middleware.py` | ~180 | `TenantMiddleware` — Starlette middleware que intercepta cada request HTTP |
| `auth.py` | ~200 | `AuthService` — Resolución JWT + API keys, generación de tokens |
| `billing.py` | ~220 | `TokenBillingService` — Facturación prepaid (compra, consumo, balance) |
| `repository.py` | ~350 | 18 métodos Postgres (CRUD tenants, wallets, transactions, API keys) |
| `models.py` | ~100 | Dataclasses: Tenant, TokenWallet, TokenTransaction, CommissionRate, ApiKey |
| `context.py` | ~50 | `TenantContext` dataclass + contextvars (set/get/reset per request) |

---

## Autenticación

### TenantMiddleware

Cada request HTTP pasa por `TenantMiddleware` que:

1. **Open paths**: Rutas en `OPEN_PATHS` (health, docs, ws, static) pasan sin autenticación.
2. **REQUIRE_AUTH=false** (default): Todas las requests se asignan automáticamente al tenant "principal" (owner). No se necesita token ni API key.
3. **REQUIRE_AUTH=true**: Se busca `Authorization` header:
   - `Bearer <jwt>` → Se valida JWT HS256 con `JWT_SECRET`, se extrae `tenant_id`.
   - `ApiKey <key>` → Se busca en DB, se verifica que sea activa y no expirada.
   - Sin header → 401.
4. Se crea `TenantContext(tenant_id, is_owner, tier)` y se almacena en `contextvars`.

### AuthService

```python
class AuthService:
    generate_jwt(tenant_id, is_owner, tier, ttl_hours=24) → str
    validate_jwt(token) → dict  # {tenant_id, is_owner, tier, exp}
    resolve_api_key(raw_key) → TenantContext | None
    hash_api_key(raw_key) → str  # SHA-256
    generate_raw_key() → str     # "doe_" + 48 random chars
```

### API Keys

Cada tenant puede crear múltiples API keys para acceso programático:

```
POST /api/auth/api-keys         → Crear nueva key (devuelve raw_key UNA vez)
GET  /api/auth/api-keys         → Listar keys del tenant (sin raw_key)
DELETE /api/auth/api-keys/{id}  → Revocar (soft delete: is_active=false)
DELETE /api/auth/api-keys/{id}/permanent → Eliminar permanentemente
GET  /api/auth/status           → Estado de la sesión actual
```

---

## Facturación de Tokens

### Modelo Prepaid

Los tenants operan con balance prepaid. Cada llamada LLM consume tokens del saldo.

```
Flujo: Comprar tokens → Usar LLM → Deducir tokens + comisión → Registrar transacción
```

### Tarifas por Proveedor

| Provider | Rate per 1M tokens | Modelo |
|----------|-------------------|--------|
| Gemini | $0.15 | gemini-2.5-flash |
| Groq | $0.05 | llama-3.3-70b |
| Claude | $3.00 | claude-sonnet-4 |
| Ollama | $0.00 | qwen3-embedding-8B |

### Comisiones por Tier

| Tier | Comisión | Descripción |
|------|----------|-------------|
| free | 30% | Tier gratuito con margen alto |
| basic | 20% | Uso low-volume |
| pro | 15% | Uso profesional |
| enterprise | 0% | Sin margen — precio de costo |

### TokenBillingService

```python
class TokenBillingService:
    check_balance(tenant_id) → {can_proceed, balance, minimum_required}
    record_consumption(tenant_id, tokens, provider, model, interaction_id) → Transaction
    purchase_tokens(tenant_id, amount) → Transaction
    get_wallet(tenant_id) → TokenWallet
    get_transactions(tenant_id, limit) → list[TokenTransaction]
```

---

## Aislamiento de Datos

### Tablas con `tenant_id`

Las 16 tablas de persistencia existentes incluyen columna `tenant_id`:

| Tabla | Grupo |
|-------|-------|
| `conversations` | Core |
| `messages` | Core |
| `audit_log` | Core |
| `kpi_snapshots` | Core |
| `config_versions` | Core (excluido en serial counter) |
| `interactions` | Persistence |
| `trace_nodes` | Persistence |
| `evaluations` | Persistence |
| `token_usage` | Persistence |
| `memory_operations` | Persistence |
| `goals` | Teleology |
| `goal_progress` | Teleology |
| `goal_conflicts` | Teleology |
| `reward_signals` | Teleology |
| `identity_signals` | Identity |
| `identity_evolution` | Identity |

### Tablas Dedicadas a Tenancy (5)

| Tabla | Columnas clave |
|-------|---------------|
| `tenants` | id, name, email, tier, status, is_platform_owner, settings, created_at |
| `token_wallets` | tenant_id, balance, lifetime_purchased, lifetime_consumed, lifetime_commission |
| `token_transactions` | id, tenant_id, type, amount, balance_after, provider, model, cost_usd, commission_usd |
| `commission_rates` | tier, rate, provider (optional) |
| `api_keys` | id, tenant_id, key_hash, key_prefix, name, scopes, is_active, expires_at |

### Auto-resolución de Tenant

Todos los métodos de `PersistenceRepository` tienen `tenant_id: Optional[str] = None`. Si no se proporciona, se resuelve automáticamente desde `get_tenant_context()`:

```python
def save_interaction(self, ..., tenant_id: Optional[str] = None):
    if tenant_id is None:
        ctx = get_tenant_context()
        tenant_id = ctx.tenant_id if ctx else "principal"
    ...
```

---

## Endpoints API

### Tenant Management (14 endpoints en `routers/tenant.py`)

| Método | Ruta | Descripción | Acceso |
|--------|------|-------------|--------|
| GET | `/tenants/me` | Mi tenant + wallet | Cualquier tenant |
| GET | `/tenants` | Listar todos | Owner only |
| GET | `/tenants/{id}` | Detalle de un tenant | Owner only |
| POST | `/tenants` | Crear tenant | Owner only |
| PUT | `/tenants/{id}/tier` | Cambiar tier | Owner only |
| PUT | `/tenants/{id}/status` | Cambiar status | Owner only |
| GET | `/billing/wallet` | Mi wallet | Cualquier tenant |
| GET | `/billing/wallet/{id}` | Wallet de otro tenant | Owner only |
| POST | `/billing/purchase` | Comprar tokens | Cualquier tenant |
| POST | `/billing/purchase/{id}` | Comprar para otro | Owner only |
| GET | `/billing/balance` | Check balance | Cualquier tenant |
| GET | `/billing/transactions` | Mis transacciones | Cualquier tenant |
| GET | `/billing/transactions/{id}` | Transacciones de otro | Owner only |
| GET | `/billing/rates` | Tarifas y comisiones | Cualquier tenant |
| GET | `/billing/revenue` | Revenue total | Owner only |

### Auth (5 endpoints en `routers/auth.py`)

| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/auth/api-keys` | Crear API key |
| GET | `/auth/api-keys` | Listar mis API keys |
| DELETE | `/auth/api-keys/{id}` | Revocar API key |
| DELETE | `/auth/api-keys/{id}/permanent` | Eliminar API key |
| GET | `/auth/status` | Estado de sesión |

---

## Dashboard Integration

4 nuevas páginas + auth flow integrado:

| Ruta | Página | Descripción |
|------|--------|-------------|
| `/login` | Login | Auto-login (open) o API key auth (require_auth) |
| `/billing` | Billing & Wallet | Balance, compra, historial de transacciones, tarifas |
| `/api-keys` | API Keys | CRUD de claves de acceso programático |
| `/admin/tenants` | Tenant Management | Listar, crear, cambiar tier/status (owner only) |
| `/admin/revenue` | Revenue Dashboard | Revenue por provider y por tier (owner only) |

### Auth Flow

```
Login Page → api.getMe() 
  ├── REQUIRE_AUTH=false → auto-login as principal (no token needed)
  └── REQUIRE_AUTH=true → show API key form → validate → login
      └── JWT stored in localStorage → injected in all API calls
          └── Auto-logout on 401
```

### Role-Based UI

- **Sidebar**: Admin section (Tenants, Revenue) only visible when `isOwner()`
- **Header**: User menu dropdown with name, tier badge, wallet balance, logout
- **AuthGuard**: Wraps all pages except `/login`, redirects unauthenticated users

---

## Configuración

Variables de entorno relevantes:

| Variable | Default | Descripción |
|----------|---------|-------------|
| `REQUIRE_AUTH` | `false` | Si es true, requiere JWT/ApiKey en cada request |
| `JWT_SECRET` | random | Secreto HS256 para tokens JWT |
| `JWT_TTL_HOURS` | `24` | Duración del token JWT |
| `PLATFORM_OWNER_ID` | `"principal"` | ID del tenant propietario |

---

## Tests

- 57 tests de aislamiento de datos en `tests/test_tenant_isolation.py`
- Tests unitarios para middleware, auth, billing en archivos dedicados
- Total: 2,602 tests pasando
