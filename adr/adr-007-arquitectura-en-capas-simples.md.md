# ADR-007 · Capas simples en lugar de arquitectura hexagonal estricta

- **Estado:** aceptada
- **Contexto:** la constitución (artículos I a IV) exige dominio sin dependencias, puertos definidos por la aplicación y un único punto de composición, verificados con NetArchTest. Eso es idiomático en .NET, pero costoso de replicar en Laravel dentro del tiempo de la prueba.
- **Decisión:** separar controladores (solo traducen HTTP), servicios (un caso de uso cada uno) y modelos (invariantes de datos). Las reglas de negocio no viven en controladores. Se mantiene un servicio por caso de uso (SRP) y el cuerpo de error único generado en un solo manejador global.
- **Alternativas descartadas:** hexagonal completa con interfaces para cada repositorio (demasiado costo para el tiempo disponible); lógica en controladores (rompe SRP y dificulta las pruebas).
- **Consecuencias:** se pierde la verificación automática de los artículos I a III (T-19 queda fuera de alcance, declarado en `SPEC-NOTES.md`). Se conservan las invariantes de negocio y el contrato.
- **Cómo se verifica:** ningún controlador contiene reglas de negocio; cada caso de uso tiene su servicio.