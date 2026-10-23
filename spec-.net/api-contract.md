# Contrato de la API — Manager Library
El contrato HTTP es **independiente del lenguaje**: cualquier implementación debe respetarlo, independientemente de la tecnología utilizada. Qué hace cada operación está en [`spec.md`](spec.md); qué se guarda, en [`data-model.md`](data-model.md).

## 1. Convenciones

| Tema | Regla |
|---|---|
| Base | `/api/v1` |
| Formato | JSON, UTF-8. Nombres de campo en `snake_case` |
| Identificadores | UUID v4 como texto |
| Fechas | ISO 8601 en UTC, con `Z` (`2026-10-14T14:15:00Z`) |
| Dinero | No hay. El sistema no maneja importes |
| Autenticación | `Authorization: Bearer <token>` en todo salvo `POST /auth/login` y `GET /health` |
| Paginación | `?page=1&size=20`. `size` máximo 50. La respuesta es `{ "items": [...], "page": 1, "size": 20, "total": 134 }` |
| Idioma | Mensajes de error en español; campos y códigos en inglés |

### Cuerpo de error (único para todos los errores)

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No quedan ejemplares disponibles de «Cien años de soledad».",
  "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ]
}
```

`details` puede ir vacío. **Nunca** incluye trazas, SQL, rutas del servidor ni la contraseña.

### Códigos de estado

| Estado | Cuándo |
|---|---|
| 200 / 201 / 204 | Éxito / creado / sin contenido |
| 401 | Sin token, token inválido o vencido, credenciales inválidas |
| 403 | Rol insuficiente |
| 404 | No existe (o está dado de baja) |
| 409 | Conflicto de estado: duplicado, sin ejemplares, ya devuelto, socio con vencidos, libro con préstamos abiertos |
| 422 | Validación: campo inválido, regla de entrada rota, rango de fechas invertido |
**### Reglas de respuestas HTTP**

- Todas las respuestas de error deben utilizar el cuerpo común definido en esta sección, con los campos `code`, `message` y `details`.
- `details` puede ser una lista vacía cuando no existan detalles adicionales.
- Las respuestas `401`, `403`, `404`, `409` y `422` deben utilizar los códigos de error de negocio definidos en la sección 2 cuando corresponda.
- Las respuestas de error nunca deben incluir contraseñas, hashes de contraseñas, trazas de ejecución, consultas SQL ni rutas internas del servidor.
- Las operaciones exitosas deben respetar el código HTTP y el formato de respuesta especificados para cada endpoint.

## 2. Códigos de error de negocio

| `code` | Estado | Origen |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | CA-01.2 |
| `FORBIDDEN_ROLE` | 403 | CA-01.3 |
| `NOT_FOUND` | 404 | Cualquier recurso |
| `USERNAME_TAKEN` | 409 | CA-02.2 |
| `ISBN_TAKEN` | 409 | CA-03.2 |
| `DOCUMENT_TAKEN` | 409 | CA-05.1 |
| `BOOK_HAS_OPEN_LOANS` | 409 | CA-03.5 |
| `BOOK_NOT_AVAILABLE` | 409 | CA-06.3 |
| `MEMBER_HAS_OVERDUE` | 409 | CA-06.5 |
| `ITEM_ALREADY_RETURNED` | 409 | CA-07.2 |
| `MEMBER_LIMIT_EXCEEDED` | 422 | CA-06.4 |
| `DUPLICATE_BOOK_IN_LOAN` | 422 | CA-06.2 |
| `VALIDATION_ERROR` | 422 | Cualquier campo inválido |
| `INVALID_DATE_RANGE` | 422 | CA-10.2 |

## 3. Endpoints

### 3.1 Autenticación y salud

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | `/auth/login` | público | Devuelve token |
| GET | `/health` | público | `{ "status": "ok" }` |

`POST /auth/login`

```json
// petición
{ "username": "ana", "password": "********" }
// 200
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 28800, "role": "librarian" }
```

### 3.2 Usuarios

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | `/users` | admin | Crea un bibliotecario |

```json
// petición  (role solo admite "librarian")
{ "username": "Ana ", "password": "********", "role": "librarian" }
// 201  (sin password ni password_hash)
{ "id": "…", "username": "ana", "role": "librarian" }
```

### 3.3 Categorías

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/categories` | autenticado | Las cinco categorías, orden por nombre |

