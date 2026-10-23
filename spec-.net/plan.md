# Plan técnico — Manager Library (.NET)

Cómo se construye lo que dicen [`spec.md`](spec.md) y [`constitution.md`](constitution.md). El esquema **no** se describe aquí: vive en
[`data-model.md`](data-model.md). La forma del sistema, en [`architecture.md`](architecture.md).

## 1. Decisiones técnicas

| # | Decisión | Elección (.NET) | Por qué |
|---|---|---|---|
| D-01 | Lenguaje y runtime | .NET 8 / C# 12 | Stack del spec de referencia |
| D-02 | Framework HTTP | ASP.NET Core (Minimal API) | Contrato en [`api-contract.md`](api-contract.md) |
| D-03 | Persistencia | EF Core 8 + migraciones de EF sobre PostgreSQL 16 | El esquema es de las migraciones (ADR-001) |
| D-04 | Concurrencia | Control de concurrencia por libro y restricción `CHECK` de disponibilidad ([ADR-002](adr/adr-002-concurrencia-optimista.md)) | Garantizar que dos préstamos simultáneos del último ejemplar no reduzcan las existencias por debajo de cero (RNF-2). |
| D-05 | Fechas | Instantes UTC (`timestamptz`); el reloj es un puerto (`Clock`) | Probar vencimientos sin esperar 14 días |
| D-06 | Reporte | Una consulta agregada por un puerto de lectura ([ADR-004](adr/adr-004-reporte-agregado-y-congelado.md)) | Estable y barato |
| D-07 | Objetos de valor | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange`: objetos del dominio, sin tablas propias | Representan valores del dominio y no tienen identidad propia. |
| D-08 | Portadas | Binario en almacenamiento externo; disco en local y S3 en producción si se configura ese adaptador. La base guarda solo `cover_key`. | La base almacena una clave opaca, nunca una ruta ni el contenido binario. |
| D-09 | Contraseñas | BCrypt (factor 12) detrás del puerto `PasswordHasher`; el dominio nunca ve la clave en claro | Artículo IX |
| D-10 | Arranque | La aplicación crea el admin inicial desde `ADMIN_USERNAME` / `ADMIN_PASSWORD` | La base no puede producir el hash |
| D-11 | Autenticación | JWT HS256, 8 horas, clave en `JWT_SECRET` | Simple y sin estado |
| D-12 | Pruebas | xUnit, con fakes para los puertos y una base real solo en pruebas de integración | Pirámide de pruebas |

## 2. Configuración por entorno

| Variable | Obligatoria | Ejemplo |
|---|---|---|
| `DATABASE_URL` | sí | `postgresql://library:library@db:5432/library` |
| `JWT_SECRET` | sí | cadena larga aleatoria |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | sí | definidos en `.env`, nunca en el repo |
| `COVERS_DIR` | no | `./var/covers` |
| `CORS_ORIGIN` | sí | `http://localhost:5173` |

## 3. Estrategia de pruebas

| Nivel | Qué prueba | Contra qué |
|---|---|---|
| Dominio | Invariantes: máximo 3, vencimiento a 14 días, disponibles en rango | Nada: objetos puros |
| Aplicación | Casos de uso: prestar, devolver, reporte | Fakes de los puertos |
| Integración | Repositorios y consulta del reporte | PostgreSQL real |
| Contrato | Cuerpo de error único y códigos de estado | La API levantada |
| Arquitectura | Artículos I, II y III | NetArchTest |
| Concurrencia | Dos préstamos del último ejemplar | PostgreSQL real |

## 4. Orden de construcción


Esqueleto y salud → migraciones iniciales con semilla → dominio → casos de uso con fakes → adaptadores de persistencia y seguridad → API → composición y arranque → pruebas de contrato y concurrencia.

El detalle y la evidencia verificable de cada tarea están en [`tasks.md`](tasks.md).
