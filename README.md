# Caldea Cuiner Sync — Cómo funciona el módulo y endpoints para Cuiner

Repositorio de documentación del módulo **Caldea Cuiner Sync** (Odoo 19): funcionamiento interno y endpoints que intervienen en la integración con Cuiner.

---

## 1. Resumen del módulo

El módulo integra **Odoo** con **Cuiner OpenAPI** para:

- **Sincronizar productos** desde Cuiner (menús e ítems) hacia Odoo.
- **Sincronizar ventas** desde Cuiner hacia Odoo como tickets de Punto de Venta (PoS).
- **Exponer una API REST** para que Cuiner consulte saldo de wallet y registre cargos en tiempo real.

La comunicación es en **dos direcciones**:

| Dirección | Qué hace |
|-----------|----------|
| **Odoo → Cuiner** | Odoo llama a la API de Cuiner para obtener menús, ítems y ventas. |
| **Cuiner → Odoo** | Cuiner llama a la API de Odoo para consultar saldo y procesar cargos de wallet. |

---

## 2. Cómo funciona el módulo

### 2.1. Sincronización de productos (Odoo → Cuiner)

- **Trigger:** Cron **“Cuiner: Sincronización de Productos”** (cada 1 hora).
- **Modelo:** `product.template` → método `cron_sync_cuiner_products()`.
- **Flujo:**
  1. Lee configuración (Base URL, Restaurant ID, Application ID, API Key).
  2. Llama a Cuiner: `GET .../menus` → obtiene lista de IDs de menús.
  3. Para cada menú: `GET .../menu/{menu_id}` → obtiene detalle con `items`.
  4. Por cada ítem crea o actualiza un producto en Odoo con `x_cuiner_id = cui_id_{id}`.
- **Datos sincronizados:** nombre, descripción, precio de venta, tipo `consu`, venta permitida.

### 2.2. Sincronización de ventas (Odoo → Cuiner)

- **Trigger:** Cron **“Cuiner: Sincronización de Ventas (Diario)”** (1 vez al día, recomendado ~2:00).
- **Modelo:** `cuiner.sync` (abstract) → método `cron_sync_cuiner_sales()`.
- **Flujo:**
  1. Obtiene o crea una **sesión PoS** única para el día (TPV configurado en ajustes).
  2. Llama a Cuiner: `GET .../sales_by_date/{fecha}/{fecha}` → lista de IDs de ventas del día.
  3. Para cada ID, comprueba en `cuiner.sale.log` si ya está sincronizado (`state = done`); si ya existe, se omite.
  4. Para cada venta nueva: `GET .../sales/{sale_id}` → detalle de la venta.
  5. Identifica cliente por `rfid_uid` (barcode en `res.partner`) o usa cliente por defecto.
  6. Busca productos en Odoo por `x_cuiner_id = cui_id_{item.id}` y crea líneas del ticket.
  7. Crea `pos.order` en la sesión del día, registra pago y marca como pagado (o factura si `type == 'Factura'`).
  8. Registra el resultado en `cuiner.sale.log` (done/error).
  9. Al finalizar, cierra la sesión PoS.

### 2.3. Wallet en tiempo real (Cuiner → Odoo)

- **Uso:** Pagos con wallet en Cuiner deben reflejarse al instante en Odoo.
- **Flujo típico:**
  1. Cuiner consulta en Odoo el saldo del cliente por RFID → `GET /api/cuiner/balance/<rfid_uid>`.
  2. Si el saldo es suficiente, el cliente confirma el pago.
  3. Cuiner notifica el cargo a Odoo → `POST /api/cuiner/charge` con `rfid_uid`, `amount`, etc.
  4. Odoo descuenta exactamente ese importe del loyalty y responde con éxito o error (p. ej. saldo insuficiente).

Toda la lógica de balance y cargo está en el controlador **CuinerAPIController** (`controllers/cuiner_api.py`).

---

## 3. Endpoints para la integración con Cuiner

**Documentación detallada:** En **`ENDPOINTS_CUINER.md`** se explica para cada endpoint: **qué devuelve**, **cómo hay que enviar la petición** (método, URL, headers, body) y **qué información debe tener Odoo para conectarse a Cuiner** y **qué información debe tener Cuiner para conectarse a Odoo**.

Se distinguen dos grupos:

- **A) Endpoints que Odoo expone para Cuiner** (Cuiner llama a Odoo).
- **B) Endpoints de la API de Cuiner que Odoo consume** (Odoo llama a Cuiner).

---

### A) Endpoints que Odoo expone para Cuiner

Base URL de Odoo: `http://[HOST_ODOO]:8069`  
Autenticación: header **`X-API-Key`** con la API Key configurada en Odoo (Ajustes → Sincronización Cuiner).

