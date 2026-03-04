# Billing & Wallet — `/billing`

## Información General

La página de Billing muestra el estado financiero del tenant: balance de tokens, historial de transacciones, tarifas por proveedor y comisiones por tier. Permite comprar tokens adicionales para consumo de LLM.

El modelo de facturación es **prepaid** — los tenants compran tokens por adelantado y cada llamada LLM descuenta del balance según el proveedor utilizado más la comisión del tier.

---

## Información Presentada

### KPI Cards (4 tarjetas)

| Card | Valor | Descripción |
|------|-------|-------------|
| **Balance** | `wallet.balance` formateado (X.XXM / X.XXK) | Tokens disponibles actualmente |
| **Purchased** | `wallet.lifetime_purchased` | Total de tokens comprados desde el inicio |
| **Consumed** | `wallet.lifetime_consumed` | Total de tokens consumidos en llamadas LLM |
| **Commission** | `wallet.lifetime_commission` | Total pagado en comisiones de plataforma |

### Purchase Tokens

| Elemento | Descripción |
|----------|-------------|
| **Quick select buttons** | 100K, 500K, 1M, 5M — click para preseleccionar cantidad |
| **Input numérico** | Campo editable para cantidad personalizada |
| **Buy button** | Ejecuta la compra y recarga wallet |

### Provider Rates

Tabla con tarifas de cada proveedor LLM:

| Columna | Descripción |
|---------|-------------|
| **Provider** | Nombre del proveedor (gemini, groq, claude, ollama) |
| **Rate per 1M** | Costo en USD por millón de tokens |

### Commission by Tier

Tabla con porcentaje de comisión según nivel del tenant:

| Columna | Descripción |
|---------|-------------|
| **Tier** | free, basic, pro, enterprise |
| **Commission** | Porcentaje aplicado sobre cada consumo |

### Transaction History

Tabla con las últimas transacciones:

| Columna | Descripción |
|---------|-------------|
| **Type** | Badge con color: `purchase` (verde), `consumption` (rojo), `refund` (azul) |
| **Amount** | Tokens con signo (+ compra, - consumo) |
| **Balance** | Balance resultante después de la transacción |
| **Provider** | Proveedor LLM usado (solo en consumos) |
| **Cost** | Costo estimado en USD |
| **Description** | Descripción de la transacción |
| **Date** | Fecha formateada |

---

## Acciones del Usuario

| Acción | Disparador | Resultado |
|--------|-----------|-----------|
| **Quick select** | Click en 100K/500K/1M/5M | Establece cantidad en el input |
| **Purchase tokens** | Click "Buy Tokens" | `api.purchaseTokens(amount)` → recarga wallet y transacciones |

---

## Endpoints Usados

| Método | Endpoint | Propósito |
|--------|----------|-----------|
| GET | `/billing/wallet` | Obtener balance y totales del wallet |
| GET | `/billing/transactions` | Historial de transacciones |
| GET | `/billing/rates` | Tarifas por proveedor y comisiones por tier |
| POST | `/billing/purchase` | Comprar tokens |

---

## Notas Técnicas

- Al cargar, sincroniza el wallet con el auth store global (`authStore.setWallet()`)
- Las cantidades se formatean con sufijo: ≥1M → "X.XXM", ≥1K → "X.XXK"
- La tabla de transacciones muestra las últimas 50 por defecto
