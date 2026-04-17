# Guía de integración — API REST PetStays

Esta guía está pensada para **equipos que integran aplicaciones externas** (apps móviles, CRMs, channel managers, scripts propios) con el sistema de reservas **PetStays**. Aquí encontrarás la URL base, autenticación, endpoints y ejemplos listos para probar.

> **Importante:** La URL exacta y las credenciales las proporciona **PetStays / el hotel** para tu entorno (producción o pruebas). Sustituye `https://TU-DOMINIO.com` por el dominio que te indiquen.

---

## Qué permite este API

Con las credenciales del **propietario del hotel** (cuenta autorizada) puedes:

- Consultar **disponibilidad por tipo de habitación** en una sucursal y rango de fechas.
- **Listar, crear, consultar y anular** reservas asociadas a las sucursales de ese propietario.

| Concepto | Valor |
|----------|--------|
| Prefijo REST | `/wp-json/` |
| Versión del API PetStays | `petstays/v1` |
| **URL base** | `https://TU-DOMINIO.com/wp-json/petstays/v1` |

---

## Cómo obtener credenciales

1. El **hotel** debe tener activo el acceso al API de reservas (cuenta de tipo propietario autorizada).
2. Las claves se generan desde el **área de cuenta del sitio** (por ejemplo la sección *PetStays — API de reservas* en el panel del usuario), o mediante el administrador del sitio.
3. Recibirás un par:
   - **API Key** pública, prefijo `pk_…`
   - **Secret** privado, prefijo `sk_…` — **muéstralo y almacénalo con cuidado**; quien lo tenga puede actuar en nombre del hotel en este API.

Si pierdes el secret, el hotel debe **generar un par nuevo** (el anterior deja de ser válido).

---

## Autenticación

En **cada petición** HTTP debes enviar estas cabeceras:

| Cabecera | Valor |
|----------|--------|
| `X-PetStays-Api-Key` | Tu API Key (`pk_…`) |
| `X-PetStays-Secret` | Tu secret (`sk_…`) |

**Errores frecuentes**

| Código HTTP | Significado |
|-------------|-------------|
| **401** | Faltan cabeceras, clave incorrecta o secret incorrecto. |
| **403** | La cuenta no tiene permiso para usar este API o no puede acceder al recurso (p. ej. sucursal que no es suya). |

Las respuestas de error suelen incluir un cuerpo JSON con `code` y `message` (formato WordPress REST API).

---

## Uso desde navegador u otra web (CORS)

Las respuestas del API PetStays incluyen cabeceras **CORS** para permitir llamadas desde otros dominios (por ejemplo un front en otro servidor). Si tu integración es solo servidor a servidor, puedes ignorar este apartado.

---

## Identificadores que usarás

| Nombre en la documentación | Qué es |
|----------------------------|--------|
| **Sucursal** (`property_id`) | Identificador numérico de la sucursal del hotel en PetStays. Te lo facilita el hotel o PetStays. |
| **Tipo de habitación** (`room_type_id`) | Categoría de habitación en esa sucursal. |
| **Reserva** (`id` en rutas) | Identificador numérico de la reserva creada en PetStays. |

Las fechas de entrada/salida se envían preferiblemente como **`YYYY-MM-DD`**. El servidor puede aceptar otros formatos habituales, pero se recomienda unificar en `YYYY-MM-DD`.

---

## Endpoints

Todas las rutas de abajo van **después** de la URL base  
`https://TU-DOMINIO.com/wp-json/petstays/v1`

### 1. Listar reservas

**GET** `/reservations`

Devuelve reservas del hotel autenticado, con paginación.

| Parámetro (query) | Obligatorio | Descripción |
|-------------------|-------------|-------------|
| `property_id` | No | Filtra solo la sucursal indicada. Si no es una sucursal del propietario → error 403. |
| `page` | No | Página (por defecto `1`). |
| `per_page` | No | Tamaño de página (máximo `100`, por defecto `20`). |

**Respuesta 200 — ejemplo JSON**

Los valores son ilustrativos; en producción verás los datos reales del hotel. Algunos campos pueden llegar como cadena vacía `""` si no están rellenados en el sistema.

