
# Constitución — Manager Library (.NET)

Principios innegociables del proyecto. No son estilo ni preferencia: cada artículo incluye cómo se verifica, porque un principio que nadie puede comprobar es una intención. Un cambio que viole un artículo se rechaza sin discutir el resto.

---

## Artículo I — El dominio no depende de nada

`src/Library.Domain/` no importa ningún otro módulo del proyecto ni librerías de terceros; solo utiliza la biblioteca base de .NET. No depende de EF Core, ASP.NET ni FluentValidation.

**Por qué:** permite probar las invariantes sin base de datos, sin servidor y sin contenedor, y cambiar de ORM sin modificar las reglas del dominio.

**Verificación:** NetArchTest comprueba que el dominio no dependa de `application`, `adapters` o `bootstrap`. También se verifica que el proyecto no tenga referencias a paquetes de terceros.

## Artículo II — El núcleo declara sus puertos; el mundo los implementa

La capa de aplicación solo conoce el dominio. Todo lo que necesita del exterior (repositorios, reloj, hash de claves, almacenamiento de portadas y generación de tokens) lo expresa mediante un **puerto que ella misma define**. Ningún tipo de framework o de ORM aparece en la firma de un puerto.

**Verificación:** NetArchTest impide que `application` importe `adapters`. Los tests de aplicación corren con fakes y sin referenciar `adapters`.

## Artículo III — Un único punto de composición

Un solo módulo (`src/Library.Bootstrap/Composition/PortBindings.cs`) es donde un puerto se conecta con su adaptador. Fuera de él no se instancia infraestructura.

**Verificación:** ninguna instanciación de una clase de `adapters` fuera de ese módulo; se comprueba con un test de arquitectura.

## Artículo IV — SOLID no es decorativo

| Principio | Aplicación concreta | Verificación |
|---|---|---|
| **SRP** | Un servicio de aplicación corresponde a un caso de uso. `LendBooksService` presta y no hace nada más. | Las responsabilidades independientes se separan en servicios diferentes. |
| **OCP** | Cambiar el almacenamiento de portadas (disco → S3) no modifica el dominio ni la aplicación. | El nuevo adaptador se incorpora mediante la composición. |
| **LSP** | Los fakes de prueba sustituyen al adaptador real sin cambiar las expectativas. | Los mismos casos de prueba pueden ejecutarse contra ambas implementaciones. |
| **ISP** | Puertos pequeños: `Clock`, `PasswordHasher`, `TokenIssuer` y `CoverStorage`. | Ningún implementador debe implementar operaciones que no necesita. |
| **DIP** | El puerto lo define quien lo consume. | Los puertos de los casos de uso viven en la capa de aplicación. |

## Artículo V — El esquema es propiedad de las migraciones

El repositorio `infra` levanta un motor **vacío**. Todo el DDL (tablas, índices, restricciones, extensiones y semilla) vive en las migraciones de EF Core de la aplicación. Ver [ADR-001](adr/adr-001-propiedad-del-esquema.md).

**Verificación:** `infra` no contiene instrucciones `CREATE TABLE`. Las migraciones crean el esquema y aplican las restricciones definidas en `data-model.md`.

## Artículo VI — Las invariantes que la base puede imponer, la base las impone

Una regla que solo vive en código protege a la aplicación, no a los datos. Toda invariante expresable en el motor (`CHECK`, `UNIQUE`, FK) se declara ahí. Lo que todavía no está aplicado se marca **pendiente** en [`data-model.md`](data-model.md); no se promete como implementado.

**Verificación:** las consultas de `data-model.md`, sección 10, coinciden con las restricciones e índices existentes en la base de datos.

## Artículo VII — Lo derivado se calcula, no se almacena

El estado de un préstamo (`open`, `overdue`, `closed`) y los días de retraso se calculan. No hay columna para ellos.

**Verificación:** `data-model.md` no lista ninguna columna `status` ni `days_late`.

## Artículo VIII — Los valores congelados no siguen al catálogo

La línea de préstamo guarda el título, el ISBN y la categoría correspondientes al instante del préstamo. Ningún reporte los lee del catálogo actual.

**Verificación:** una prueba renombra un libro después de prestarlo y comprueba que el reporte conserva el título anterior. Ver [ADR-004](adr/adr-004-reporte-agregado-y-congelado.md).

## Artículo IX — Ningún secreto entra al repositorio ni a un log

Ni contraseñas, ni hashes precalculados, ni claves de tokens. Todo secreto viene del entorno. El `.env` real está en `.gitignore`; solo se versiona `.env.example`.

**Verificación:** `gitleaks` o una búsqueda de patrones en CI detecta posibles secretos. Ninguna respuesta HTTP incluye `password` ni `password_hash`.

## Artículo X — Gana el motor

Si el documento de datos y la base de datos discrepan, se investiga y se corrige el documento o la implementación para restablecer la coherencia. Las restricciones marcadas como pendientes no se consideran implementadas.

## Artículo XI — Idioma

Código, columnas y rutas en inglés. Documentación y mensajes al usuario en español.

## Artículo XII — Nada se da por hecho sin evidencia

Una tarea de [`tasks.md`](tasks.md) está terminada cuando su **evidencia** existe y se puede ejecutar, no cuando «ya está programada».