### 3.4 Libros

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/books` | Autenticado | Busca coincidencias parciales en el título o autor mediante `q`; permite filtrar por `category_id` y paginar con `page` y `size` (máximo 50 resultados por página). Devuelve únicamente libros activos, ordenados por título ascendente. La respuesta incluye `items`, `page`, `size` y `total`, que indica el total de resultados antes de la paginación. |
| GET | `/books/{id}` | autenticado | Detalle |
| POST | `/books` | admin | Crea |
| PUT | `/books/{id}` | admin | Reemplaza título, autor, categoría y total |
| DELETE | `/books/{id}` | admin | Baja lógica |
| PUT | `/books/{id}/cover` | admin | Sube portada (`multipart/form-data`, campo `file`) |
| DELETE | `/books/{id}/cover` | admin | Quita la portada |
| GET | `/books/{id}/cover` | autenticado | Devuelve el archivo de imagen de la portada. Responde `404 Not Found` si el libro no existe, está dado de baja o no tiene portada. |
Libro (respuesta):

```json
{
  "id": "…", "title": "Cien años de soledad", "author": "Gabriel García Márquez",
  "isbn": "9780307474728", "category": { "id": "…", "name": "Novela" },
  "total_copies": 4, "available_copies": 3, "cover_key": null
}
```

`POST /books` (petición): `title`, `author`, `isbn`, `category_id`, `total_copies`. Los ejemplares disponibles (`available_copies`) los calcula el servidor y se inicializan con el valor de `total_copies`.

`PUT /books/{id}` (actualización): al modificar `total_copies`, el servidor ajusta `available_copies` según la diferencia entre la cantidad total nueva y la anterior. La actualización debe rechazarse con `409 Conflict` si la nueva cantidad total es menor que la cantidad de ejemplares actualmente prestados. En ningún caso `available_copies` puede ser menor que `0` ni mayor que `total_copies`.

`cover_key` es una clave opaca: para mostrar la portada se usa `GET /books/{id}/cover`, que devuelve el archivo de imagen o `404 Not Found` si el libro no existe, está eliminado o no tiene portada.

**Reglas para las portadas**

- Solo se permiten archivos JPG o PNG de máximo 2 MB.
- El servidor debe validar el tipo de archivo por su contenido, no solamente por su extensión.
- `cover_key` almacena una clave opaca, nunca una ruta física ni el contenido binario.
- Al quitar una portada, primero se debe establecer `cover_key` en `null` y confirmar la transacción. Después se elimina físicamente el archivo.
- Si el libro no existe o está dado de baja, la operación debe responder `404`.
- Si el libro no tiene portada, `DELETE /books/{id}/cover` debe responder de forma consistente, sin provocar una eliminación incorrecta.

### 3.5 Socios

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/members` | admin, librarian | Lista y busca: `q`, `page`, `size` |
| GET | `/members/{id}` | admin, librarian | Detalle con `active_loans` (conteo sin devolver) |
| POST | `/members` | admin, librarian | Crea: `full_name`, `document_number` |

### 3.6 Préstamos

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | `/loans` | admin, librarian | Crea un préstamo para un socio. La petición incluye `member_id` y entre 1 y 3 elementos en `items`, cada uno con un `book_id` diferente. El servidor registra como responsable al usuario autenticado, establece `loaned_at` en UTC y calcula `due_at` a 14 días. La creación del préstamo y la disminución de ejemplares disponibles deben realizarse de forma atómica. Devuelve `201 Created` si tiene éxito. |
| GET | `/loans` | admin, librarian | Lista: `from`, `to`, `status`, `member_id`, `page`, `size` |
| GET | `/loans/{id}` | admin, librarian | Detalle con líneas y estado |
| POST | `/loans/{loan_id}/items/{item_id}/return` | admin, librarian | Devuelve una línea |
| GET | `/loans/overdue` | admin, librarian | Líneas vencidas |

