# Endpoints integración Cuiner ↔ Odoo

**API Cuiner:** [openapi.cuiner.net/documentacion](https://openapi.cuiner.net/documentacion)

Todas las llamadas de Cuiner a Odoo llevan el header **X-API-Key** (la misma clave configurada en Odoo). Si hay varias bases de datos: `?db=nombre_bd`.

---

## Qué necesita cada uno

| Quién | Necesita |
|-------|----------|
| **Odoo → Cuiner** | API Key, Restaurant ID, Application ID (en Ajustes > Sincronización Cuiner). |
| **Cuiner → Odoo** | URL base de Odoo y la misma API Key. |

---

## Cuiner llama a Odoo (API de Odoo)

Base: `http://[HOST]:8069`

| Endpoint | Qué hace |
|----------|----------|
| **GET** `/api/cuiner/balance/<rfid_uid>` | Devuelve saldo wallet del cliente por RFID y las etiquetas (tags) del contacto. |
| **GET** `/api/cuiner/profile/<rfid_uid>` | Devuelve perfil del cliente por RFID (nombre, email, etiquetas). Sin saldo. |
| **GET** `/api/cuiner/profile/email/<email>` | Igual que perfil pero por email. En la URL usar `%40` para la arroba. |
| **POST** `/api/cuiner/charge` | Registra un cargo en el wallet (descuenta importe del saldo). Body JSON-RPC: `rfid_uid`, `amount`; opc. `cuiner_sale_id`, `date`, `description`. |
| **POST** `/api/cuiner/cuadre` | Envía el resultado del cuadre de caja. Body: `date` (YYYY-MM-DD), `surplus`, `shortage`, `reason`; opc. `pos_session_id`. Odoo lo guarda. |
| **POST** `/api/cuiner/rectification` | Registra una rectificación como orden del TPV en la **sesión del día en curso**. Body: `numero_factura_rectificada`, `fecha_factura` (YYYY-MM-DD), `amount` o `lines`; opc. `reason`, `rfid_uid`. Las sesiones cerradas no se modifican. |

---

## Odoo llama a Cuiner (API de Cuiner)

Base: la configurada en Odoo. Header **X-API-Key**, **Accept: application/json**.

| Endpoint | Qué hace |
|----------|----------|
| **GET** `.../restaurant/{id}/application/{app_id}/menus` | Devuelve la lista de IDs de menús. |
| **GET** `.../restaurant/{id}/application/{app_id}/menu/{menu_id}` | Devuelve el detalle del menú: objeto con `items` (id, name, description, price). Odoo crea/actualiza productos con `x_cuiner_id = cui_id_{id}`. |
| **GET** `.../restaurant/{id}/sales_by_date/{fecha_inicio}/{fecha_fin}` | Devuelve array de IDs de ventas del rango de fechas (YYYY-MM-DD). |
| **GET** `.../restaurant/{id}/sales/{sale_id}` | Devuelve detalle de la venta: `date`, `items` (id, qty, price), opc. `rfid_uid`, `type` ("Factura" o no), `payments` (varios métodos). Odoo crea el ticket en el TPV y el log. |

---

## Resumen en tabla

| Quién llama | Endpoint | Qué hace |
|-------------|----------|----------|
| Cuiner → Odoo | GET `/api/cuiner/balance/<rfid>` | Saldo y tags del cliente. |
| Cuiner → Odoo | GET `/api/cuiner/profile/<rfid>` | Perfil y etiquetas por RFID. |
| Cuiner → Odoo | GET `/api/cuiner/profile/email/<email>` | Perfil y etiquetas por email. |
| Cuiner → Odoo | POST `/api/cuiner/charge` | Cargo en wallet. |
| Cuiner → Odoo | POST `/api/cuiner/cuadre` | Envío de cuadre de caja. |
| Cuiner → Odoo | POST `/api/cuiner/rectification` | Rectificación en sesión en curso. |
| Odoo → Cuiner | GET `.../menus` | Lista de menús. |
| Odoo → Cuiner | GET `.../menu/{id}` | Detalle menú (ítems). |
| Odoo → Cuiner | GET `.../sales_by_date/...` | IDs de ventas por fechas. |
| Odoo → Cuiner | GET `.../sales/{id}` | Detalle de una venta. |
