# Tareas — Manager Library (Laravel + React)

Adaptación de [`tasks.md` del spec .NET](https://github.com/code-sena/test-manager-library-3239188-docs) al stack del reto. Se conserva la numeración T-01 a T-23 para mantener la trazabilidad con las historias (ver [`SPEC-NOTES.md`](SPEC-NOTES.md)). Cada tarea termina cuando su **evidencia** existe y se puede ejecutar (artículo XII).

**Estado inicial de todas: pendiente.** Hasta ahora lo entregado es diseño y documentación; ninguna tarea de implementación se da por hecha.

## 1. Backend (repo `api`)

| # | Tarea (Laravel) | Depende de | Evidencia | Estado |
|---|---|---|---|---|
| T-01 | Esqueleto del proyecto, prefijo `/api/v1` y ruta de salud | — | `GET /api/v1/health` responde `{"status":"ok"}` | Pendiente |
| T-02 | Migración inicial: seis tablas en el esquema `library`, FK, índices únicos, índice parcial de ISBN y las cinco categorías con sus UUID fijos | T-01 | Las consultas de `data-model.md` §10 devuelven lo documentado; ninguna columna con valor por defecto | Pendiente |
| T-03 | Reglas de entrada reutilizables: ISBN de 13 dígitos, cantidad mayor que 0, rango de fechas con fin ≥ inicio | T-01 | Pruebas unitarias de cada regla | Pendiente |
| T-04 | Modelo `Book`: descontar y devolver ejemplares, baja lógica | T-03 | Pruebas: disponibles nunca < 0 ni > total | Pendiente |
| T-05 | Modelo `Loan` y `LoanItem`: agregar línea, devolver línea, estado calculado, vencimiento a 14 días, máximo 3 | T-04 | Pruebas con reloj fijo: `open`, `overdue`, `closed` | Pendiente |
| T-06 | Modelos `Member` y `User`: normalización del nombre de usuario, rol válido | T-03 | `"Ana "` queda `ana`; rol `admin` pedido por la API rechazado | Pendiente |
| T-07 | Servicios de autenticación: inicio de sesión y alta de bibliotecarios | T-06 | HU-01 y HU-02 pasan en pruebas de funcionalidad | Pendiente |
| T-08 | Servicios de libros: crear, buscar, editar, baja lógica, portada | T-04 | HU-03 y HU-04 pasan en pruebas de funcionalidad | Pendiente |
| T-09 | Servicio de **prestar** con el orden de validaciones de `SPEC-NOTES.md` §3 | T-05, T-08 | HU-06 completa: límites, vencidos, sin ejemplares; un fallo no registra nada | Pendiente |
| T-10 | Servicios de **devolver**, listar préstamos y **vencidos** | T-09 | HU-07, HU-08 y HU-09; devolver dos veces responde `ITEM_ALREADY_RETURNED` | Pendiente |
| T-11 | Servicio de **reporte de más prestados**, consulta agregada sobre la línea de préstamo | T-09 | HU-10; el título viejo se conserva tras renombrar el libro (ADR-004) | Pendiente |
| T-12 | Persistencia de libros, socios y usuarios | T-02, T-08 | Pruebas de funcionalidad contra PostgreSQL real | Pendiente |
| T-13 | Persistencia de préstamos con actualización condicionada del contador de disponibles (ADR-006) | T-12, T-09 | Dos préstamos del último ejemplar: uno 201, otro 409 (RNF-2) | Pendiente |
| T-14 | Seguridad: hash de claves (12 rondas) y token JWT (ADR-005) | T-07 | La clave nunca aparece en logs ni respuestas (RNF-3) | Pendiente |
| T-15 | Portadas en disco local | T-08 | PNG de 2 MB ok; 3 MB responde 422; sin portada, `cover_key` nulo | Pendiente |
| T-16 | Rutas de `api-contract.md`, manejador global del cuerpo de error único, autorización por rol | T-10, T-11, T-14, T-15 | Prueba de contrato recorre todas las rutas y todos los códigos de §2 | Pendiente |
| T-17 | Arranque con admin inicial desde variables de entorno | T-16 | Con la base vacía se puede iniciar sesión como admin | Pendiente |
| T-18 | Restricciones adicionales en la base: `ck_book_total_positive` y `ck_user_role` (incluidas en la migración inicial) | T-02 | Una inserción directa que las viole falla; `data-model.md` §4 coincide | Pendiente |
| T-19 | Pruebas de arquitectura (artículos I a III) | — | **Fuera de alcance** por el ADR-007 | No se hará |

## 2. Entorno, datos y frontend

| # | Tarea | Repo | Depende de | Evidencia | Estado |
|---|---|---|---|---|---|
| T-20 | Un solo comando levanta motor vacío, API y frontend | `infra` | T-17 | Un comando; la base arranca sin tablas hasta correr las migraciones | Pendiente (diseño en su README) |
| T-21 | Sembrador de demostración **por la API** (≥ 40 libros, 10 socios, 15 préstamos, 3 vencidos) | `tool` | T-17 | Tras correrlo, `/loans/overdue` devuelve 3 líneas y el reporte tiene filas | Pendiente (diseño en su README) |
| T-22 | Frontend: login, catálogo, socios, prestar, devolver, vencidos, reporte | `app` | T-16 | Recorrido de punta a punta: login → prestar → devolver → reporte | Pendiente (diseño en su README) |
| T-23 | Sitio público estático de presentación | `page` | — | Se abre sin la API; correcto en móvil y escritorio | Pendiente (diseño en su README) |

## 3. Ruta crítica y prioridad

Si el tiempo no alcanza, este es el orden de valor, siguiendo la definición de terminado del spec (llegar a T-17 con evidencia es el núcleo):

1. T-01 y T-02: sin esto no hay nada que probar.
2. T-14, T-07 y T-16: autenticación y contrato de errores, porque todo lo demás depende de ellos.
3. T-08, T-12 y las reglas de T-04: libros y socios.
4. **T-09, T-13 y T-05**: prestar y concurrencia, la pieza que más puntúa.
5. T-10 y T-11: devolver, vencidos y reporte.
6. T-22, T-20, T-21 y T-23: lo que separa un trabajo bueno de uno sobresaliente.

## 4. Reglas de actualización de este documento

- Una tarea se marca **Hecha** solo con su evidencia escrita en esta tabla y en `SPEC-NOTES.md` §5, en el mismo commit.
- Una tarea a medias se queda **Pendiente** y se anota qué falta.
- Una tarea que no se hará se pasa a **No se hará**, con su razón, para que no parezca un olvido.