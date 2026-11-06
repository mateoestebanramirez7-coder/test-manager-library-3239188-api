# SPEC-NOTES · Manager Library (Laravel)

Spec de referencia: `test-manager-library-3239188-docs`, carpeta `spec-.net/`. Se eligió esa versión porque describe el mismo producto que `spec-python/` (mismas historias, reglas y endpoints); solo cambian las decisiones de tecnología, que aquí se traducen a Laravel y React.

## 0. Documentos de este repositorio

| Documento | Qué contiene |
|---|---|
| `SPEC-NOTES.md` | Este archivo: traducción, supuestos, trazabilidad y estado con evidencia |
| `plan.md` | Plan técnico del spec .NET traducido a Laravel y React (D-01 a D-14) |
| `tasks.md` | Tareas T-01 a T-23 adaptadas a Laravel y React, con evidencia y estado |
| `adr/adr-005-jwt-en-lugar-de-sanctum.md` | Por qué el token es JWT y no Sanctum |
| `adr/adr-006-concurrencia-con-actualizacion-condicionada.md` | Cómo se cumple RNF-2 sin la columna de versión |
| `adr/adr-007-arquitectura-en-capas-simples.md` | Por qué no se aplica hexagonal estricta |

Los ADR 001 a 004 son los del spec original y se respetan tal cual. La numeración de los propios continúa desde el 005.

## 1. Qué se respeta tal cual

- Historias HU-01 a HU-10, criterios de aceptación y reglas RN-1 a RN-6 (`spec.md`).
- Contrato HTTP completo: rutas bajo `/api/v1`, `snake_case`, paginación `{items, page, size, total}`, cuerpo de error único `{code, message, details}` y códigos de la tabla §2 de `api-contract.md`.
- Modelo de datos de `data-model.md`: esquema `library`, seis tablas en singular, UUID, `timestamptz`, sin `DEFAULT`, sin columnas de auditoría.
- ADR-001 a ADR-004.
- Convención de idioma (artículo XI): código, columnas y rutas en inglés; documentación y mensajes al usuario en español.

## 2. Traducción de tecnología (.NET → Laravel)

| Spec .NET | Decisión en este proyecto | Motivo |
|---|---|---|
| Minimal API | Rutas en `routes/api.php` con prefijo `v1` | Contrato independiente del lenguaje |
| EF Core + migraciones | Migraciones de Laravel; SQL crudo para índice parcial de ISBN y CHECK | ADR-001: el esquema es de las migraciones |
| Un servicio por caso de uso | Una clase de servicio por caso de uso; el controlador solo traduce HTTP | Mantiene SRP sin la ceremonia hexagonal completa (ADR-007) |
| Errores de dominio → cuerpo único | Excepciones propias + handler global que arma `{code, message, details}` | RNF-4 |
| BCrypt factor 12 | `Hash` de Laravel con 12 rondas | D-09 |
| JWT HS256, 8 h, `sub` y `role` | Librería JWT de PHP, clave en `JWT_SECRET` | D-11; no se usa Sanctum porque el contrato pide JWT (ADR-005) |
| Testigo de versión por libro | Actualización condicionada dentro de la transacción + CHECK de rango | El modelo de datos no lista la columna (ADR-006) |
| Puerto `Clock` | Reloj inyectable (o `Carbon::setTestNow` en pruebas) | Probar vencimientos sin esperar 14 días |
| Admin inicial desde entorno | Seeder/comando de arranque con `ADMIN_USERNAME` y `ADMIN_PASSWORD` | D-10 |
| xUnit | PHPUnit | D-12 |
| Frontend de referencia | React con cliente HTTP único y manejo central del cuerpo de error | D-13 |

## 3. Supuestos y desviaciones

1. **Concurrencia (ADR-002 vs `data-model.md`).** El ADR habla de un "testigo de versión", pero el modelo de datos no lista esa columna en `book`. Como el modelo gana (artículo X), no agrego columna. Cumplo RNF-2 con una actualización condicionada a que queden ejemplares disponibles dentro de la transacción del préstamo; si no afecta ninguna fila, respondo 409 `BOOK_NOT_AVAILABLE`. El CHECK `ck_book_available_range` queda como última barrera. Decisión formalizada en el ADR-006.
2. **Arquitectura.** No implemento hexagonal estricta ni pruebas de arquitectura (artículos I a III, T-19). Dejo capas simples: controladores → servicios → modelos. Las reglas de negocio viven en servicios y modelos, no en controladores. Decisión formalizada en el ADR-007.
3. **Autenticación.** Uso JWT y no Sanctum, porque el contrato exige un token firmado de 8 horas con `sub` y `role`. Decisión formalizada en el ADR-005.
4. **Orden de validaciones al prestar** (el spec no lo fija): socio con vencidos (409) → límite de 3 sin devolver contando todos los préstamos (422, con el conteo actual) → libro repetido (422) → ejemplares (409). Un fallo no registra nada.
5. **CHECK de T-18** (`ck_book_total_positive`, `ck_user_role`): se incluyen directamente en la migración inicial, porque en Laravel cuesta lo mismo.
6. **Rango del reporte.** `from` y `to` son fechas inclusivas en UTC: desde las 00:00 de `from` hasta el final del día de `to`.
7. **Vencida** = línea sin devolver con `due_at` menor que ahora. Los días de retraso se calculan en la consulta.
8. **Timestamps de Eloquent desactivados**: el modelo no tiene `created_at`/`updated_at`. Los UUID los genera la aplicación.
9. **Portadas** en disco local (`COVERS_DIR`); la base guarda solo `cover_key`.
10. **Numeración de tareas.** `tasks.md` conserva T-01 a T-23 del spec original para mantener la trazabilidad con las historias; la ruta crítica y la prioridad están en su sección 3.

## 4. Trazabilidad

| HU | Endpoints | Criterios clave | Tarea |
|---|---|---|---|
| HU-01 Login | `POST /auth/login`, `GET /health` | 401 sin revelar qué falló; token 8 h; 401/403 | T-01, T-14, T-16 |
| HU-02 Bibliotecarios | `POST /users` | minúsculas y recorte; 409 repetido; rol `admin` → 422 | T-07,