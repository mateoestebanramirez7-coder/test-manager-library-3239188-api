# Casos de prueba — Manager Library (Laravel + React)

Un caso por cada criterio de aceptación de [`spec.md`](spec.md) y por cada requisito no funcional. Sirve para definir la **evidencia** de cada tarea (artículo XII) y para que cualquiera pueda comprobar el sistema sin abrir el código. Niveles según [`plan.md`](plan.md) §3: **U** unitaria, **F** funcionalidad contra PostgreSQL real, **C** contrato de la API, **K** concurrencia.

> Documento de diseño. Ninguno de estos casos se da por ejecutado hasta que su evidencia se registre en [`SPEC-NOTES.md`](SPEC-NOTES.md) §5 y en [`tasks.md`](tasks.md).

## HU-01 · Iniciar sesión (T-07, T-14, T-16)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-01.1 | Iniciar sesión con usuario y clave válidos | 200 con token, tipo `Bearer`, vigencia 28 800 s y rol; el token trae solo `sub` y `role` | F |
| CA-01.2 | Clave incorrecta con usuario existente, y usuario inexistente | Ambos devuelven 401 `INVALID_CREDENTIALS` con el mismo mensaje; no se distingue qué falló | C |
| CA-01.3 | Llamar a un endpoint protegido sin token, con token inválido y con token vencido | 401 en los tres casos | C |
| CA-01.3 | Un bibliotecario pide el reporte | 403 `FORBIDDEN_ROLE` | C |
| CA-01.3 | Llamar a `/health` y a `/auth/login` sin token | Responden sin exigir token | C |

## HU-02 · Dar de alta bibliotecarios (T-07, T-16)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-02.1 | El admin crea un usuario con nombre, clave y rol `librarian` | 201; la respuesta trae id, nombre y rol, sin clave ni hash | F |
| CA-02.2 | Crear `"Ana "` | Se guarda como `ana` (minúsculas y recortado) | U |
| CA-02.2 | Crear de nuevo `ana` | 409 `USERNAME_TAKEN` | F |
| CA-02.3 | Pedir el rol `admin` por la API | 422 | C |
| CA-02.3 | Revisar respuestas y logs tras crear un usuario | La clave no aparece en ninguno | F |
| CA-01.3 | Un bibliotecario intenta crear usuarios | 403 | C |

## HU-03 · Gestionar libros (T-04, T-08, T-15)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-03.1 | Crear un libro con 4 ejemplares | 201; disponibles = 4 (los calcula el servidor aunque el cliente envíe otro valor) | F |
| CA-03.2 | ISBN con 12 dígitos, o con letras | 422 | U |
| CA-03.2 | Crear dos libros activos con el mismo ISBN | El segundo recibe 409 `ISBN_TAKEN` | F |
| CA-03.2 | Dar de baja un libro y registrar otro con su ISBN | Se permite (el índice solo cubre activos) | F |
| CA-03.3 | Total = 0 | 422 | U |
| CA-03.3 | Subir el total de 4 a 6 con 4 disponibles | Disponibles pasan a 6 | F |
| CA-03.3 | Bajar el total por debajo de lo prestado, de modo que los disponibles quedarían negativos | 422 y no cambia nada | F |
| CA-03.4 | Subir un PNG de 2 MB | 200; `cover_key` queda con una clave opaca | F |
| CA-03.4 | Subir un archivo de 3 MB, o un archivo que no es imagen aunque se llame `.png` | 422 | F |
| CA-03.4 | Libro sin portada | `cover_key` es `null`, nunca cadena vacía; `GET /books/{id}/cover` responde 404 | C |
| CA-03.5 | Dar de baja un libro sin préstamos abiertos | 204; luego `GET /books/{id}` responde 404 | F |
| CA-03.5 | Dar de baja un libro con un préstamo abierto | 409 `BOOK_HAS_OPEN_LOANS` | F |
| CA-03.5 | Consultar un préstamo antiguo cuyo libro fue dado de baja | El préstamo conserva su línea con el título congelado | F |
| CA-01.3 | Un bibliotecario intenta crear, editar o dar de baja un libro | 403 | C |

