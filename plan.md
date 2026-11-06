# Plan técnico — Manager Library (Laravel + React)

Traducción de [`plan.md` del spec .NET](https://github.com/code-sena/test-manager-library-3239188-docs) al stack del reto. El negocio (historias, reglas, contrato, modelo de datos) se respeta tal cual; aquí solo cambia la tecnología. Supuestos y trazabilidad: [`SPEC-NOTES.md`](SPEC-NOTES.md).

## 1. Decisiones técnicas

| # | Decisión | Spec .NET | Elección en el reto | Por qué |
|---|---|---|---|---|
| D-01 | Lenguaje y runtime | .NET 8 / C# 12 | PHP 8.x con Laravel (última versión estable) | Stack del reto |
| D-02 | Framework HTTP | ASP.NET Core Minimal API | Rutas de Laravel bajo el prefijo `/api/v1` | El contrato HTTP no depende del lenguaje |
| D-03 | Persistencia | EF Core 8 + migraciones | Eloquent + migraciones de Laravel sobre PostgreSQL 16; SQL crudo para lo que el constructor de esquemas no expresa | ADR-001: el esquema es de las migraciones |
| D-04 | Concurrencia | Testigo de versión + CHECK de rango | Actualización condicionada del contador de disponibles dentro de la transacción + CHECK de rango | Ver ADR-006 |
| D-05 | Fechas | UTC, `Clock` como puerto | UTC en todo; reloj inyectable en los servicios | Probar vencimientos sin esperar 14 días |
| D-06 | Reporte | Consulta agregada por un puerto de lectura | Una sola consulta agregada sobre `loan_item`, sin unir con el catálogo | ADR-004 |
| D-07 | Objetos de valor | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange` | Validación en las solicitudes de entrada y reglas en los modelos; sin tabla propia | No tienen identidad |
| D-08 | Portadas | Disco local o S3, la base guarda `cover_key` | Disco local en `COVERS_DIR`; la base guarda solo la clave opaca | Alcance del reto |
| D-09 | Contraseñas | BCrypt factor 12 | `Hash` de Laravel con 12 rondas | Artículo IX |
| D-10 | Arranque | Admin inicial desde entorno | Seeder o comando de arranque con `ADMIN_USERNAME` y `ADMIN_PASSWORD` | La base no puede producir el hash |
| D-11 | Autenticación | JWT HS256, 8 horas | JWT HS256, 8 horas, clave en `JWT_SECRET`, con `sub` y `role` | Ver ADR-005 |
| D-12 | Pruebas | xUnit | PHPUnit; pruebas de funcionalidad contra PostgreSQL real | Pirámide simplificada |
| D-13 | Frontend | React (repo `app`) | React con enrutador, cliente HTTP único y manejo central del cuerpo de error | Stack del reto |
| D-14 | Arquitectura | Hexagonal estricta | Capas simples: controladores → servicios → modelos | Ver ADR-007 |

## 2. Configuración por entorno

| Variable | Obligatoria | Uso |
|---|---|---|
| `DATABASE_URL` (o variables de conexión) | sí | Conexión a PostgreSQL 16, esquema `library` |
| `JWT_SECRET` | sí | Firma del token |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | sí | Admin inicial |
| `COVERS_DIR` | no | Carpeta de portadas |
| `CORS_ORIGIN` | sí | Origen del frontend |

El `.env` real no se versiona; solo `.env.example`.

## 3. Estrategia de pruebas

| Nivel | Qué prueba | Contra qué |
|---|---|---|
| Unitarias | Reglas del modelo: máximo 3, vencimiento a 14 días, disponibles en rango, estado calculado | Sin base de datos, con reloj fijo |
| Funcionalidad | Casos de uso: prestar, devolver, vencidos, reporte | PostgreSQL real |
| Contrato | Cuerpo de error único y códigos de estado de `api-contract.md` §2 | La API levantada |
| Concurrencia | Dos préstamos del último ejemplar: uno 201, otro 409 (RNF-2) | PostgreSQL real |

No se incluyen pruebas de arquitectura (T-19); ver ADR-007.

## 4. Orden de construcción

Esqueleto y salud → migración inicial con semilla → autenticación y manejador de errores → libros → socios → prestar → devolver y vencidos → reporte → frontend → `infra`, `tool`, `page`. El detalle, con evidencia, está en [`tasks.md`](tasks.md).

## 5. Estado

Documento de diseño. La implementación está pendiente; el estado real de cada tarea se registra en `SPEC-NOTES.md`.