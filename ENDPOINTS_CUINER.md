# Endpoints para la integración con Cuiner

Referencia de **qué devuelve cada endpoint**, **cómo enviar las peticiones** y **qué información necesita cada sistema** para conectarse.

**API Cuiner (origen de ventas):** [https://openapi.cuiner.net/documentacion](https://openapi.cuiner.net/documentacion)

---

## 1. Qué información necesita cada sistema

### 1.1. Información que debe tener Odoo para conectarse a Cuiner

Odoo llama a la API de Cuiner para **productos** y **ventas**. Esta configuración se guarda en **Ajustes → Sincronización Cuiner** (parámetros `ir.config_parameter`):

| Dato | Parámetro Odoo | Descripción |
|------|----------------|-------------|
| **API Key** | `caldea_cuiner_sync.api_key` | Clave que Cuiner asigna a Odoo para autenticar las peticiones. |
| **Restaurant ID** | `caldea_cuiner_sync.restaurant_id` | Identificador del restaurante en Cuiner. |
| **Application ID** | `caldea_cuiner_sync.application_id` | Identificador de la aplicación en Cuiner. |
| **TPV (PoS)** | `caldea_cuiner_sync.pos_config_id` | ID de la configuración de Punto de Venta donde se crean los tickets de ventas sincronizadas. |
| **Cliente por defecto** | `caldea_cuiner_sync.default_partner_id` | ID del partner usado cuando la venta no tiene `rfid_uid` o no se encuentra el contacto. |
| **Diario de ventas** | `caldea_cuiner_sync.sale_journal_id` | Diario contable para facturas generadas desde los tickets. |

Con esto Odoo puede:
- Llamar a `GET .../menus` y `GET .../menu/{id}` (productos).
- Llamar a `GET .../sales_by_date/...` y `GET .../sales/{id}` (ventas).

---

### 1.2. Información que debe tener Cuiner para conectarse a Odoo

Cuiner llama a la API de Odoo solo para **wallet** (consulta de saldo y cargo). Cuiner necesita:

| Dato | Descripción |
|------|-------------|
| **URL base de Odoo** | Ej. `http://midominio.com:8069` o `https://odoo.empresa.com`. Sin barra final. |
| **API Key** | La **misma** clave configurada en Odoo en Sincronización Cuiner. Odoo la valida en el header `X-API-Key`. |
| **Nombre de la base de datos** | Si Odoo tiene varias bases de datos, el nombre de la BD (ej. `prod`, `caldea`). Se envía en query: `?db=nombre_bd`. |

Con esto Cuiner puede:
- Llamar a `GET /api/cuiner/balance/<rfid_uid>` (consultar saldo y tags).
- Llamar a `GET /api/cuiner/profile/<rfid_uid>` (perfil y etiquetas por RFID).
- Llamar a `GET /api/cuiner/profile/email/<email>` (perfil y etiquetas por email; usar %40 para @).
- Llamar a `POST /api/cuiner/charge` (registrar cargo).
- Llamar a `POST /api/cuiner/cuadre` (enviar resultado del cuadre de caja).

No se usa usuario/contraseña: solo el header **X-API-Key**.

---

## 2. Endpoints que Cuiner llama (API de Odoo)

Base: `http://[HOST_ODOO]:8069`  
Headers en **todas** las peticiones:

- **`X-API-Key`**: API Key configurada en Odoo (obligatorio).
- **`Content-Type`**: `application/json` (recomendado).

Si hay varias bases de datos: añadir **`?db=nombre_bd`** a la URL.

---

### 2.1. GET `/api/cuiner/balance/<rfid_uid>`

**Uso:** Consultar el saldo total de wallet del cliente identificado por su código RFID.

**Tags del usuario (última incorporación):** Cuando Cuiner solicita datos usando el RFID del usuario, la respuesta incluye los **tags** (etiquetas/categorías) del contacto asociado a ese RFID en Odoo. Así Cuiner puede usar esa información para tarifas, permisos, segmentación o lógica de negocio (por ejemplo, identificar si es socio, empleado, accionista, etc.). Los tags se devuelven en la respuesta exitosa junto con `balance`, `cards` y el resto de datos del contacto.

**Cómo enviarlo:**

- **Método:** GET.
- **URL:** `/api/cuiner/balance/<rfid_uid>`  
  Ejemplo: `/api/cuiner/balance/044123456789`  
  Con otra BD: `/api/cuiner/balance/044123456789?db=caldea`
- **Headers:** `X-API-Key: [API_KEY]`, `Content-Type: application/json`.
- **Body:** No se envía body.

**Qué devuelve Odoo:**

- **200 OK – Éxito**
  - `success`: `true`
  - `rfid_uid`: código RFID consultado
  - `partner_id`: ID del contacto en Odoo
  - `balance`: saldo total (suma de todas las tarjetas de fidelidad activas)
  - `currency`: código de moneda (ej. `EUR`)
  - `cards`: array de tarjetas, cada una con `id`, `code`, `points`, `program`
  - **`tags`**: etiquetas/categorías del contacto asociado al RFID (para que Cuiner las use en tarifas, permisos, etc.)

  Ejemplo:
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

- **401 Unauthorized – Falta o inválida API Key**
  - `error`: `MISSING_API_KEY` o `INVALID_API_KEY`
  - `message`: texto descriptivo

- **404 Not Found – Cliente no encontrado**
  - `success`: `false`
  - `error`: `PARTNER_NOT_FOUND`
  - `message`: "No se encontró cliente con RFID: ..."

---

### 2.2. GET `/api/cuiner/profile/<rfid_uid>`

**Uso:** Obtener solo el perfil del cliente (etiquetas) por RFID para aplicar tarifa (caso 4.2). No incluye saldo wallet.

**Cómo enviarlo:** GET, header `X-API-Key`, sin body. Opcional `?db=nombre_bd`.

**Qué devuelve:** `success`, `rfid_uid`, `partner_id`, `name`, `email`, **`etiquetas`** (lista de categorías del contacto).

---

### 2.3. GET `/api/cuiner/profile/email/<email>`

**Uso:** Perfil por correo electrónico (caso 4.2 PDF: “correo electrónico como identificador único”). En la URL usar `%40` para la arroba (ej. `cliente%40empresa.com`).

**Qué devuelve:** `success`, `email`, `partner_id`, `name`, **`etiquetas`**. 404 si no existe el contacto.

---

### 2.4. POST `/api/cuiner/charge`

**Uso:** Registrar un cargo de wallet (descontar un importe del saldo del cliente).

**Cómo enviarlo:**

- **Método:** POST.
- **URL:** `/api/cuiner/charge` (y `?db=nombre_bd` si aplica).
- **Headers:** `X-API-Key: [API_KEY]`, `Content-Type: application/json`.
- **Body:** JSON. Odoo usa ruta `type='json'`, por lo que acepta **JSON-RPC 2.0**:

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

  Campos del objeto `params`:
  - **`rfid_uid`** (obligatorio): código RFID del cliente (debe coincidir con el campo “Código de barras” del contacto en Odoo).
  - **`amount`** (obligatorio): importe a descontar (número > 0).
  - **`cuiner_sale_id`** (opcional): ID del ticket/venta en Cuiner (trazabilidad).
  - **`date`** (opcional): fecha del ticket (ej. `YYYY-MM-DD` o `YYYY-MM-DD HH:MM:SS`).
  - **`description`** (opcional): texto libre.

**Qué devuelve Odoo:**

La respuesta es JSON-RPC: el resultado real está en `result`. Siempre HTTP 200; el éxito o error se ve en `result.success` y `result.error`.

- **Éxito** – `result.success === true`
  - `rfid_uid`, `partner_id`
  - `amount_charged`: importe descontado (debe ser el solicitado)
  - `remaining_balance`: saldo restante después del cargo
  - `cuiner_sale_id`: el que enviaste (o generado)
  - `cards_used`: array con `card_id` y `deducted` por cada tarjeta usada

  Ejemplo:
  ```json
  {
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
      "success": true,
      "rfid_uid": "044123456789",
      "partner_id": 42,
      "amount_charged": 25.50,
      "remaining_balance": 125.00,
      "cuiner_sale_id": "TICKET-12345",
      "cards_used": [{ "card_id": 1, "deducted": 25.50 }]
    }
  }
  ```

- **Error** – `result.success === false`, con `result.error` y `result.message`:
  - **MISSING_PARAMETER**: falta `rfid_uid` (u otro obligatorio).
  - **INVALID_AMOUNT**: `amount` faltante o ≤ 0.
  - **PARTNER_NOT_FOUND**: no existe contacto con ese RFID en Odoo.
  - **INSUFFICIENT_BALANCE**: saldo insuficiente; además devuelve `available_balance`, `required_amount`, `shortage`.
  - **MISSING_API_KEY** / **INVALID_API_KEY**: si la autenticación falla (también en `result`).

Ejemplo de error saldo insuficiente:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": false,
    "error": "INSUFFICIENT_BALANCE",
    "message": "Saldo insuficiente",
    "available_balance": 100.00,
    "required_amount": 150.00,
    "shortage": 50.00
  }
}
```

---

### 2.5. POST `/api/cuiner/cuadre`

**Uso:** Enviar a Odoo el resultado del cuadre de caja (caso 4.6). Body JSON (o JSON-RPC params): `date` (YYYY-MM-DD, obligatorio), `surplus`, `shortage`, `reason`, `pos_session_id` (opcional). Odoo guarda el registro en `cuiner.cuadre`.

**Qué devuelve:** `success: true` y `message: "Cuadre registrado"`, o error con `success: false` y `error`, `message`.

---

## 3. Endpoints que Odoo llama (API de Cuiner)

Base: la configurada en Odoo en Sincronización Cuiner.  
Headers: **`X-API-Key`** (API Key de Cuiner para Odoo), **`Accept: application/json`**.

Cuiner debe exponer estos endpoints y devolver el formato que Odoo espera; así Odoo puede sincronizar productos y ventas.

---

### 3.1. GET `.../restaurant/{restaurant_id}/application/{application_id}/menus`

**Uso (en Odoo):** Obtener la lista de IDs de menús para luego pedir el detalle de cada uno.

**Cómo lo envía Odoo:**

- GET, con headers `X-API-Key` y `Accept: application/json`.
- Sin body.

**Qué debe devolver Cuiner:**

- **200 OK:** Un **array de identificadores** de menús (números o strings).  
  Ejemplo: `[1, 2, 3]` o `["menu_1", "menu_2"]`.

---

### 3.2. GET `.../restaurant/{restaurant_id}/application/{application_id}/menu/{menu_id}`

**Uso (en Odoo):** Obtener el detalle de un menú para crear/actualizar productos.

**Cómo lo envía Odoo:**

- GET con el `menu_id` obtenido del endpoint de menús.
- Sin body.

**Qué debe devolver Cuiner:**

- **200 OK:** Un objeto JSON que tenga al menos la clave **`items`**.
- **`items`**: objeto/diccionario donde cada clave es un identificador de ítem y el valor es un objeto con:
  - **`id`**: identificador del ítem (obligatorio para sincronizar).
  - **`name`**: nombre del producto.
  - **`description`**: descripción (opcional).
  - **`price`**: precio (número; si no viene, Odoo usa 0).

Ejemplo de estructura que Odoo usa:
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

Odoo mapea cada ítem a un producto con `x_cuiner_id = cui_id_{item.id}` (ej. `cui_id_101`). Si falta `id`, Odoo ignora ese ítem.

---

### 3.3. GET `.../restaurant/{restaurant_id}/sales_by_date/{fecha_inicio}/{fecha_fin}`

**Uso (en Odoo):** Obtener los IDs de ventas de un rango de fechas (normalmente el mismo día).

**Cómo lo envía Odoo:**

- GET; las fechas en formato **YYYY-MM-DD** (ej. `2026-02-26`).

**Qué debe devolver Cuiner:**

- **200 OK:** Un **array de IDs de ventas** (números o strings).  
  Ejemplo: `["123", "124", "125"]` o `[123, 124, 125]`.

Si no hay ventas, puede devolver `[]`.

---

### 3.4. GET `.../restaurant/{restaurant_id}/sales/{sale_id}`

**Uso (en Odoo):** Obtener el detalle de una venta para crear el ticket PoS y el log.

**Cómo lo envía Odoo:**

- GET con el `sale_id` obtenido de `sales_by_date`.
- Sin body.

**Qué debe devolver Cuiner:**

- **200 OK:** Un objeto JSON con al menos:
  - **`date`**: fecha/hora de la venta (Odoo lo guarda en el log).
  - **`items`**: **array** de líneas, cada una con:
    - **`id`**: identificador del producto/ítem en Cuiner (debe coincidir con el que se usa en menús para que Odoo encuentre el producto por `x_cuiner_id = cui_id_{id}`).
    - **`qty`**: cantidad (por defecto 1 si no viene).
    - **`price`**: precio unitario (por defecto 0 si no viene).
  - **`rfid_uid`** (opcional): código RFID del cliente; si viene y existe en Odoo (barcode del partner), se asocia el ticket a ese cliente; si no, se usa el “cliente por defecto”.
  - **`type`** (opcional): si es `"Factura"`, Odoo genera factura del ticket; si no, solo marca como pagado.
  - **`payments`** (opcional, caso 4.1): lista de pagos para varios métodos, ej. `[{"method": "Efectivo", "amount": 10}, {"method": "Tarjeta", "amount": 15.50}]`. Si viene, Odoo registra un pago por cada ítem (por nombre de método o `method_id`). También se aceptan claves `payment_lines` o `payment_methods`.

Ejemplo de estructura que Odoo espera:
```json
{
  "date": "2026-02-26 14:30:00",
  "rfid_uid": "044123456789",
  "type": "Ticket",
  "items": [
    { "id": "101", "qty": 2, "price": 2.50 },
    { "id": "102", "qty": 1, "price": 5.00 }
  ]
}
```

Si un ítem tiene `id` que no existe en Odoo como `cui_id_{id}`, la sincronización de esa venta falla con error “Producto Cuiner '...' no encontrado en Odoo”.

---

## 4. Resumen rápido

| Quién llama | Endpoint | Cómo enviar | Qué devuelve / qué debe devolver |
|-------------|----------|-------------|-----------------------------------|
| **Cuiner → Odoo** | GET `/api/cuiner/balance/<rfid_uid>` | GET + header `X-API-Key` (+ `?db=...`) | 200: `success`, `balance`, `currency`, `tags`, `cards`. 401: API Key. 404: cliente no encontrado. |
| **Cuiner → Odoo** | GET `/api/cuiner/profile/<rfid_uid>` | GET + `X-API-Key` | 200: `success`, `partner_id`, `name`, `email`, `etiquetas`. |
| **Cuiner → Odoo** | GET `/api/cuiner/profile/email/<email>` | GET + `X-API-Key` (email con %40 para @) | 200: `success`, `email`, `partner_id`, `name`, `etiquetas`. 404 si no existe. |
| **Cuiner → Odoo** | POST `/api/cuiner/charge` | POST + body JSON-RPC con `params`: `rfid_uid`, `amount`, opc. `cuiner_sale_id`, `date`, `description` | 200 + `result`: `success`, `amount_charged`, `remaining_balance`, o `success: false` + `error`, `message` (y en INSUFFICIENT_BALANCE: `available_balance`, `required_amount`, `shortage`). |
| **Cuiner → Odoo** | POST `/api/cuiner/cuadre` | POST JSON: `date`, `surplus`, `shortage`, `reason`, opc. `pos_session_id` | 200: `success`, `message`. |
| **Odoo → Cuiner** | GET `.../menus` | GET + `X-API-Key` | Array de IDs de menús. |
| **Odoo → Cuiner** | GET `.../menu/{menu_id}` | GET + `X-API-Key` | Objeto con `items`: diccionario de ítems con `id`, `name`, `description`, `price`. |
| **Odoo → Cuiner** | GET `.../sales_by_date/{fecha}/{fecha}` | GET + `X-API-Key` | Array de IDs de ventas. |
| **Odoo → Cuiner** | GET `.../sales/{sale_id}` | GET + `X-API-Key` | Objeto con `date`, `items` (array con `id`, `qty`, `price`), opc. `rfid_uid`, `type`, `payments` (varios métodos). |

Documentación más detallada de la API Wallet (ejemplos cURL, Python, códigos de error): ver `../caldea_cuiner_sync_docs/DOCUMENTACION_API_CUINER.md` si está disponible en el proyecto.