## HU-04 · Buscar libros (T-08)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-04.1 | Buscar texto parcial que coincide con un título y con un autor | Aparecen ambos | F |
| CA-04.1 | Filtrar por categoría | Solo libros de esa categoría | F |
| CA-04.1 | Buscar con libros dados de baja en el catálogo | Los dados de baja no aparecen | F |
| CA-04.1 | Orden y paginación con `size=50` y con `size=51` | Orden por título; 51 se rechaza o se limita a 50 según el contrato | C |
| CA-04.2 | Cualquier búsqueda | La respuesta trae `items`, `page`, `size` y `total` | C |
| CA-04.3 | Consultar un id inexistente y uno dado de baja | 404 `NOT_FOUND` en ambos | C |

## HU-05 · Gestionar socios (T-12)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-05.1 | El admin y el bibliotecario crean un socio | 201 en ambos casos | F |
| CA-05.1 | Crear dos socios con el mismo documento | 409 `DOCUMENT_TAKEN` | F |
| CA-05.2 | Buscar por parte del nombre y por parte del documento | Ambos encuentran al socio; la respuesta es paginada | F |
| CA-05.3 | Intentar eliminar un socio | La API no ofrece la operación (404 o 405) | C |
| HU-05 | Detalle de un socio con 2 libros sin devolver | `active_loans` = 2 | F |

## HU-06 · Registrar un préstamo (T-05, T-09, T-13)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-06.1 | Préstamo válido con 1 libro, y otro con 3 | 201; el préstamo queda a nombre del usuario autenticado | F |
| CA-06.1 | Préstamo con 0 libros y con 4 libros | 422 en ambos | U |
| CA-06.2 | Préstamo con el mismo libro dos veces | 422 `DUPLICATE_BOOK_IN_LOAN` | U |
| CA-06.3 | Prestar un libro con ejemplares | Los disponibles bajan en 1 | F |
| CA-06.3 | Préstamo de 2 libros donde el segundo no tiene ejemplares | 409 `BOOK_NOT_AVAILABLE` y el primero **no** pierde ejemplar; no se crea préstamo | F |
| CA-06.4 | Socio con 2 libros sin devolver pide 2 más | 422 `MEMBER_LIMIT_EXCEEDED` con el conteo actual (2); no se registra nada | F |
| CA-06.4 | Socio con 2 libros pide 1 más | Se permite (queda en 3) | F |
| CA-06.5 | Socio con una línea vencida pide un libro | 409 `MEMBER_HAS_OVERDUE` | F |
| CA-06.5 | Socio que devolvió su libro vencido pide otro | Se permite | F |
| CA-06.6 | Registrar un préstamo con reloj fijo | `due_at` = `loaned_at` + 14 días exactos | U |
| CA-06.6 | Renombrar el libro y cambiar su categoría tras prestarlo | La línea conserva título, ISBN y categoría del instante del préstamo | F |
| CA-06.7 | Buscar una operación para editar o borrar un préstamo | La API no la ofrece | C |
| Orden | Socio con vencidos que además pide un libro repetido | Gana el 409 de vencidos (orden de `SPEC-NOTES.md` §3, punto 4) | F |

## HU-07 · Registrar una devolución (T-10)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-07.1 | Devolver una línea | 200 con `returned_at` lleno; los disponibles suben en 1; las demás líneas no cambian | F |
| CA-07.2 | Devolver la misma línea dos veces | 409 `ITEM_ALREADY_RETURNED`; los disponibles no suben una segunda vez | F |
| CA-07.3 | Provocar una devolución que superaría el total | El CHECK de rango de la base lo impide | F |
| CA-07.4 | Devolver todas las líneas de un préstamo | El préstamo figura como `closed` | F |
| HU-07 | Devolver una línea con un `item_id` que no pertenece al `loan_id` | 404 | C |