| POST | `/loans` | Admin, bibliotecario | Crea un préstamo para un socio. La petición debe incluir `member_id` y entre 1 y 3 elementos en `items`, cada uno con un `book_id` diferente. El servidor registra como responsable al usuario autenticado, establece `loaned_at` en UTC y calcula `due_at` a 14 días. La creación del préstamo y la disminución de ejemplares disponibles deben realizarse de forma atómica. Devuelve `201 Created` si la operación tiene éxito. |

```json
// petición  (1 a 3 elementos, sin libros repetidos)
{ "member_id": "…", "items": [ { "book_id": "…" }, { "book_id": "…" } ] }
// 201
{
  "id": "…", "member_id": "…", "loaned_at": "2026-10-14T14:15:00Z", "status": "open",
  "items": [
    { "id": "…", "book_id": "…", "book_title": "Cien años de soledad", "isbn": "9780307474728",
      "category_name": "Novela", "due_at": "2026-10-28T14:15:00Z", "returned_at": null }
  ]
}
```

**Estado del préstamo:** el campo `status` se calcula en cada consulta y no se almacena en la base de datos.

- `open`: hay al menos una línea sin devolver y ninguna de las líneas pendientes está vencida.
- `overdue`: al menos una línea sin devolver tiene `due_at` anterior a la fecha y hora actual.
- `closed`: todas las líneas del préstamo han sido devueltas.

Una línea solo se considera vencida si no ha sido devuelta y su `due_at` es anterior al momento actual. Todas las fechas y horas se manejan en UTC.

| POST | `/loans/{loan_id}/items/{item_id}/return` | Admin, bibliotecario | Registra la devolución de un ejemplar de un préstamo. Establece `returned_at` en UTC e incrementa `available_copies` exactamente una vez, dentro de una operación atómica. Si el ejemplar ya fue devuelto, responde `409 Conflict` con el código `ITEM_ALREADY_RETURNED`. La cantidad disponible nunca puede superar `total_copies`. Devuelve `200 OK` con los datos actualizados del elemento devuelto. |

`GET /loans/overdue` devuelve en `items` los préstamos vencidos con los siguientes campos por elemento: `loan_id`, `item_id`, `book_title`, `member_name`, `due_at` y `days_late`. Los resultados se ordenan por `days_late` descendente, del mayor atraso al menor. En este listado operativo sí se muestra el nombre del socio (`member_name`); **el reporte de préstamos más frecuentes del §3.7 no debe incluir información del socio ni del bibliotecario**.

### 3.7 Reporte

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/reports/most-borrowed` | Admin | `from` y `to` son obligatorios y representan fechas inclusivas. Agrupa los resultados por libro y conserva el título y la categoría registrados al momento del préstamo. Ordena por `times_borrowed` descendente y, en caso de empate, por `book_title` ascendente. Si `from` es posterior a `to`, devuelve `422 Unprocessable Entity` con el código `INVALID_DATE_RANGE`. Si no hay préstamos en el período, devuelve una lista vacía. No incluye nombres de socios ni bibliotecarios. |

```json
{
  "from": "2026-10-01", "to": "2026-10-31",
  "rows": [
    { "book_id": "…", "book_title": "Cien años de soledad", "category_name": "Novela", "times_borrowed": 12 }
  ]
}
```

Una fila por `book_id + book_title + category_name` congelados. Orden: `times_borrowed` descendente, luego título. Sin socios ni bibliotecarios.

## 4. Seguridad

| Aspecto | Regla |
|---|---|
| Token | JWT firmado, 8 horas, con `sub` y `role`. Sin datos personales dentro |
| Contraseñas | Solo se guarda el hash; el hash nunca sale en ninguna respuesta |
| Autorización | Se evalúa en cada endpoint por rol; un 403 no revela si el recurso existe |
| CORS | Solo el origen del frontend configurado por entorno |
| Subida de portada | Solo `image/jpeg` y `image/png`, máx. 2 MB; el tipo se valida por contenido, no por extensión |
