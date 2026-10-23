# Especificación — Manager Library

**Qué se construye y por qué.** Un sistema para que una biblioteca pequeña lleve su catálogo de libros,
sus socios y sus préstamos, y vea qué libros se prestan más.

Rige bajo [`constitution.md`](constitution.md). El modelo de datos vive solo en [`data-model.md`](data-model.md)
y el contrato HTTP solo en [`api-contract.md`](api-contract.md).

---

## 1. Actores

| Actor | Qué hace | Cómo se autentica |
|---|---|---|
| **Administrador** (`admin`) | Mantiene el catálogo de libros y los socios; da de alta bibliotecarios; ve todos los reportes | Usuario y contraseña → token |
| **Bibliotecario** (`librarian`) | Registra préstamos y devoluciones; consulta catálogo, socios y vencidos | Usuario y contraseña → token |
| **Socio** | Persona que pide libros prestados. **No entra al sistema**: es un dato, no un usuario | — |

El rol `admin` lo provisiona el despliegue desde variables de entorno. **Nadie lo otorga en ejecución.**

## 2. Alcance

**Dentro:** catálogo de libros con portada opcional, cinco categorías fijas, socios, préstamos con varias
líneas, devolución por línea, libros vencidos, reporte de más prestados, inicio de sesión con roles.

**Fuera, y no se reabre:** multas y pagos, reservas, renovaciones, correo o SMS, importación masiva,
préstamos entre sedes, atributos de libro más allá de los de §3 (sin editorial, sin año, sin idioma).

## 3. Historias de usuario y criterios de aceptación

### HU-01 · Iniciar sesión
Como usuario quiero autenticarme para operar según mi rol.
- **CA-01.1** Con credenciales válidas se recibe un token con vigencia de 8 horas.
- **CA-01.2** Con credenciales inválidas responde 401 con un mensaje que **no** dice si falló el usuario o la clave.
- **CA-01.3** Todo endpoint salvo el inicio de sesión y `/health` exige token. Sin token: 401. Rol insuficiente: 403.

### HU-02 · Dar de alta bibliotecarios (admin)
- **CA-02.1** El admin crea un usuario con nombre, clave y rol `librarian`.
- **CA-02.2** El nombre de usuario se guarda en minúsculas y recortado; es único. Repetido: 409.
- **CA-02.3** Pedir el rol `admin` por la API responde 422. La clave nunca se devuelve ni se registra en logs.

### HU-03 · Gestionar libros (admin)
- **CA-03.1** Crear libro con título, autor, ISBN, categoría y total de ejemplares. Los disponibles arrancan iguales al total.
- **CA-03.2** El ISBN tiene 13 dígitos y es único entre libros activos. Repetido: 409.
- 
- **CA-03.3** El total de ejemplares debe ser mayor o igual que 1. Al modificarlo, los ejemplares disponibles se ajustan en la misma diferencia. La operación se rechaza con 422 si el nuevo total deja los disponibles por debajo de 0 o resulta incompatible con los ejemplares actualmente prestados.

- **CA-03.4** Subir portada (JPG/PNG, máx. 2 MB) guarda una **clave opaca**; sin portada el campo es `null`, nunca cadena vacía.
- **CA-03.5** Eliminar un libro es **baja lógica**: deja de aparecer en el catálogo pero sigue en los préstamos pasados. Un libro con préstamos abiertos no se da de baja (409).

### HU-04 · Buscar libros
- **CA-04.1** Búsqueda por texto parcial en título o autor, filtro por categoría, solo libros activos, orden por título, paginada (`page`, `size`, máx. 50).
- **CA-04.2** La respuesta incluye el total de resultados.
- **CA-04.3** Consultar un libro por id inexistente o dado de baja: 404.

### HU-05 · Gestionar socios
- **CA-05.1** El admin y el bibliotecario crean socios con nombre y documento. El documento es único: repetido 409.
- **CA-05.2** Se lista y se busca por nombre o documento, paginado.
- **CA-05.3** Los socios no se eliminan.