```json
{
  "items": [
    {
      "id": 41310,
      "title": "Reserva API — Ana Pérez",
      "status": "publish",
      "property_id": 41153,
      "room_type_id": 8821,
      "room_id": 15001,
      "check_in_date": "2026-04-20",
      "check_out_date": "2026-04-25",
      "guest_name": "Ana Pérez",
      "guest_email": "ana@example.com",
      "guest_phone": "+34 600 11 22 33",
      "booking_status": "pending",
      "total_price": "245.50",
      "booking_source": "petstays_rest_api",
      "modified_gmt": "2026-04-15 14:32:01"
    },
    {
      "id": 41288,
      "title": "Reserva de Habitación #1713187200",
      "status": "publish",
      "property_id": 41153,
      "room_type_id": 8820,
      "room_id": 14990,
      "check_in_date": "2026-05-01",
      "check_out_date": "2026-05-03",
      "guest_name": "Carlos Ruiz",
      "guest_email": "carlos@example.com",
      "guest_phone": "",
      "booking_status": "confirmed",
      "total_price": "180.00",
      "booking_source": "",
      "modified_gmt": "2026-04-10 09:15:00"
    }
  ],
  "total": 47,
  "page": 1,
  "per_page": 20
}
```

**Campos útiles para tu código**

| Campo | Tipo habitual | Uso típico |
|-------|----------------|------------|
| `id` | número | Identificador único de la reserva en PetStays. |
| `property_id` | número | Sucursal. |
| `room_type_id` | número | Tipo de habitación reservado. |
| `room_id` | número | Habitación física asignada (puede ser `0` si aún no hay asignación). |
| `check_in_date` / `check_out_date` | string `YYYY-MM-DD` | Rango de la estancia. |
| `booking_status` | string | Ej.: `pending`, `confirmed`, `payment_pending`, etc. |
| `total_price` | string | Precio tal como está guardado (trátalo como decimal en tu app). |
| `modified_gmt` | string | Última modificación (hora UTC). |

---

### 2. Obtener una reserva

**GET** `/reservations/{id}`

| Parámetro | Descripción |
|-----------|-------------|
| `id` | Número de la reserva. |

**Respuesta 200 — ejemplo JSON**

Misma forma que **un solo objeto** dentro de `items` del listado:

```json
{
  "id": 41310,
  "title": "Reserva API — Ana Pérez",
  "status": "publish",
  "property_id": 41153,
  "room_type_id": 8821,
  "room_id": 15001,
  "check_in_date": "2026-04-20",
  "check_out_date": "2026-04-25",
  "guest_name": "Ana Pérez",
  "guest_email": "ana@example.com",
  "guest_phone": "+34 600 11 22 33",
  "booking_status": "pending",
  "total_price": "245.50",
  "booking_source": "petstays_rest_api",
  "modified_gmt": "2026-04-15 14:32:01"
}
```

**404** — Reserva inexistente o no pertenece a las sucursales del propietario autenticado (ejemplo de cuerpo típico WordPress REST):

```json
{
  "code": "petstays_api_not_found",
  "message": "Reserva no encontrada.",
  "data": {
    "status": 404
  }
}
```

---

### 3. Crear reserva

**POST** `/reservations`  
**Content-Type:** `application/json`

| Campo (JSON) | Obligatorio | Descripción |
|--------------|-------------|-------------|
| `property_id` | Sí | ID de la sucursal. |
| `room_type_id` | Sí | ID del tipo de habitación. |
| `check_in_date` | Sí | Fecha de entrada. |
| `check_out_date` | Sí | Fecha de salida (debe ser posterior a la entrada). |
| `guest_name` | Sí | Nombre del huésped. |
| `guest_email` | Sí | Email del huésped. |
| `guest_phone` | Según caso | Puede ser obligatorio si el sistema debe crear un usuario cliente nuevo. |
| `guest_document` | Según caso | Puede ser obligatorio si el cliente no existe y hay reglas de alta de usuario. |
| `quantity` | No | Número de unidades (por defecto `1`). |
| `pet` | No | Objeto con datos de mascota: `name`, `type`, `breed`, `age`, `size`, `gender`, `sterilized`, `vaccinated`, `details`, etc. |
| `pet_details` | No | Texto libre con detalle de mascotas. |
| `booking_status` | No | Por ejemplo `pending`, `confirmed` o `waiting` (por defecto `pending`). |
| Precios opcionales | No | `price_per_night`, `nights`, `total_price`, `tax_percent`, `tax_amount`, `subtotal_before_tax` si el flujo lo requiere. |

