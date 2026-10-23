# Prompt para Gemini — Migración integral a JWT HS256 por ambiente

Actúa como **arquitecto senior .NET, especialista en seguridad JWT y GitOps/Kubernetes**. Debes corregir la implementación; no hagas una auditoría meramente descriptiva.

## Contexto vinculante

Trabaja sobre estos repositorios locales:

- `MS.Cobroup.BackendAdmin`: autoridad de identidad y único emisor lógico.
- `MS.Cobroup.Backend`: API operativa y validador; nunca emite tokens.
- `cobroup.environments`: manifiestos, render y workflows por ambiente.
- `cobroup.md`: decisiones y documentación normativa.

Lee antes de modificar:

1. `cobroup.md/DECISION_AUTENTICACION_JWT.md`.
2. `cobroup.md/BACKLOG_SEGURIDAD_FIRMA_SIMETRICA.md`.
3. `MS.Cobroup.BackendAdmin/docs/decisions/0007-symmetric-hs256-per-environment.md`.
4. Los `AGENTS.md` aplicables y el framework C# del proyecto.

La decisión está cerrada: **JWT exclusivo con firma simétrica `HS256` y secreto independiente por ambiente**. No se usarán OIDC, OpenIddict, Authorization Code + PKCE, RSA, ES256, certificados, PEM/PFX, JWKS, `alg=none`, claves efímeras, proveedores persistentes RSA ni banderas de acknowledge. No propongas conservarlos como fallback o compatibilidad.

Hay cambios locales sin commit. Presérvalos, inspecciona el estado de cada repositorio antes de editar y no hagas commit, push, merge, rebase, tag ni PR.

## Objetivo obligatorio

Implementa la migración completa de emisión, validación, configuración, GitOps, pruebas y documentación activa a `HS256`, sin debilitar issuer, audiences, claims, sesiones, cookies, CSRF, tenant ni autorización.

## Contrato criptográfico exacto

1. Usa únicamente `SecurityAlgorithms.HmacSha256` / encabezado `alg=HS256`.
2. Cada secreto debe ser generado con CSPRNG, transportado en Base64 y decodificar a **mínimo 32 bytes**. Nunca derives la clave de una frase humana.
3. Variables reales de GitHub por ambiente:
   - `DEV_JWT_SIGNING_KEY_BASE64`
   - `QA_JWT_SIGNING_KEY_BASE64`
   - `PROD_JWT_SIGNING_KEY_BASE64`
4. Configuración Kubernetes/.NET: `Jwt__SigningKeyBase64`.
5. Dev, QA y Prod deben tener valores distintos. Dentro del mismo ambiente, BackendAdmin y Backend reciben exactamente el mismo secreto.
6. El secreto nunca debe aparecer en Git, ConfigMap, `appsettings*.json`, imagen, frontend, logs, excepciones, salida de pruebas, artefactos ni manifiesto renderizado persistente.
7. Ambos backends deben fallar al iniciar si la clave falta, contiene placeholder, no es Base64 válido o decodifica a menos de 32 bytes. No permitas defaults ni generación automática, ni siquiera en Development.
8. La validación debe usar una allowlist exacta de `HS256` y rechazar explícitamente `none`, RSA, ES256 y otros algoritmos HMAC.

## Cambios exigidos en BackendAdmin

- Sustituye el proveedor RSA por una abstracción pequeña y testeable para el conjunto de claves simétricas activo/anterior. Puedes conservar un puerto hexagonal solo si aporta valor real; no mantengas tipos o nombres RSA.
- BackendAdmin es el único emisor lógico de access y refresh. No cambies esa frontera.
- Emite un `kid` de versión no secreto. Nuevos tokens usan solo la clave activa.
- Permite cargar claves anteriores únicamente para validación durante una ventana acotada. Diseña configuración explícita, por ejemplo `Jwt:ActiveKeyId`, `Jwt:SigningKeyBase64` y una colección `Jwt:PreviousValidationKeys`; no uses una lista ambigua sin `kid`.
- Elimina endpoint/controlador/servicio JWKS, clientes internos JWKS, health checks, precarga y pruebas asociadas.
- Elimina `EphemeralRsaSigningKeyProvider`, `PersistentRsa`, `IJwtSigningKeyProvider` si quedó semánticamente RSA, `Jwt:SigningProvider`, `Jwt:AllowEphemeralSigning`, `Jwt:AcknowledgeNonProductionEphemeralRiskOutsideDevelopment` y todo código muerto relacionado.
- Conserva access máximo 15 minutos, refresh máximo 30 días, audiences separadas, claims obligatorios, cookie `HttpOnly`/`Secure`/`SameSite`, protección CSRF, hash server-side, rotación atómica, replay familiar, cambio de organización y rate limiting.

## Cambios exigidos en Backend