### HU-06 · Registrar un préstamo
- **CA-06.1** Un préstamo es de **un** socio, lo registra el usuario autenticado y trae de 1 a 3 líneas.
- **CA-06.2** No se repite el mismo libro en un préstamo (422).
- **CA-06.3** Cada línea descuenta **1** ejemplar disponible en la misma operación. Sin ejemplares: 409 y **no se registra nada**.
- **CA-06.4** Un socio no puede tener más de **3 libros sin devolver en total**, contando todas sus líneas de préstamo que todavía no tengan `returned_at`. Si la nueva operación supera el límite, responde 422 y no registra el préstamo.

- **CA-06.5** Un socio con algún libro **vencido** no puede pedir otro (409).
- **CA-06.6** La fecha de vencimiento es `loaned_at + 14 días`. La línea congela título, ISBN y nombre de categoría **en ese instante**.
- **CA-06.7** El préstamo registrado no se edita ni se borra.

### HU-07 · Registrar una devolución
- **CA-07.1** Se devuelve **una línea** a la vez. Pone `returned_at` y suma 1 a los disponibles.
- **CA-07.2** Devolver una línea ya devuelta: 409. **Nunca** suma dos veces.
- **CA-07.3** Los disponibles jamás superan el total (la base lo garantiza).
- **CA-07.4** Cuando todas las líneas están devueltas, el préstamo figura como **cerrado**.

### HU-08 · Ver un préstamo y listar préstamos
- **CA-08.1** Consultar un préstamo trae sus líneas y su estado calculado (`open`, `overdue`, `closed`).
- **CA-08.2** Listar por rango de fechas, estado y socio, orden por fecha descendente, paginado.
- **CA-08.3** El estado **se calcula**, no se almacena.

### HU-09 · Libros vencidos

- **CA-09.1** Lista las líneas de préstamo sin devolver cuyo `due_at` sea anterior al instante UTC actual. Incluye el libro, el socio y los días de retraso, calculados a partir de la diferencia entre el vencimiento y el instante UTC actual.

- **CA-09.2** Ordenadas por más días de retraso primero. Solo `admin` y `librarian`.

### HU-10 · Reporte de libros más prestados
- **CA-10.1** Dado un rango de fechas, el reporte agrupa los préstamos por identificador del libro y nombre de categoría congelado en `loan_item`. Cuenta las líneas de préstamo registradas dentro del rango, ordena los resultados de mayor a menor cantidad y utiliza los valores históricos, no los del catálogo actual.
- **CA-10.2** Fin anterior al inicio: 422. Rango sin préstamos: lista vacía, no error.
- **CA-10.3** El reporte **no** se desglosa por bibliotecario ni por socio.
- **CA-10.4** El reporte de un rango de fechas debe conservar los valores históricos de título, ISBN y categoría registrados en cada línea de préstamo, aunque posteriormente se renombre el libro o cambie su categoría en el catálogo.
- **CA-10.5** Solo `admin`.

## 4. Reglas de negocio

| # | Regla |
|---|---|
| RN-1 | Disponibles entre 0 y total, siempre. |
| RN-2 | Plazo de préstamo: 14 días. Máximo 3 libros sin devolver por socio. |
| RN-3 | Un socio con vencidos no pide más hasta devolverlos. |
| RN-4 | Lo congelado en la línea (título, ISBN, categoría) no sigue al catálogo. |
| RN-5 | Préstamos y devoluciones no se borran. Los libros solo tienen baja lógica. |
| RN-6 | Las fechas son instantes UTC. Mensajes al usuario en español; código y columnas en inglés. |

## 5. Requisitos no funcionales

| # | Requisito | Cómo se mide |
|---|---|---|
| RNF-1 | La búsqueda de libros responde en menos de 300 ms con 5 000 libros | Prueba de carga con el sembrador de `tool` |
| RNF-2 | Dos bibliotecarios prestando el **último ejemplar** a la vez: uno gana, el otro recibe 409 | Prueba concurrente |
| RNF-3 | La contraseña nunca viaja ni se guarda en claro; nunca aparece en logs | Revisión de logs y de respuestas |
| RNF-4 | Todo error HTTP usa el mismo cuerpo (ver contrato) | Prueba de contrato |
| RNF-5 | El sistema levanta con un solo comando y la base de datos se inicializa mediante las migraciones de EF Core, incluyendo el esquema y las cinco categorías fijas. | Instrucciones verificables en el `README` de `infra` y comprobación de las migraciones sobre una base de datos vacía. |

