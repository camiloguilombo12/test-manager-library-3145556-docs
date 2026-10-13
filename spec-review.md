# Revisión del spec — Manager Library

Lectura del spec antes de construir. Aquí quedan los errores, contradicciones y huecos encontrados en `spec-python/` y `spec-.net/`,
y la decisión que se toma para cada uno en el reto (React + Laravel).

Los archivos originales del spec **no se modifican**: este documento manda sobre ellos solo en los puntos que lista, y el resto del spec
se respeta tal como está.

- Las líneas citadas son de `spec-python/`. En `spec.md`, `data-model.md`, `api-contract.md` y `adr/` son las mismas en `spec-.net/`,
  porque esos archivos son idénticos en las dos carpetas.
- Gravedad: **alta** (bloquea una tarea o un requisito), **media** (contradicción o hueco que obliga a decidir), **baja** (redacción).

---

## 1. Orden de precedencia usado

`data-model.md:7` dice que si algo contradice al modelo de datos «gana este documento», pero `constitution.md:64-66` (artículo X) y
`data-model.md:225` dicen que si el documento y la base discrepan **gana el motor** y se corrige el documento. Además `spec.md:6` y
`data-model.md:7` dicen que el modelo rige bajo la constitución.

**Decisión:** constitución → spec → plan y arquitectura → modelo de datos y contrato → tareas. Frente al motor gana el motor (artículo X).
Este documento resuelve los conflictos que aparecen a continuación.

---

## 2. Errores que afectan la entrega

### E-01 · El testigo de versión no existe en el modelo de datos — alta

- **Dónde:** `adr/adr-002-concurrencia-optimista.md:6`, `plan.md:13` (D-04) y `tasks.md:19` (T-13) piden un testigo de versión por libro;
  la tabla `book` en `data-model.md:63-75` no tiene esa columna y `tasks.md:8` (T-02) no la crea.
- **Problema:** sin la columna no se puede implementar la concurrencia optimista ni cumplir RNF-2 (`spec.md:106`).
- **Decisión:** se agrega `book.version integer NOT NULL` en la migración inicial. Sin `DEFAULT` (`data-model.md:54`): el dominio la
  inicia en 1 y la sube en cada escritura. Toda escritura del libro es
  `UPDATE … WHERE id = :id AND version = :version_leida`; si afecta cero filas se revierte todo y se responde 409 `BOOK_NOT_AVAILABLE`.

### E-02 · El sembrador no puede crear préstamos vencidos solo con la API — alta

- **Dónde:** `tasks.md:27` (T-21) pide 3 préstamos vencidos; `data-model.md:206` obliga a que el sembrador use solo la API;
  `spec.md:65` (CA-06.6) fija `due_at = loaned_at + 14 días` y el contrato no recibe `loaned_at` (`api-contract.md:141-153`).
- **Problema:** un préstamo creado hoy vence dentro de 14 días. Con esas tres reglas juntas, T-21 es imposible.
- **Decisión:** el puerto `Clock` (`plan.md:14`, D-05) tiene un adaptador de sistema con desfase opcional en días, leído de
  `CLOCK_OFFSET_DAYS` (por defecto `0`). Para la demo se arranca la API con `-20`, el sembrador crea los préstamos que deben quedar
  vencidos, se reinicia con `0` y se crea el resto. El contrato no cambia y no se escribe SQL a mano.
  Los socios con préstamos vencidos ya no pueden pedir más (CA-06.5), así que el sembrador los elige aparte.
- **Descartado:** aceptar `loaned_at` en `POST /loans` (permitiría fechas falsas), un endpoint solo para sembrar, o `UPDATE` directo.

### E-03 · `GET /loans/overdue` choca con `GET /loans/{id}` — alta

- **Dónde:** `api-contract.md:137-139` lista `/loans/{id}` antes que `/loans/overdue`.
- **Problema:** si las rutas se registran en ese orden, el router toma `overdue` como un id y responde 404.
- **Decisión:** `/loans/overdue` se registra antes y `{id}` solo acepta UUID.

### E-04 · Falta la ruta de descarga de portada — media

- **Dónde:** `api-contract.md:121` dice que la portada se pinta con `GET /books/{id}/cover`, pero la tabla de rutas
  (`api-contract.md:100-108`) solo tiene `PUT` y `DELETE`, sin rol para el `GET`.
- **Decisión:** `GET /books/{id}/cover` con rol «autenticado»; devuelve el binario o 404. Como exige token (CA-01.3), el frontend la
  descarga con `fetch` y la muestra desde un `blob`; un `<img src>` directo no envía el encabezado `Authorization`.

### E-05 · No hay código de error para varios estados — media

- **Dónde:** `api-contract.md:44-59`. El único 401 es `INVALID_CREDENTIALS`, pero `api-contract.md:36` también usa 401 para «sin token,
  token inválido o vencido». No hay código para 405 ni para errores no previstos (500).
- **Problema:** RNF-4 (`spec.md:108`) exige el mismo cuerpo para **todo** error.
- **Decisión:** se agregan `UNAUTHENTICATED` (401, sin token o token inválido), `METHOD_NOT_ALLOWED` (405) e `INTERNAL_ERROR`
  (500, mensaje genérico y `details` vacío). Una ruta inexistente responde 404 `NOT_FOUND`.