#### A.1. Consultar saldo de wallet

| Método | URL | Descripción |
|--------|-----|-------------|
| **GET** | `/api/cuiner/balance/<rfid_uid>` | Devuelve el saldo total de wallet del cliente identificado por `rfid_uid` (código RFID / barcode del contacto). |

**Query opcional:** `db=<nombre_bd>` si hay varias bases de datos.

**Respuesta exitosa (200):** JSON con `success`, `balance`, `currency`, `cards` (detalle de tarjetas de fidelidad) y **`tags`** (etiquetas/categorías del contacto asociado al RFID, para que Cuiner las use en tarifas, permisos o segmentación).

**Errores:** 401 si falta o es inválida la API Key; 404 si no existe partner con ese RFID.

#### A.2. Procesar cargo de wallet

| Método | URL | Descripción |
|--------|-----|-------------|
| **POST** | `/api/cuiner/charge` | Registra un cargo de wallet: descuenta `amount` del saldo del cliente identificado por `rfid_uid`. |

**Body (JSON):**  
`rfid_uid` (obligatorio), `amount` (obligatorio, > 0), `cuiner_sale_id` (opcional), `date` (opcional), `description` (opcional).

**Respuesta:** JSON con `success`, `amount_charged`, `remaining_balance`, `cards_used`, o error (p. ej. `INSUFFICIENT_BALANCE`, `PARTNER_NOT_FOUND`).

Documentación detallada de estos endpoints (ejemplos, códigos de error, cURL, Python, etc.):  
ver en el mismo proyecto la documentación de la API Wallet (p. ej. `../caldea_cuiner_sync_docs/DOCUMENTACION_API_CUINER.md` si está disponible).

---

### B) Endpoints de la API de Cuiner que Odoo consume

Base URL de Cuiner: la configurada en Odoo (Ajustes → Sincronización Cuiner). No se documenta aquí la URL del proveedor.  
Autenticación: header **`X-API-Key`** con la API Key configurada para Cuiner.

En las URLs, `{base_url}` = Base URL configurada en Odoo, `{restaurant_id}` y `{application_id}` = IDs configurados en Odoo.

#### B.1. Productos (sincronización de catálogo)

| Método | URL | Uso en el módulo |
|--------|-----|-------------------|
| **GET** | `{base_url}/restaurant/{restaurant_id}/application/{application_id}/menus` | Lista de IDs de menús. |
| **GET** | `{base_url}/restaurant/{restaurant_id}/application/{application_id}/menu/{menu_id}` | Detalle del menú con `items` (id, name, description, price, etc.) para crear/actualizar productos. |

#### B.2. Ventas (sincronización diaria)

| Método | URL | Uso en el módulo |
|--------|-----|-------------------|
| **GET** | `{base_url}/restaurant/{restaurant_id}/sales_by_date/{fecha_inicio}/{fecha_fin}` | Lista de IDs de ventas del día (ej. `YYYY-MM-DD`). |
| **GET** | `{base_url}/restaurant/{restaurant_id}/sales/{sale_id}` | Detalle de una venta (fecha, items con id/qty/price, rfid_uid, type, etc.) para crear el ticket PoS y el log. |

---

## 4. Configuración en Odoo

En **Ajustes** (o Inventario, según menú) → **Sincronización Cuiner**:


- **API Key**: misma clave para llamadas a Cuiner y para que Cuiner llame a Odoo (Wallet).
- **Restaurant ID** y **Application ID**: identificadores del restaurante y aplicación en Cuiner.
- **TPV para Sincronización**: configuración PoS donde se crean los tickets de ventas sincronizadas.
- **Cliente por defecto**: partner usado cuando la venta no tiene `rfid_uid` o no se encuentra el contacto.
- **Diario de ventas**: diario contable para facturas generadas desde los tickets.

---

## 5. Trazabilidad y logs

- **Logs de ventas Cuiner:** menú **Punto de Venta → Informes → Logs de Ventas Cuiner** (`cuiner.sale.log`). Cada venta sincronizada (o fallo) queda registrada con `cuiner_sale_id`, estado, cliente, mensaje de error si aplica.
- **Wallet:** cada cargo desde Cuiner se registra en el historial de loyalty y en `cuiner.sale.log` (cuando aplica).

---

## 6. Dependencias del módulo

- `product`, `sale`, `stock`
- `point_of_sale`
- `rfid_management`
- `sale_loyalty`

---

## 7. Referencias

- Código del módulo: `caldea_cuiner_sync` ().
- Controlador API (Wallet): `controllers/cuiner_api.py`.
- Motor de sincronización: `models/cuiner_sync.py`.
- Sincronización de productos: `models/product_template.py`.


---

**Versión:** 1.0 · **Odoo:** 19 · **Proyecto:** Caldea / Vanture
