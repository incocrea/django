# Login — `/login`

## Información General

La página de Login es el punto de entrada al sistema. Soporta dos modos de autenticación dependiendo de la configuración del backend: **Open Mode** (cuando `REQUIRE_AUTH=false`, valor por defecto) y **Auth Mode** (cuando `REQUIRE_AUTH=true`).

En Open Mode, el login es automático — la página detecta que no se requiere autenticación y ejecuta `api.getMe()` para autenticar como el tenant principal (owner). En Auth Mode, presenta un formulario para ingresar una API key (`doe_xxxx...`).

---

## Información Presentada

### Open Mode (REQUIRE_AUTH=false)

Al cargar la página, se detecta automáticamente que la autenticación no es requerida:

| Elemento | Descripción |
|----------|-------------|
| **Título** | "Welcome to Doe" |
| **Subtítulo** | "Autonomous Digital Delegate System" |
| **Auto-login** | Llama a `api.getMe()` automáticamente |
| **Resultado** | Redirige a `/` tras login exitoso |

### Auth Mode (REQUIRE_AUTH=true)

| Elemento | Descripción |
|----------|-------------|
| **Título** | "Welcome to Doe" |
| **API Key input** | Campo de texto para ingresar la API key |
| **Login button** | Valida la key contra el backend |
| **Error message** | Muestra error si la key es inválida o expirada |

---

## Acciones del Usuario

| Acción | Disparador | Resultado |
|--------|-----------|-----------|
| **Auto-login** (open mode) | Carga de página | `api.getMe()` → store auth → redirect `/` |
| **Submit API key** (auth mode) | Click "Login" o Enter | Valida key → JWT → store auth → redirect `/` |

---

## Flujo de Auth

```
Página Login
  ├── REQUIRE_AUTH=false
  │   └── api.getMe() → {tenant_id, name, tier, is_owner}
  │       └── authStore.login(token, user) → redirect "/"
  │
  └── REQUIRE_AUTH=true
      └── User enters API key
          └── api.login(apiKey) → {jwt, tenant}
              └── authStore.login(jwt, user) → redirect "/"
```

---

## Estado Global

El auth store (`lib/auth-store.ts`) persiste en `localStorage` key `"doe-auth"`:

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `token` | `string \| null` | JWT para Authorization header |
| `user` | `AuthUser \| null` | `{tenant_id, name, email, tier, is_owner}` |
| `wallet` | `WalletInfo \| null` | `{balance, lifetime_purchased, lifetime_consumed}` |

Computed:
- `isAuthenticated()` → `token !== null`
- `isOwner()` → `user?.is_owner === true`

---

## Endpoints Usados

| Método | Endpoint | Propósito |
|--------|----------|-----------|
| GET | `/tenants/me` | Obtener info del tenant actual |
| GET | `/auth/status` | Verificar sesión |

---

## Notas Técnicas

- La página NO está protegida por `AuthGuard` (es pública)
- `ClientShell` detecta `/login` como `PUBLIC_PATH` y renderiza sin sidebar/header
- El token JWT se inyecta automáticamente en todas las llamadas API via `fetchAPI()`
- Un 401 en cualquier endpoint dispara auto-logout y redirect a `/login`