---

## 3. Contradicciones en el modelo de datos

### C-01 · Invariantes que la base puede imponer y no impone — media

- **Dónde:** el artículo VI (`constitution.md:43-47`) exige declarar en el motor toda invariante expresable, o marcarla **pendiente**.
  `data-model.md:119-141` no menciona varias reglas que sí caben en un `CHECK`:
  ISBN de solo dígitos (`data-model.md:70`), `cover_key` nunca vacío (`data-model.md:74`, CA-03.4) y `username` en minúsculas
  y recortado (`data-model.md:112`, CA-02.2). Además deja `ck_book_total_positive` y `ck_user_role` como pendientes de T-18
  (`data-model.md:134,136`).
- **Decisión:** todas van en la migración inicial (T-02):

| Restricción | Regla |
|---|---|
| `ck_book_available_range` | `available_copies BETWEEN 0 AND total_copies` (ya estaba) |
| `ck_book_total_positive` | `total_copies >= 1` |
| `ck_book_isbn_digits` | `isbn ~ '^[0-9]{13}$'` |
| `ck_book_cover_key_not_empty` | `cover_key <> ''` |
| `ck_user_role` | `role IN ('admin', 'librarian')` |
| `ck_user_username_normalized` | `username = lower(btrim(username))` |

  T-18 (`tasks.md:24`) queda como verificación: un `INSERT` por `psql` que viole cada una debe fallar.

### C-02 · `CASCADE` en `loan_item.loan_id` aunque los préstamos no se borran — media

- **Dónde:** `data-model.md:99,131,151`. RN-5 (`spec.md:98`) dice que los préstamos no se borran.
- **Problema:** con `CASCADE`, un `DELETE` manual de un préstamo borra líneas abiertas sin devolver los ejemplares, y
  `available_copies` queda mal para siempre. Es justo lo que FK-3 evita con `RESTRICT` («barrera de última instancia»).
- **Decisión:** `loan_item.loan_id` con `ON DELETE RESTRICT`.

### C-03 · El máximo de 3 libros por socio puede romperse con concurrencia — media

- **Dónde:** `spec.md:63` (CA-06.4); `data-model.md:137` lo deja como «solo dominio». La versión de E-01 es del libro, no del socio.
- **Problema:** dos préstamos simultáneos del mismo socio pueden pasar los dos la validación y dejarlo con más de 3.
- **Decisión:** dentro de la transacción del préstamo se bloquea **solo la fila del socio** (`SELECT … FOR UPDATE`). No contradice
  ADR-002, que descarta bloquear el libro porque serializaría toda la biblioteca.

### C-04 · La tabla de migraciones del framework rompe la comprobación de §10 — baja

- **Dónde:** `data-model.md:211-213` espera que el esquema `library` no tenga ningún `DEFAULT`. La tabla `migrations` de Laravel tiene
  `id` con `DEFAULT nextval(...)`.
- **Decisión:** la tabla `migrations` vive en `public` (`search_path = library,public`) y la primera migración crea el esquema `library`
  (ADR-001: todo el DDL va en migraciones). Así §10 solo muestra las seis tablas del modelo.

---

## 4. Huecos que obligan a decidir

| # | Tema | Dónde | Decisión |
|---|---|---|---|
| A-01 | Reintento en conflicto de versión | `adr/adr-002-concurrencia-optimista.md:7` («o reintenta una vez») | No se reintenta: 409 directo. Un reintento hace el resultado difícil de predecir y de probar |
| A-02 | Corte del rango del reporte | `api-contract.md:167` («fecha, ambos inclusive») frente a `loaned_at` que es un instante | Días en UTC (RN-6). Entra `loaned_at >= from 00:00Z` y `< (to + 1 día) 00:00Z`. Un `BETWEEN from AND to` perdería el último día |
| A-03 | Rango de `GET /loans` | `api-contract.md:136` | Misma regla que A-02 |
| A-04 | Días de retraso | `spec.md:80`, `api-contract.md:160` | Días completos desde `due_at`, mínimo 1 |
| A-05 | Paginación fuera de rango | `api-contract.md:16`, `spec.md:50` | `page < 1`, `size < 1` o `size > 50`: 422 `VALIDATION_ERROR` |
| A-06 | Búsqueda por texto | `spec.md:50` | Sin distinguir mayúsculas. `%` y `_` se escapan para que no actúen como comodines |
| A-07 | Orden de validación al prestar | `spec.md:60-66`, `architecture.md:69-73` | Datos de entrada (422) → socio con vencidos (409) → límite de 3 (422) → libros inexistentes (404) → sin ejemplares (409). Se responde el primero que falle |
| A-08 | Devolver una línea de otro préstamo | `api-contract.md:138,158` | Si `item_id` no pertenece a `loan_id`: 404. La devolución es `UPDATE … WHERE id = :item AND loan_id = :loan AND returned_at IS NULL`; cero filas nunca suma ejemplares |
| A-09 | Tamaño máximo de portada | `spec.md:46`, `api-contract.md:188`, `tasks.md:21` | 2 MB = 2 097 152 bytes, inclusive. Más grande: 422 `VALIDATION_ERROR` |
| A-10 | Cambiar el ISBN | `api-contract.md:105` (el `PUT` reemplaza título, autor, categoría y total) | El ISBN no se cambia; si viene en el `PUT`: 422 |
| A-11 | Ruta de salud | `api-contract.md:68`, `tasks.md:7` | Todas las rutas son relativas a la base: `GET /api/v1/health` |
| A-12 | Enumeración de usuarios en el login | `spec.md:34` (CA-01.2) | Si el usuario no existe se verifica igual contra un hash ficticio, para que tarde lo mismo. Máximo 5 intentos por minuto por IP |
| A-13 | Credenciales de ejemplo | `plan.md:27` (`library:library`) | Solo en `.env.example`; nunca en archivos versionados con valores reales (artículo IX) |