## HU-08 · Ver y listar préstamos (T-10)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-08.1 | Consultar un préstamo | Trae sus líneas y estado calculado | F |
| CA-08.1 | Préstamo con líneas sin devolver y ninguna vencida; con una vencida; con todas devueltas | `open`, `overdue` y `closed` respectivamente | U |
| CA-08.2 | Listar por rango de fechas, por estado y por socio | Filtra correctamente; orden por fecha descendente; paginado | F |
| CA-08.3 | Revisar el modelo de datos | No existe columna de estado ni de días de retraso | F |

## HU-09 · Libros vencidos (T-10)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-09.1 | Hay 3 líneas vencidas y 2 vigentes | La lista trae solo las 3, con libro, socio y días de retraso | F |
| CA-09.2 | Líneas con 10, 3 y 7 días de retraso | Orden 10, 7, 3 | F |
| CA-09.2 | Un usuario sin sesión intenta consultar | 401; admin y bibliotecario sí pueden | C |
| Ruta | Llamar a `/loans/overdue` | No se confunde con `/loans/{id}` | C |

## HU-10 · Reporte de más prestados (T-11)

| CA | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| CA-10.1 | Préstamos de varios libros en octubre | Una fila por libro y categoría congelada, con `times_borrowed`, de mayor a menor y luego por título | F |
| CA-10.1 | Un libro cambia de categoría dentro del rango | Aparece en **dos** filas | F |
| CA-10.2 | `to` anterior a `from` | 422 `INVALID_DATE_RANGE` | C |
| CA-10.2 | Rango sin préstamos | 200 con lista vacía | F |
| CA-10.2 | `from` y `to` iguales | Incluye los préstamos de ese día completo (rango inclusivo, supuesto 6) | F |
| CA-10.3 | Revisar el cuerpo del reporte | Sin socios ni bibliotecarios | C |
| CA-10.4 | Prestar, renombrar el libro y pedir el reporte del mismo rango | El título del reporte es el viejo | F |
| CA-10.5 | Un bibliotecario pide el reporte | 403 | C |

## Requisitos no funcionales

| RNF | Caso | Resultado esperado | Nivel |
|---|---|---|---|
| RNF-1 | Buscar con 5 000 libros sembrados por `tool` | Responde en menos de 300 ms | Carga |
| RNF-2 | Dos peticiones simultáneas por el último ejemplar | Exactamente una recibe 201 y la otra 409 `BOOK_NOT_AVAILABLE`; disponibles = 0, nunca negativo | K |
| RNF-3 | Revisar logs y todas las respuestas tras crear usuarios e iniciar sesión | No aparecen `password` ni `password_hash` | F |
| RNF-4 | Provocar un error por cada código de `api-contract.md` §2 | Todos usan el cuerpo único `{code, message, details}`; ninguno trae trazas, SQL ni rutas | C |
| RNF-5 | Levantar el entorno con `infra` en una máquina limpia | Un solo comando; la base arranca vacía | Manual |

## Recorrido de punta a punta del frontend (T-22)

| Paso | Resultado esperado |
|---|---|
| Iniciar sesión como bibliotecario | Entra y el menú no muestra lo exclusivo del admin |
| Buscar un libro y registrar un préstamo | El catálogo refleja un ejemplar menos |
| Intentar prestar el último ejemplar ya agotado | Aviso en español del 409, sin romper la pantalla |
| Devolver una línea | Los disponibles vuelven a subir |
| Iniciar sesión como admin y abrir el reporte | Muestra las filas del periodo, sin socios |
| Dejar vencer el token | La app vuelve al login |

## Cobertura

Los 10 historias (HU-01 a HU-10), sus criterios CA-01.1 a CA-10.5 y los 5 requisitos no funcionales tienen al menos un caso. Si algún criterio queda sin ejecutar, se declara en `SPEC-NOTES.md` §6.