**Respuesta 201 — ejemplo JSON**

Usa siempre `item` como fuente de verdad de la reserva recién creada; `id` se repite arriba por comodidad.

```json
{
  "id": 41310,
  "message": "Reserva creada.",
  "item": {
    "id": 41310,
    "title": "Reserva API — Ana Pérez",
    "status": "publish",
    "property_id": 41153,
    "room_type_id": 8821,
    "room_id": 15001,
    "check_in_date": "2026-04-20",
    "check_out_date": "2026-04-25",
    "guest_name": "Ana Pérez",
    "guest_email": "ana@example.com",
    "guest_phone": "+34 600 11 22 33",
    "booking_status": "pending",
    "total_price": "245.50",
    "booking_source": "petstays_rest_api",
    "modified_gmt": "2026-04-15 14:32:01"
  }
}
```

**Errores — ejemplos JSON**

Sin disponibilidad en PetStays para esas fechas (`409`):

```json
{
  "code": "petstays_api_no_availability",
  "message": "No hay suficientes habitaciones disponibles para esas fechas.",
  "data": {
    "status": 409
  }
}
```

Petición incompleta (`400`):

```json
{
  "code": "petstays_api_bad_request",
  "message": "property_id y room_type_id son obligatorios.",
  "data": {
    "status": 400
  }
}
```

Sin permiso sobre la sucursal (`403`):

```json
{
  "code": "petstays_api_forbidden",
  "message": "No tienes permiso sobre esta sucursal.",
  "data": {
    "status": 403
  }
}
```

> El campo `code` es estable para ramificar lógica en tu cliente; `message` puede variar ligeramente según idioma o versión.

**Errores habituales (resumen):** `400`, `403`, `404`, `409`.

#### Nota para hoteles con sistema externo sincronizado

Algunos hoteles tienen configurada una **sincronización con otro sistema** (PMS, CRM propio, etc.). En esos casos, al crear la reserva PetStays puede **validar primero** disponibilidad y alta en ese sistema. Si esa validación falla, la petición puede responder con error **sin crear la reserva en PetStays**. El mensaje de error y el código HTTP te indicarán el motivo. Si tu integración es solo con PetStays y el hotel no usa esa opción, puedes ignorar este párrafo.

---

### 4. Anular reserva (enviar a papelera)

**DELETE** `/reservations/{id}`

Marca la reserva como eliminada en PetStays (equivalente a moverla a papelera).

**Respuesta 200 — ejemplo JSON**

```json
{
  "id": 41310,
  "deleted": true,
  "message": "Reserva movida a la papelera."
}
```

---

### 5. Disponibilidad por tipo de habitación

**GET** `/availability/room-types`

Devuelve, para una sucursal y un rango de fechas, cuántas habitaciones hay por **tipo**, cuántas están ocupadas y cuántas libres, según el calendario de PetStays.

| Parámetro (query) | Obligatorio | Descripción |
|-------------------|-------------|-------------|
| `property_id` | Sí | ID de la sucursal. |
| `check_in_date` | Sí | Inicio del rango (recomendado `YYYY-MM-DD`). |
| `check_out_date` | Sí | Fin del rango. Debe ser **posterior** a `check_in_date`. La disponibilidad se calcula con el rango tipo **entrada inclusiva / salida exclusiva** (como una estancia de noche). |
| `room_type_id` | No | Si lo envías, solo se devuelve ese tipo. Si no existe en la sucursal → **404**. |

**Respuesta 200 — ejemplo JSON (sin filtro `room_type_id`)**

Un elemento en `items` por cada tipo de habitación configurado en esa sucursal.