---

## 5. Inconsistencias menores

| # | Dónde | Problema | Decisión |
|---|---|---|---|
| M-01 | `spec-python/tasks.md:10-11`, `spec-python/architecture.md:54-55,72` | El spec de Python nombra los métodos al estilo .NET (`AddItem`, `Withdraw`) | En PHP se usa camelCase (PSR-1): `addItem`, `withdraw`, `returnItem`, `restock` |
| M-02 | `spec-.net/architecture.md:32` frente a `spec-.net/constitution.md:10,24` | El árbol de carpetas usa `src/Library/`, pero la constitución nombra `src/Library.Domain/` y `src/Library.Bootstrap/` | No afecta al reto; se registra |
| M-03 | `tasks.md:12` (T-06) | Rechazar el rol `admin` «por API» es una regla de aplicación, no de dominio | Se valida en el caso de uso `CreateLibrarianService` |
| M-04 | README de cada repo | Los enlaces apuntan a los repos originales (`code-sena`), no a los forks | No se cambia: es información del enunciado |

---

## 6. Traducción a Laravel y React

El README pide respetar el negocio y traducir la tecnología. Estas costumbres de Laravel chocan con la constitución y se resuelven así:

| Tema del spec | En el reto |
|---|---|
| Framework HTTP (D-02) | Laravel 12, rutas bajo `apiPrefix: 'api/v1'` |
| Persistencia (D-03) | Migraciones de Laravel sobre PostgreSQL 16. MySQL no sirve: no tiene índice único parcial (ADR-003) |
| Dominio sin dependencias (artículo I) | Clases PHP puras en `app/Domain`, con `DateTimeImmutable`. Sin Eloquent, Carbon, facades ni `now()` |
| Puertos (artículo II) | Interfaces en `app/Application/Ports`; servicios en `app/Application/Services` |
| Un punto de composición (artículo III) | Un solo `PortBindingsServiceProvider` |
| Seis tablas sin auditoría (`data-model.md:50,116`) | Tablas en singular, sin `timestamps()`. Se quitan las migraciones por defecto (`users`, `cache`, `jobs`); sesión `array`, caché `file`, colas `sync` |
| UUID v4 (`api-contract.md:12`) | `Str::uuid()` desde la aplicación. No `HasUuids`, que en Laravel 12 genera v7 |
| Contraseñas (D-09) | `Hash` con driver `argon2id` detrás del puerto `PasswordHasher` |
| Token (D-11) | JWT HS256 con `firebase/php-jwt`, 8 horas, `sub` = id del usuario y `role`. Sanctum no sirve: no es JWT y agrega una tabla |
| Cuerpo de error único (RNF-4) | Un manejador en `bootstrap/app.php` para todo error; `APP_DEBUG=false` para no exponer trazas |
| Subida de portada con `PUT` | PHP no procesa `multipart/form-data` en `PUT`: el cliente envía `POST` con `_method=PUT` y Laravel lo enruta como `PUT` |
| Validación de portada | `mimetypes:image/jpeg,image/png` y `max:2048`. La regla `image` no sirve: acepta GIF, WebP y SVG |
| Admin inicial (D-10) | Comando `php artisan app:ensure-admin` al arrancar; lo crea solo si no existe |
| Variables de entorno | `DATABASE_URL` del plan se expresa como `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` |
| Pruebas (D-12) | PHPUnit, con pruebas de arquitectura que revisan los `use` de `app/Domain` y `app/Application` |
| Sembrador (T-21) | Script de Node en `tool` que solo usa la API |

---

## 7. Cómo cambian las tareas

| Tarea | Cambio |
|---|---|
| T-02 | Además de lo pedido: esquema `library`, columna `book.version` (E-01), todos los `CHECK` de C-01 y FK `RESTRICT` de C-02 |
| T-13 | Usa `book.version` y el bloqueo del socio (C-03) |
| T-16 | Incluye los códigos de E-05, el orden de rutas de E-03 y la ruta de E-04 |
| T-18 | Pasa a ser verificación de los `CHECK` creados en T-02 |
| T-21 | Usa el reloj con desfase (E-02) |
