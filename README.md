# test-manager-library-3239188-api

Backend en PHP (Laravel) de *Manager Library*. Prueba técnica SDD · Ficha ADSO 3239188.

## Documentación principal

Las decisiones de traducción del spec .NET a Laravel, los supuestos, la trazabilidad HU → endpoint → tarea y el estado con evidencia están en [`SPEC-NOTES.md`](SPEC-NOTES.md). Ese documento es la fuente de verdad de este repositorio.

## Resumen del enfoque

- Se respeta el negocio del spec: historias HU-01 a HU-10, reglas RN-1 a RN-6, contrato HTTP y modelo de datos.
- Se traduce la tecnología: rutas bajo `/api/v1`, migraciones de Laravel para todo el esquema (ADR-001), JWT en lugar de Sanctum, servicios por caso de uso y un manejador global que produce el cuerpo de error único.
- Concurrencia sobre el último ejemplar: actualización condicionada dentro de la transacción del préstamo, con el CHECK de rango como última barrera (ADR-002).
- Baja lógica de libros (ADR-003) y reporte calculado con valores congelados (ADR-004).

## Estado

| Elemento | Estado |
|---|---|
| Notas de traducción, supuestos y trazabilidad | Hecho |
| Implementación de las tareas T-01 a T-18 | Pendiente (ver tabla de `SPEC-NOTES.md`) |

> Este README describe el diseño. El estado real de cada tarea y su evidencia se registran solo en `SPEC-NOTES.md`.