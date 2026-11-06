# SPEC-NOTES · Manager Library (Laravel)

Spec de referencia: `test-manager-library-3239188-docs`, carpeta `spec-.net/`. Se eligió esa versión porque describe el mismo producto que `spec-python/` (mismas historias, reglas y endpoints); solo cambian las decisiones de tecnología, que aquí se traducen a Laravel y React.

## 0. Documentos de este repositorio

| Documento | Qué contiene |
|---|---|
| `SPEC-NOTES.md` | Este archivo: traducción, supuestos, trazabilidad, estado con evidencia y riesgos |
| `plan.md` | Plan técnico del spec .NET traducido a Laravel y React (D-01 a D-14) |
| `architecture.md` | Capas del backend Laravel, flujo de prestar, errores y diseño del frontend React |
| `tasks.md` | Tareas T-01 a T-23 adaptadas a Laravel y React, con evidencia y estado |
| `test-cases.md` | Un caso de prueba por criterio de aceptación y por requisito no funcional |
| `adr/adr-005-jwt-en-lugar-de-sanctum.md` | Por qué el token es JWT y no Sanctum |
| `adr/adr-006-concurrencia-con-actualizacion-condicionada.md` | Cómo se cumple RNF-2 sin la columna de versión |
| `adr/adr-007-arquitectura-en-capas-simples.md` | Por qué no se aplica hexagonal estricta |
| `spec-laravel-react/README.md` (repo `docs`) | Índice que enlaza estos documentos desde el repo del spec |

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
5. **CHECK de T-18** (`ck_book_total_positive`, `ck_user_role`): `data-model.md` §4 los marca como pendientes de T-18; aquí se incluyen directamente en la migración inicial, porque en Laravel cuesta lo mismo. Al hacerlo, pasan a ser reglas del **motor**.
6. **Rango del reporte.** `from` y `to` son fechas inclusivas en UTC: desde las 00:00 de `from` hasta el final del día de `to`.
7. **Vencida** = línea sin devolver con `due_at` menor que ahora. Los días de retraso se calculan en la consulta.
8. **Timestamps de Eloquent desactivados**: el modelo no tiene `created_at`/`updated_at`. Los UUID los genera la aplicación.
9. **Portadas** en disco local (`COVERS_DIR`); la base guarda solo `cover_key`.
10. **Numeración de tareas.** `tasks.md` conserva T-01 a T-23 del spec original para mantener la trazabilidad con las historias; la ruta crítica y la prioridad están en su sección 3.
11. **Paginación.** Con `size` mayor que 50 se limita a 50 en lugar de rechazar, porque el contrato dice "máximo 50" sin definir un error para ese caso.

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

Los casos de prueba de cada criterio están en [`test-cases.md`](test-cases.md).

## 5. Estado de las tareas y evidencia

Se marca cada tarea solo cuando existe su evidencia (artículo XII). Hasta ahora el trabajo entregado es de **diseño y documentación**: las tareas de implementación siguen pendientes y no se marcan como hechas. El detalle de cada tarea está en [`tasks.md`](tasks.md).

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
| T-19 Pruebas de arquitectura | No se hará | Fuera de alcance por el ADR-007 |
| T-20 `infra` | [ ] | Diseño documentado en su README; implementación pendiente |
| T-21 `tool` | [ ] | Diseño documentado en su README; implementación pendiente |
| T-22 `app` | [ ] | Diseño documentado en su README y en `architecture.md` §9; implementación pendiente |
| T-23 `page` | [ ] | Diseño documentado en su README; implementación pendiente |

## 6. Fuera de alcance de este intento

- Pruebas de arquitectura (T-19). Se declara aquí para que no parezca un olvido; la razón está en el ADR-007.
- Implementación de las tareas T-01 a T-18 y T-20 a T-23: se entrega el diseño completo y su evidencia esperada, pero no el código.

## 7. Riesgos y preguntas abiertas

| # | Riesgo o pregunta | Cómo se resolvió o qué falta |
|---|---|---|
| R-1 | El enunciado pide "resolver el proyecto con el spec" y no aclara si se acepta solo documentación | Se entregó el ciclo SDD completo (traducción, ADR, tareas, casos de prueba). Pendiente confirmar con el instructor |
| R-2 | ADR-002 menciona un testigo de versión que `data-model.md` no incluye | Gana el modelo de datos (artículo X); decisión en el ADR-006 |
| R-3 | El spec no fija el orden de las validaciones al prestar | Supuesto 4; si se revisa distinto, cambia solo el servicio de préstamo |
| R-4 | No se define qué hace la API con `size` mayor que 50 | Supuesto 11: se limita a 50 |
| R-5 | `data-model.md` marca dos CHECK como pendientes de T-18 | Supuesto 5: se adelantan a la migración inicial; habría que actualizar §4 del modelo de datos al implementar |
| R-6 | El reporte puede mostrar un mismo libro en dos filas | Es el comportamiento correcto según ADR-004 y CA-10.1; el frontend debe aceptarlo |
| R-7 | Solo se leyó `spec-.net/`; `spec-python/` no se revisó | El README del repo `docs` confirma que ambos describen el mismo producto; si hay una duda de negocio, se consulta el otro. La carpeta `spec-.net/` no se modificó: la traducción vive en este repo y el repo `docs` solo recibe un índice (`spec-laravel-react/`) |
| R-8 | La ruta de vencidos puede confundirse con la de detalle de préstamo | Registrar `/loans/overdue` antes que `/loans/{id}`; hay un caso en `test-cases.md` |
| R-9 | Sin código, ninguna evidencia ejecutable existe todavía | Cada tarea queda pendiente, con su evidencia esperada en `tasks.md` y `test-cases.md` |