- Sustituye la obtención/caché de JWKS por validación local con la clave simétrica del ambiente.
- Aplica la misma validación de configuración y el mismo modelo de `kid` activo/anterior.
- Acepta exclusivamente `typ=access`, audiencia `cobroup-api`, issuer exacto, claims y vigencia actuales.
- No añadas servicios, casos de uso ni endpoints de emisión. Documenta y prueba esta frontera.
- Elimina dependencias, background services, health checks, opciones y pruebas exclusivas de JWKS/RSA.

## Cambios exigidos en `cobroup.environments`

- Actualiza Secrets, Deployments, Kustomize, render y workflows para Dev/QA/Prod.
- Inyecta `Jwt__SigningKeyBase64` en `admin-backend` y `backend`; no lo inyectes en frontends.
- Consume los tres GitHub Environment Secrets indicados, sin valores reales versionados.
- El renderer debe rechazar: secreto ausente, vacío, placeholder, Base64 inválido, menos de 32 bytes, referencias RSA/PEM/JWKS/acknowledge activas y algoritmo distinto de `HS256`.
- Añade controles para detectar reutilización entre ambientes cuando los tres valores estén disponibles en la misma ejecución. Si GitHub aísla secretos por environment, documenta y prueba una validación operativa equivalente sin imprimirlos.
- Garantiza consistencia en múltiples réplicas: todas las réplicas de un ambiente reciben el mismo Secret y no dependen de material generado al iniciar.
- No fuerces `Recreate` por razones criptográficas; conserva o cambia la estrategia solo con justificación operacional y pruebas.
- La salida o logs del render nunca deben imprimir el valor ni una forma reversible del secreto.

## Rotación obligatoria

Implementa y documenta un procedimiento seguro con:

1. Nueva clave y nuevo `kid` instalados primero como validación anterior/futura en ambos backends.
2. Activación coordinada para que BackendAdmin emita con la nueva clave.
3. Ventana máxima de solapamiento igual al TTL de access token más tolerancia de reloj.
4. Retiro verificable de la clave anterior.
5. Tratamiento explícito de refresh tokens: reemisión controlada o revocación según el diseño existente.
6. Rollback y rotación de emergencia por compromiso.

No inventes un endpoint que publique el secreto ni una derivación pública del mismo.

## Pruebas mínimas obligatorias

Añade o adapta pruebas automatizadas que demuestren:

- Emisión y validación `HS256` correctas entre BackendAdmin y Backend.
- Rechazo de `none`, RS256, ES256, HS384/HS512, algoritmo ausente y confusión de algoritmo.
- Rechazo por secreto incorrecto, `kid` desconocido, issuer, audience, `typ`, firma, expiración, `nbf` o claims inválidos.
- Fail-fast por clave ausente, vacía, placeholder, Base64 inválido y 31 bytes; aceptación desde 32 bytes.
- Clave activa emite; clave anterior solo valida; retiro de anterior invalida los tokens correspondientes.
- Un token emitido por una réplica se valida desde otra usando el mismo Secret.
- Backend no expone capacidades de emisión ni JWKS.
- Ningún secreto o JWT completo aparece en logs/excepciones.
- Pruebas GitOps positivas y negativas para los tres ambientes y para ambos backends.
- Los controles de login, refresh concurrente/replay, logout, CSRF, cookie, tenant y roles siguen verdes.

## Limpieza y documentación

- Busca y elimina referencias activas a `RSA`, `RS256`, `ES256`, `JWKS`, `PEM`, `EphemeralRsa`, `PersistentRsa`, `SigningProvider`, `AllowEphemeralSigning` y `AcknowledgeNonProductionEphemeralRiskOutsideDevelopment` en código, configuración, manifiestos, tests y documentación vigente.
- No reescribas informes ni ADR históricos; márcalos como supersedidos si todavía parecen vigentes.
- Actualiza READMEs, contratos, ADR y runbook de rotación con nombres exactos y sin valores secretos.

## Verificación final exigida

Ejecuta, en cada repositorio afectado:

- restauración, build con warnings tratados según la solución y suite completa de tests;
- pruebas GitOps/render positivas y negativas;
- búsqueda de secretos y referencias heredadas activas;
- `git diff --check` y revisión del diff completo.

No des por resuelto un control solo porque compila. Si una prueba no puede ejecutarse, informa el comando exacto, causa y evidencia faltante; no la marques como aprobada.

## Entrega requerida

Entrega un reporte final con:

1. archivos modificados por repositorio;
2. componentes RSA/JWKS/acknowledge eliminados;
3. diseño exacto de configuración y rotación;
4. pruebas creadas/adaptadas y resultados con conteos;
5. búsquedas de residuos y hallazgos pendientes;
6. riesgos restantes, especialmente el impacto de que Backend conozca la clave;
7. estado explícito de cada `SYM-01` a `SYM-10`.

No hagas commit. No declares producción apta mientras algún control bloqueante carezca de evidencia.
