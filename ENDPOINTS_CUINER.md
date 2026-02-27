# Documentación API – Integración Cuiner ↔ Odoo

**API Cuiner:** [openapi.cuiner.net/documentacion](https://openapi.cuiner.net/documentacion)

**Convenciones**
- Base URL Odoo: `http://[HOST]:8069` (o la que use la instalación).
- Todas las peticiones **Cuiner → Odoo** llevan el header **`X-API-Key`** (clave configurada en Odoo). Opcional **`Content-Type: application/json`**.
- Si Odoo tiene varias bases de datos: añadir **`?db=nombre_bd`** a la URL.

---

# Parte 1: API de Odoo (Cuiner llama a Odoo)

---

## 1. GET `/api/cuiner/balance/<rfid_uid>`

Devuelve el saldo wallet del cliente identificado por RFID y las etiquetas (tags) del contacto.

### Request

| | |
|---|---|
| **Método** | `GET` |
| **Path** | `/api/cuiner/balance/{rfid_uid}` |
| **Headers** | `X-API-Key`: (obligatorio) |
| **Body** | No |

Ejemplo: `GET /api/cuiner/balance/044123456789?db=caldea`

### Response

**200 OK**
```json
{
  "success": true,
  "rfid_uid": "044123456789",
  "partner_id": 42,
  "balance": 150.50,
  "currency": "EUR",
  "tags": ["socio", "acceso_restaurante"],
  "cards": [
    { "id": 1, "code": "CARD-001", "points": 100.00, "program": "Programa Fidelidad" },
    { "id": 2, "code": "CARD-002", "points": 50.50, "program": "Programa Fidelidad" }
  ]
}
```

**401 Unauthorized**
```json
{
  "error": "MISSING_API_KEY",
  "message": "Header X-API-Key requerido"
}
```
o `"error": "INVALID_API_KEY"`

**404 Not Found**
```json
{
  "success": false,
  "error": "PARTNER_NOT_FOUND",
  "message": "No se encontró cliente con RFID: ..."
}
```

---

## 2. GET `/api/cuiner/profile/<rfid_uid>`

Devuelve el perfil del cliente por RFID (nombre, email, etiquetas). No incluye saldo.

### Request

| | |
|---|---|
| **Método** | `GET` |
| **Path** | `/api/cuiner/profile/{rfid_uid}` |
| **Headers** | `X-API-Key`: (obligatorio) |
| **Body** | No |

### Response

**200 OK**
```json
{
  "success": true,
  "rfid_uid": "044123456789",
  "partner_id": 42,
  "name": "Nombre del cliente",
  "email": "cliente@ejemplo.com",
  "etiquetas": ["socio", "acceso_restaurante"]
}
```

**401** – Igual que en balance (`MISSING_API_KEY` / `INVALID_API_KEY`).

**404**
```json
{
  "success": false,
  "error": "PARTNER_NOT_FOUND",
  "message": "No se encontró cliente con RFID: ..."
}
```

---

## 3. GET `/api/cuiner/profile/email/<email>`

Mismo que perfil pero identificando al cliente por email. En la URL codificar la arroba como `%40` (ej. `cliente%40empresa.com`).

### Request

| | |
|---|---|
| **Método** | `GET` |
| **Path** | `/api/cuiner/profile/email/{email}` |
| **Headers** | `X-API-Key`: (obligatorio) |
| **Body** | No |

### Response

**200 OK**
```json
{
  "success": true,
  "email": "cliente@empresa.com",
  "partner_id": 42,
  "name": "Nombre del cliente",
  "etiquetas": ["socio"]
}
```

**401** – Igual que en balance.  
**404** – `success: false`, `error: "PARTNER_NOT_FOUND"`.

---

## 4. POST `/api/cuiner/charge`

Registra un cargo en el wallet (descuenta un importe del saldo del cliente).

### Request

| | |
|---|---|
| **Método** | `POST` |
| **Path** | `/api/cuiner/charge` |
| **Headers** | `X-API-Key`: (obligatorio), `Content-Type: application/json` |
| **Body** | JSON-RPC 2.0 |

**Composición del body**

```json
{
  "jsonrpc": "2.0",
  "method": "call",
  "params": {
    "rfid_uid": "044123456789",
    "amount": 25.50,
    "cuiner_sale_id": "TICKET-12345",
    "date": "2026-02-16",
    "description": "Consumo restaurante - Mesa 5"
  },
  "id": 1
}
```

| Parámetro | Tipo | Obligatorio | Descripción |
|-----------|------|-------------|-------------|
| `rfid_uid` | string | Sí | Código RFID del cliente. |
| `amount` | number | Sí | Importe a descontar (> 0). |
| `cuiner_sale_id` | string | No | ID del ticket/venta en Cuiner (trazabilidad). |
| `date` | string | No | Fecha del ticket (ej. `YYYY-MM-DD` o `YYYY-MM-DD HH:MM:SS`). |
| `description` | string | No | Texto libre. |

### Response

HTTP 200. El contenido útil va en el cuerpo; para endpoints `type='json'` Odoo puede devolver el resultado dentro de un envelope JSON-RPC en `result`.

**Éxito (`result`)**
```json
{
  "success": true,
  "rfid_uid": "044123456789",
  "partner_id": 42,
  "amount_charged": 25.50,
  "remaining_balance": 125.00,
  "cuiner_sale_id": "TICKET-12345",
  "cards_used": [{ "card_id": 1, "deducted": 25.50 }]
}
```

**Error (`result`)**
```json
{
  "success": false,
  "error": "MISSING_PARAMETER",
  "message": "rfid_uid es obligatorio"
}
```

Códigos de error: `MISSING_PARAMETER`, `INVALID_AMOUNT`, `PARTNER_NOT_FOUND`, `INSUFFICIENT_BALANCE`, `MISSING_API_KEY`, `INVALID_API_KEY`.  
En `INSUFFICIENT_BALANCE` el cuerpo incluye además: `available_balance`, `required_amount`, `shortage`.

---

## 5. POST `/api/cuiner/cuadre`

Envía el resultado del cuadre de caja. Odoo lo guarda en el modelo de cuadres.

### Request

| | |
|---|---|
| **Método** | `POST` |
| **Path** | `/api/cuiner/cuadre` |
| **Headers** | `X-API-Key`: (obligatorio), `Content-Type: application/json` |
| **Body** | JSON-RPC 2.0 |

**Composición del body**

```json
{
  "jsonrpc": "2.0",
  "method": "call",
  "params": {
    "date": "2026-02-26",
    "surplus": 5.0,
    "shortage": 0.0,
    "reason": "Motivo opcional",
    "pos_session_id": 123
  },
  "id": 1
}
```

| Parámetro | Tipo | Obligatorio | Descripción |
|-----------|------|-------------|-------------|
| `date` | string | Sí | Fecha del cuadre en formato `YYYY-MM-DD`. |
| `surplus` | number | No | Sobrante (default 0). |
| `shortage` | number | No | Faltante (default 0). |
| `reason` | string | No | Motivo de la diferencia. |
| `pos_session_id` | number | No | ID de sesión PoS en Odoo, si aplica. |

### Response

**Éxito**
```json
{
  "success": true,
  "message": "Cuadre registrado"
}
```

**Error**
```json
{
  "success": false,
  "error": "MISSING_PARAMETER",
  "message": "date es obligatorio"
}
```
Otros códigos: `INVALID_DATE`, `INVALID_API_KEY`, etc.

---

## 6. POST `/api/cuiner/rectification`

Registra una rectificación como orden del TPV en la **sesión del día en curso**. Las sesiones cerradas no se modifican.

### Request

| | |
|---|---|
| **Método** | `POST` |
| **Path** | `/api/cuiner/rectification` |
| **Headers** | `X-API-Key`: (obligatorio), `Content-Type: application/json` |
| **Body** | JSON-RPC 2.0 |

**Composición del body**

```json
{
  "jsonrpc": "2.0",
  "method": "call",
  "params": {
    "numero_factura_rectificada": "FAC-2026-00123",
    "fecha_factura": "2026-02-20",
    "amount": -25.50,
    "reason": "Devolución cliente",
    "rfid_uid": "044123456789"
  },
  "id": 1
}
```

| Parámetro | Tipo | Obligatorio | Descripción |
|-----------|------|-------------|-------------|
| `numero_factura_rectificada` | string | Sí | Número de la factura original que se rectifica. |
| `fecha_factura` | string | Sí | Fecha de la factura original `YYYY-MM-DD`. |
| `amount` | number | Sí* | Importe total de la rectificación (puede ser negativo). *Obligatorio si no se envía `lines`. |
| `lines` | array | Sí* | Líneas: `[{"id": "101", "qty": 1, "price": -10.0}]`. `id` = id producto en Cuiner. *Obligatorio si no se envía `amount`. |
| `reason` | string | No | Motivo o descripción. |
| `rfid_uid` | string | No | RFID del cliente para asociar el pedido. |

### Response

**Éxito**
```json
{
  "success": true,
  "message": "Rectificación registrada en sesión en curso",
  "pos_order_id": 123,
  "pos_reference": "Rectificación FAC-2026-00123 (2026-02-20)",
  "numero_factura_rectificada": "FAC-2026-00123",
  "fecha_factura": "2026-02-20"
}
```

**Error**
```json
{
  "success": false,
  "error": "MISSING_PARAMETER",
  "message": "numero_factura_rectificada es obligatorio"
}
```
Códigos: `MISSING_PARAMETER`, `INVALID_DATE`, `CONFIG_ERROR`, `SESSION_ERROR`, `PRODUCT_NOT_FOUND`.

---

# Parte 2: API de Cuiner (Odoo llama a Cuiner)

Base: URL configurada en Odoo (Sincronización Cuiner).  
Headers: **`X-API-Key`**, **`Accept: application/json`**.

---

## 7. GET `.../restaurant/{restaurant_id}/application/{application_id}/menus`

Obtiene la lista de IDs de menús.

### Request

| | |
|---|---|
| **Método** | `GET` |
| **Path** | `{base_url}/restaurant/{restaurant_id}/application/{application_id}/menus` |
| **Headers** | `X-API-Key`, `Accept: application/json` |
| **Body** | No |

### Response

**200 OK** – Array de identificadores de menús (números o strings).

```json
[1, 2, 3]
```
o
```json
["menu_1", "menu_2"]
```

---

## 8. GET `.../restaurant/{restaurant_id}/application/{application_id}/menu/{menu_id}`

Obtiene el detalle de un menú (ítems con id, nombre, descripción, precio). Odoo mapea cada ítem a producto con `x_cuiner_id = cui_id_{id}`.

### Request

| | |
|---|---|
| **Método** | `GET` |
| **Path** | `{base_url}/restaurant/{restaurant_id}/application/{application_id}/menu/{menu_id}` |
| **Headers** | `X-API-Key`, `Accept: application/json` |
| **Body** | No |

### Response

**200 OK** – Objeto con al menos la clave `items`.

```json
{
  "items": {
    "101": {
      "id": "101",
      "name": "Café con leche",
      "description": "Café con leche",
      "price": 2.50
    },
    "102": {
      "id": "102",
      "name": "Bocadillo",
      "description": "",
      "price": 5.00
    }
  }
}
```

| Campo en ítem | Tipo | Descripción |
|---------------|------|-------------|
| `id` | string/number | Identificador del ítem (obligatorio para Odoo). |
| `name` | string | Nombre del producto. |
| `description` | string | Opcional. |
| `price` | number | Precio; si no viene, Odoo usa 0. |

---

## 9. GET `.../restaurant/{restaurant_id}/sales_by_date/{fecha_inicio}/{fecha_fin}`

Obtiene los IDs de ventas en un rango de fechas. Odoo usa el mismo día en inicio y fin para el cron diario.

### Request

| | |
|---|---|
| **Método** | `GET` |
| **Path** | `{base_url}/restaurant/{restaurant_id}/sales_by_date/{fecha_inicio}/{fecha_fin}` |
| **Headers** | `X-API-Key`, `Accept: application/json` |
| **Body** | No |

Fechas en formato **YYYY-MM-DD** (ej. `2026-02-26`).

### Response

**200 OK** – Array de IDs de ventas.

```json
["123", "124", "125"]
```
o
```json
[123, 124, 125]
```

Sin ventas: `[]`.

---

## 10. GET `.../restaurant/{restaurant_id}/sales/{sale_id}`

Obtiene el detalle de una venta. Odoo crea con ello el ticket en el TPV y el registro en el log.

### Request

| | |
|---|---|
| **Método** | `GET` |
| **Path** | `{base_url}/restaurant/{restaurant_id}/sales/{sale_id}` |
| **Headers** | `X-API-Key`, `Accept: application/json` |
| **Body** | No |

### Response

**200 OK** – Objeto con la estructura que Odoo espera.

```json
{
  "date": "2026-02-26 14:30:00",
  "rfid_uid": "044123456789",
  "type": "Ticket",
  "items": [
    { "id": "101", "qty": 2, "price": 2.50 },
    { "id": "102", "qty": 1, "price": 5.00 }
  ],
  "payments": [
    { "method": "Efectivo", "amount": 10.00 },
    { "method": "Tarjeta", "amount": 15.50 }
  ]
}
```

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| `date` | string | No | Fecha/hora de la venta. |
| `items` | array | Sí | Líneas: `id` (producto Cuiner), `qty`, `price`. Odoo busca producto por `x_cuiner_id = cui_id_{id}`. |
| `rfid_uid` | string | No | RFID del cliente; si existe en Odoo, se asocia al ticket. |
| `type` | string | No | Si es `"Factura"`, Odoo genera factura; si no, solo marca como pagado. |
| `payments` | array | No | Varios métodos: `[{"method": "Efectivo", "amount": 10}, ...]`. También se aceptan claves `payment_lines` o `payment_methods`. |

---

## Resumen rápido

| Quién | Endpoint | Función |
|-------|----------|---------|
| Cuiner → Odoo | GET `/api/cuiner/balance/<rfid>` | Saldo y tags. |
| Cuiner → Odoo | GET `/api/cuiner/profile/<rfid>` | Perfil y etiquetas por RFID. |
| Cuiner → Odoo | GET `/api/cuiner/profile/email/<email>` | Perfil y etiquetas por email. |
| Cuiner → Odoo | POST `/api/cuiner/charge` | Cargo en wallet. |
| Cuiner → Odoo | POST `/api/cuiner/cuadre` | Envío de cuadre. |
| Cuiner → Odoo | POST `/api/cuiner/rectification` | Rectificación en sesión en curso. |
| Odoo → Cuiner | GET `.../menus` | Lista de menús. |
| Odoo → Cuiner | GET `.../menu/{id}` | Detalle menú (ítems). |
| Odoo → Cuiner | GET `.../sales_by_date/...` | IDs de ventas por fechas. |
| Odoo → Cuiner | GET `.../sales/{id}` | Detalle de una venta. |