```json
{
  "property_id": 41153,
  "room_type_id": null,
  "range": {
    "check_in_date": "2026-04-20",
    "check_out_date": "2026-04-25"
  },
  "items": [
    {
      "room_type_id": 8820,
      "room_type_name": "Estándar",
      "total_rooms": 8,
      "occupied_rooms": 3,
      "available_rooms": 5,
      "has_availability": true
    },
    {
      "room_type_id": 8821,
      "room_type_name": "Suite",
      "total_rooms": 3,
      "occupied_rooms": 3,
      "available_rooms": 0,
      "has_availability": false
    },
    {
      "room_type_id": 8822,
      "room_type_name": "Familiar",
      "total_rooms": 4,
      "occupied_rooms": 1,
      "available_rooms": 3,
      "has_availability": true
    }
  ]
}
```

**Respuesta 200 — ejemplo JSON (con `room_type_id=8821`)**

Solo una fila en `items`; `room_type_id` en la raíz repite el filtro.

```json
{
  "property_id": 41153,
  "room_type_id": 8821,
  "range": {
    "check_in_date": "2026-04-20",
    "check_out_date": "2026-04-25"
  },
  "items": [
    {
      "room_type_id": 8821,
      "room_type_name": "Suite",
      "total_rooms": 3,
      "occupied_rooms": 3,
      "available_rooms": 0,
      "has_availability": false
    }
  ]
}
```

**Cómo usar estos campos en tu código**

| Campo | Significado |
|-------|-------------|
| `total_rooms` | Habitaciones publicadas y disponibles para reserva de ese tipo en la sucursal. |
| `occupied_rooms` | Habitaciones que **chocan** con otra reserva en el rango pedido. |
| `available_rooms` | `total_rooms - occupied_rooms` (mínimo `0`). |
| `has_availability` | `true` si `available_rooms > 0`. |

**Errores — ejemplo JSON (`400`, fechas mal formadas o rango inválido)**

```json
{
  "code": "petstays_api_bad_dates_range",
  "message": "check_out_date debe ser mayor que check_in_date.",
  "data": {
    "status": 400
  }
}
```

---

## Respuestas de error (autenticación)

Si faltan o son incorrectas las cabeceras (`401`), el cuerpo suele ser similar a:

```json
{
  "code": "petstays_api_auth_invalid",
  "message": "API key no válida.",
  "data": {
    "status": 401
  }
}
```

Los textos de `message` pueden variar; usa siempre **`data.status`** y **`code`** para la lógica de reintentos y mensajes al usuario.

---

## Ejemplos con línea de comandos

Sustituye dominio y credenciales.

### Listar reservas

```bash
curl -s \
  -H "X-PetStays-Api-Key: pk_TU_CLAVE" \
  -H "X-PetStays-Secret: sk_TU_SECRETO" \
  "https://TU-DOMINIO.com/wp-json/petstays/v1/reservations?per_page=20&page=1"
```

### Consultar disponibilidad por tipo de habitación

```bash
curl -s \
  -H "X-PetStays-Api-Key: pk_TU_CLAVE" \
  -H "X-PetStays-Secret: sk_TU_SECRETO" \
  "https://TU-DOMINIO.com/wp-json/petstays/v1/availability/room-types?property_id=41153&check_in_date=2026-04-20&check_out_date=2026-04-25"
```

### Crear una reserva

```bash
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "X-PetStays-Api-Key: pk_TU_CLAVE" \
  -H "X-PetStays-Secret: sk_TU_SECRETO" \
  -d '{
    "property_id": 41153,
    "room_type_id": 100,
    "check_in_date": "2026-04-20",
    "check_out_date": "2026-04-25",
    "guest_name": "Ana Pérez",
    "guest_email": "ana@example.com",
    "guest_phone": "+34600112233",
    "quantity": 1,
    "pet": { "name": "Luna", "breed": "Golden", "size": "Grande" }
  }' \
  "https://TU-DOMINIO.com/wp-json/petstays/v1/reservations"
```

---

## Soporte y cambios

- Dudas sobre **URLs, IDs de sucursal o permisos**: contacta a **PetStays** o al hotel con el que integras.
- PetStays puede publicar **avisos de cambios** en este API; conviene acordar un canal (email, changelog, wiki interna) para integradores.

---

*Documentación orientada a integradores. Versión alineada al producto PetStays — API `petstays/v1`.*
