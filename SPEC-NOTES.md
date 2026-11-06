# SPEC-NOTES · Manager Library (Laravel)

Spec de referencia: `test-manager-library-3239188-docs`, carpeta `spec-.net/`. Se eligió esa versión porque describe el mismo producto que `spec-python/` (mismas historias, reglas y endpoints); solo cambian las decisiones de tecnología, que aquí se traducen a Laravel.

## 1. Qué se respeta tal cual

- Historias HU-01 a HU-10, criterios de aceptación y reglas RN-1 a RN-6 (`spec.md`).
- Contrato HTTP completo: rutas bajo `/api/v1`, `snake_case`, paginación `{items, page, size, total}`, cuerpo de error único `{code, message, details}` y códigos de la tabla §2 de `api-contract.md`.
- Modelo de datos de `data-model.md`: esquema `library`, seis tablas en singular, UUID, `timestamptz`, sin `DEFAULT`, sin columnas de auditoría.
- ADR-001 a ADR-004.

## 2. Traducción de tecnología (.NET → Laravel)

| Spec .NET | Decisión en este proyecto | Motivo |
|---|---|---|
| Minimal API | Rutas en `routes/api.php` con prefijo `v1` | Contrato independiente del lenguaje |
| EF Core + migraciones | Migraciones de Laravel; SQL crudo para índice parcial de ISBN y CHECK | ADR-001: el esquema es de las migraciones |
| Un servicio por caso de uso | Una clase de servicio por caso de uso; el controlador solo traduce HTTP | Mantiene SRP sin la ceremonia hexagonal completa |
| Errores de dominio → cuerpo único | Excepciones propias + handler global que arma `{code, message, details}` | RNF-4 |
| BCrypt factor 12 | `Hash` de Laravel con 12 rondas | D-09 |
| JWT HS256, 8 h, `sub` y `role` | Librería JWT de PHP, clave en `JWT_SECRET` | D-11; no se usa Sanctum porque el contrato pide JWT |
| Puerto `Clock` | Reloj inyectable (o `Carbon::setTestNow` en pruebas) | Probar vencimientos sin esperar 14 días |
| Admin inicial desde entorno | Seeder/comando de arranque con `ADMIN_USERNAME` y `ADMIN_PASSWORD` | D-10 |
| xUnit | PHPUnit | D-12 |

## 3. Supuestos y desviaciones

1. **Concurrencia (ADR-002 vs `data-model.md`).** El ADR habla de un "testigo de versión", pero el modelo de datos no lista esa columna en `book`. Como el modelo gana (artículo X), no agrego columna. Cumplo RNF-2 con un `UPDATE` condicionado a `available_copies > 0` dentro de la transacción del préstamo; si no afecta ninguna fila, respondo 409 `BOOK_NOT_AVAILABLE`. El CHECK `ck_book_available_range` queda como última barrera.
2. **Arquitectura.** No implemento hexagonal estricta ni pruebas de arquitectura (artículos I a III, T-19). Dejo capas simples: controladores → servicios → modelos. Las reglas de negocio viven en servicios y modelos, no en controladores.
3. **Orden de validaciones al prestar** (el spec no lo fija): socio con vencidos (409) → límite de 3 sin devolver contando todos los préstamos (422, con el conteo actual) → libro repetido (422) → ejemplares (409). Un fallo no registra nada.
4. **CHECK de T-18** (`ck_book_total_positive`, `ck_user_role`): los incluyo directamente en la migración inicial, porque en Laravel cuesta lo mismo.
5. **Rango del reporte.** `from` y `to` son fechas inclusivas en UTC: desde las 00:00 de `from` hasta el final del día de `to`.
6. **Vencida** = línea sin devolver con `due_at` menor que ahora. Los días de retraso se calculan en la consulta.
7. **Timestamps de Eloquent desactivados**: el modelo no tiene `created_at`/`updated_at`. Los UUID los genera la aplicación.
8. **Portadas** en disco local (`COVERS_DIR`); la base guarda solo `cover_key`.

## 4. Trazabilidad

| HU | Endpoints | Criterios clave | Tarea |
|---|---|---|---|
| HU-01 Login | `POST /auth/login`, `GET /health` | 401 sin revelar qué falló; token 8 h; 401/403 | T-01, T-14, T-16 |
| HU-02 Bibliotecarios | `POST /users` | minúsculas y recorte; 409 repetido; rol `admin` → 422 | T-07, T-16 |
| HU-03 Libros (admin) | `POST/PUT/DELETE /books`, `PUT/DELETE /books/{id}/cover` | ISBN 13 dígitos único entre activos; total ≥ 1; baja lógica; portada ≤ 2 MB | T-08, T-15 |
| HU-04 Buscar | `GET /books`, `GET /books/{id}`, `GET /categories` | texto, categoría, activos, orden por título, `size` ≤ 50, 404 si baja | T-08 |
| HU-05 Socios | `GET/POST /members`, `GET /members/{id}` | documento único; sin borrado; `active_loans` | T-12 |
| HU-06 Prestar | `POST /loans` | 1 a 3 líneas; −1 ejemplar; máx. 3 sin devolver; sin vencidos; 14 días; valores congelados | T-09, T-13 |
| HU-07 Devolver | `POST /loans/{loan_id}/items/{item_id}/return` | +1 ejemplar; 409 si ya devuelta; nunca suma dos veces | T-10 |
| HU-08 Consultar préstamos | `GET /loans`, `GET /loans/{id}` | estado calculado; filtros fecha, estado, socio | T-10 |
| HU-09 Vencidos | `GET /loans/overdue` | más retraso primero; nombre del socio visible | T-10 |
| HU-10 Reporte | `GET /reports/most-borrowed` | agrupa por valores congelados; solo admin; sin socios; rango inválido 422 | T-11 |

## 5. Estado de las tareas y evidencia

Se marca cada tarea solo cuando existe su evidencia (artículo XII).

| Tarea | Estado | Evidencia |
|---|---|---|
| T-01 Esqueleto y `/health` | [ ] | |
| T-02 Migración inicial + semilla de 5 categorías | [ ] | |
| T-14 / T-07 Login, JWT y admin inicial | [ ] | |
| T-08 / T-15 Libros y portadas | [ ] | |
| T-12 Socios | [ ] | |
| T-09 / T-13 Prestar + concurrencia | [ ] | |
| T-10 Devolver, listar, vencidos | [ ] | |
| T-11 Reporte | [ ] | |
| T-16 / T-17 Contrato de errores y arranque | [ ] | |
| T-18 CHECK adicionales | [ ] | |
| T-20 `infra` | [ ] | |
| T-21 `tool` | [ ] | |
| T-22 `app` | [ ] | |
| T-23 `page` | [ ] | |

## 6. Fuera de alcance de este intento

- Pruebas de arquitectura (T-19). Se declara aquí para que no parezca un olvido.