# ADR-006 · Concurrencia con actualización condicionada en lugar de testigo de versión

- **Estado:** aceptada
- **Contexto:** ADR-002 pide control optimista para que dos bibliotecarios no presten el último ejemplar a la vez (RNF-2) y menciona un testigo de versión en `book`. Pero `data-model.md` no lista esa columna, y por el artículo X gana el modelo de datos.
- **Decisión:** descontar el ejemplar con una actualización condicionada a que queden disponibles, dentro de la misma transacción que crea el préstamo. Si la actualización no afecta ninguna fila, el servicio responde 409 `BOOK_NOT_AVAILABLE` y no se registra nada. El CHECK `ck_book_available_range` queda como última barrera.
- **Alternativas descartadas:** agregar una columna de versión (contradice `data-model.md`); bloqueo pesimista de la fila (serializa toda la biblioteca por un caso raro, como ya descartó ADR-002).
- **Consecuencias:** mismo resultado observable que ADR-002 sin cambiar el modelo. Si el CHECK llega a saltar, algo escribió fuera del servicio y es un defecto.
- **Cómo se verifica:** prueba con dos peticiones simultáneas sobre un libro con un ejemplar: exactamente una recibe 201.