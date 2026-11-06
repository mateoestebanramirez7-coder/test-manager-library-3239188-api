# ADR-005 · JWT en lugar de Sanctum

- **Estado:** aceptada
- **Contexto:** Laravel ofrece Sanctum para autenticar APIs, pero el contrato (`api-contract.md` §4) exige un JWT firmado, de 8 horas, con `sub` y `role`, sin datos personales dentro. `login` debe devolver `access_token`, `token_type` y `expires_in`.
- **Decisión:** usar una librería JWT de PHP con firma HS256 y la clave en `JWT_SECRET`. Un middleware valida el token y el rol en cada endpoint.
- **Alternativas descartadas:** Sanctum (sus tokens son opacos y se guardan en base; no cumplen el contrato ni agregan tablas permitidas por `data-model.md`); sesiones (no aplican a una API sin estado).
- **Consecuencias:** el 401 cubre sin token, token inválido o vencido; el 403 cubre rol insuficiente y no revela si el recurso existe. La clave nunca se versiona.
- **Cómo se verifica:** una petición sin token devuelve 401; un bibliotecario pidiendo `/reports/most-borrowed` recibe 403; el token decodificado solo trae `sub`, `role` y vigencia.