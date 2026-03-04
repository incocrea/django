# API Keys — `/api-keys`

## Información General

La página de API Keys permite crear, ver y revocar claves de acceso programático al sistema. Cada API key está asociada al tenant autenticado y puede usarse como alternativa al JWT para autenticar requests HTTP.

Las keys tienen el formato `doe_xxxx...` (48 caracteres random con prefijo `doe_`). El valor completo de la key **solo se muestra una vez** al momento de la creación — después solo se muestra el prefijo (`doe_xxxx...xxxx`).

---

## Información Presentada

### Create New Key

| Elemento | Descripción |
|----------|-------------|
| **Name input** | Nombre descriptivo para identificar la key |
| **Create button** | Genera una nueva API key |

### Key Created Banner

Aparece solo tras crear una nueva key:

| Elemento | Descripción |
|----------|-------------|
| **Raw key display** | La key completa en monospace con fondo destacado |
| **Copy button** | Copia al clipboard |
| **Warning** | "This key will only be shown once. Copy it now." |

### Key List

Tabla con todas las keys del tenant:

| Columna | Descripción |
|---------|-------------|
| **Name** | Nombre descriptivo de la key |
| **Prefix** | Primeros caracteres (`doe_xxxx`) — copiable al clipboard |
| **Status** | Badge: `active` (verde), `revoked` (rojo) |
| **Scopes** | Scopes de acceso (si están definidos) |
| **Last Used** | Fecha del último uso |
| **Created** | Fecha de creación |
| **Actions** | Botones de Revoke (si activa) y Delete |

---

## Acciones del Usuario

| Acción | Disparador | Resultado |
|--------|-----------|-----------|
| **Create key** | Click "Create" con nombre | `api.createApiKey(name)` → muestra raw_key una vez |
| **Copy key** | Click icono clipboard | Copia raw_key o prefijo al clipboard |
| **Revoke key** | Click icono XCircle | `api.revokeApiKey(id)` → marca como revocada (soft delete) |
| **Delete key** | Click icono Trash2 | `api.deleteApiKey(id)` → elimina permanentemente |

---

## Endpoints Usados

| Método | Endpoint | Propósito |
|--------|----------|-----------|
| GET | `/auth/api-keys` | Listar todas las keys del tenant |
| POST | `/auth/api-keys` | Crear nueva key (devuelve `raw_key`) |
| DELETE | `/auth/api-keys/{id}` | Revocar key (soft delete) |
| DELETE | `/auth/api-keys/{id}/permanent` | Eliminar key permanentemente |

---

## Notas Técnicas

- La `raw_key` completa solo aparece en la respuesta de creación — nunca más
- El backend almacena un SHA-256 hash de la key (no el valor en texto plano)
- Las keys revocadas permanecen en la lista pero no funcionan para autenticación
- El prefijo (`key_prefix`) se almacena para identificación visual
