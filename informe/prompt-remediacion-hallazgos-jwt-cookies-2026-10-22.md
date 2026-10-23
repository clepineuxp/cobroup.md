# Prompt de remediación estricta — JWT, cookies y sesión

Actúa como ingeniero principal de seguridad y .NET/Angular responsable de corregir los hallazgos de la auditoría `cobroup.md/informe/auditoria-backend-csharp-2026-10-22-v4.html`.

## Contexto y decisión inmodificable

La decisión arquitectónica ya está tomada: **COBROUP usa exclusivamente access y refresh tokens JWT con firma asimétrica. No se implementará OIDC, OpenIddict, Authorization Code ni PKCE.** `MS.Cobroup.BackendAdmin` es la única autoridad emisora y custodio de la clave privada; `MS.Cobroup.Backend` solo valida access JWT mediante JWKS.

Repositorios involucrados:

- `D:\Proyectos\cobroup\MS.Cobroup.BackendAdmin`
- `D:\Proyectos\cobroup\MS.Cobroup.Backend`
- `D:\Proyectos\cobroup\MF.cobroup-frontend-admin`
- `D:\Proyectos\cobroup\MF.cobroup-frontend`
- `D:\Proyectos\cobroup\cobroup.environments`
- Documentación y decisión: `D:\Proyectos\cobroup\cobroup.md`

Hay cambios locales sin commit. Antes de modificar, inspecciona `git status`, `git diff`, los commits recientes y todos los `AGENTS.md` aplicables. Conserva el trabajo existente, no reviertas cambios ajenos y **no hagas commit ni push**. Si una regla histórica menciona OIDC/OpenIddict/PKCE, debe alinearse con la decisión JWT vigente, sin reintroducir esos componentes.

## Mandato de ejecución

No hagas correcciones cosméticas ni declares cerrado un hallazgo por compilar, por agregar comentarios o por escribir pruebas que no ejercen el flujo real. Implementa primero el bloque obligatorio de cookies/sesión hasta dejarlo demostrado de extremo a extremo. Luego resuelve los bloqueantes JWT/JWKS. Si no alcanzas a resolver un hallazgo, déjalo expresamente abierto: no reduzcas severidad ni maquilles el resultado.

## Bloque 1 — Cookies, CSRF y ciclo de sesión (obligatorio y no negociable)

Corrige y demuestra el contrato completo siguiente:

1. Debe existir **una sola cookie de refresh**, llamada exactamente `__Host-cobroup-refresh`.
2. El refresh JWT solo puede viajar en esa cookie. Está prohibido devolverlo en JSON, exponerlo a JavaScript o guardarlo en `localStorage`, `sessionStorage`, IndexedDB, URL, cachés, service workers, telemetría o logs.
3. La cookie debe emitirse con `HttpOnly=true`, `Secure=true`, `SameSite=Strict`, `Path=/` y sin atributo `Domain`. Su expiración debe coincidir con la vigencia real del refresh JWT y con el registro server-side.
4. Login, refresh y cambio de organización deben reemplazar correctamente la misma cookie. Logout debe revocar la familia correspondiente y expirar esa cookie usando exactamente el mismo nombre, path y atributos relevantes.
5. No aceptes refresh token desde body, query string, header personalizado ni una cookie alternativa. Ante duplicidad o ambigüedad de cookies, falla de forma segura.
6. `refresh`, `logout` y cambio de organización deben exigir antiforgery real. El frontend debe obtener el token antiforgery y enviar `X-XSRF-TOKEN` con `credentials: 'include'`.
7. El servidor debe responder con un error estable y distinguible para fallo CSRF. El frontend **no puede tratar cualquier HTTP 400 como CSRF**. Solo puede renovar el token y reintentar una vez cuando el error sea inequívocamente antiforgery.
8. Logout no puede simular éxito remoto: si el antiforgery expiró, renueva y reintenta una vez. Siempre limpia el access token en memoria, pero si no se pudo revocar la familia en servidor debe registrar/mostrar un estado seguro y explícito, sin conservar secretos ni afirmar que la sesión remota fue cerrada.
9. Ante `401`, usa un único refresh en vuelo (`single-flight`) y como máximo un reintento de la petición original. Si el refresh falla, limpia el estado en memoria y exige autenticación nueva. Evita bucles y tormentas de refresh.
10. El access JWT vive únicamente en memoria. Después de recargar la aplicación, la sesión solo puede restaurarse mediante el endpoint de refresh y la cookie HttpOnly.
11. CORS con credenciales debe permitir únicamente los orígenes configurados; queda prohibido `AllowAnyOrigin` junto con credenciales.
12. Ninguna respuesta, excepción, log o traza debe incluir access JWT, refresh JWT, hash del token, contraseña, clave privada o token antiforgery.

### Pruebas obligatorias para aceptar el bloque de cookies

No basta probar métodos aislados. Agrega pruebas HTTP con `WebApplicationFactory`/`TestServer` para el backend y pruebas de integración del cliente que verifiquen como mínimo:

- `Set-Cookie` de login contiene nombre y atributos exactos; no contiene `Domain`.
- El body de login/refresh no contiene refresh token.
- Refresh rota el JWT, invalida el anterior y emite una sola cookie de reemplazo.
- Reutilizar un refresh rotado revoca toda la familia.
- Dos refresh concurrentes sobre el mismo token producen exactamente un éxito; el otro detecta replay y el estado final de la familia queda revocado según el contrato.
- Refresh, logout y cambio de organización sin CSRF o con CSRF inválido fallan sin mutar sesión.
- Con CSRF válido, las tres operaciones funcionan.
- Logout válido revoca la familia y devuelve una cookie expirada con los atributos correctos.
- El cliente reintenta una sola vez únicamente ante el código explícito de CSRF; un 400 de validación no se repite.
- Si logout recibe fallo CSRF, obtiene token nuevo, reintenta una vez y demuestra revocación remota.
- No hay tokens en ningún almacenamiento accesible a JavaScript, URL, logs ni respuestas indebidas.
- Login → refresh → petición autorizada → cambio de organización → refresh → logout funciona por las rutas reales publicadas.

Si una prueba de PostgreSQL requiere infraestructura no disponible, no debe retornar silenciosamente como `Passed`. Debe fallar en el gate de integración o declararse `Skipped` de forma visible y justificada. En CI debe ejecutarse contra PostgreSQL real.

## Bloque 2 — JWT y JWKS bloqueantes

Resuelve estos hallazgos después del bloque de cookies:

### B-14 — incompatibilidad del tipo de token en runtime

`AccessJwtValidator` no puede depender de un downcast exclusivo a `JwtSecurityToken`, porque ASP.NET Core moderno entrega normalmente `JsonWebToken`. Adapta la validación al tipo real del pipeline o configura conscientemente el pipeline, sin relajar reglas. Agrega una prueba HTTP que pase por `JwtBearer` completo y demuestre:

- access JWT válido aceptado;
- firma, issuer, audience, expiración, `nbf`, algoritmo y `kid` válidos;
- rechazo individual al faltar o ser inválidos `typ=access`, `sub`, `organization_id`, `organization_code`, `role`, `jti`, `iat` o `nbf`;
- rechazo de refresh JWT usado como bearer;
- rechazo de HMAC, `alg=none`, algoritmo distinto, `kid` ausente/desconocido y firma de clave no confiable.

No inventes `organization_code="DEFAULT"`. Si no existe organización activa y cargada, la emisión debe fallar de forma controlada.

### B-15 — estado JWKS fragmentado por DI

La precarga, autenticación y health/readiness deben observar **el mismo estado de caché JWKS**. No uses instancias transitorias con cachés privadas. Separa, si es necesario, un estado singleton seguro para concurrencia y usa `IHttpClientFactory` para transporte. Debe existir:

- precarga antes de anunciar readiness;
- refresh single-flight ante `kid` desconocido;
- last-known-good con límites explícitos;
- backoff compartido y cancelación ante fallos;
- métricas/logs sin secretos;
- readiness `unhealthy` cuando nunca se obtuvo una clave válida o el estado superó el límite permitido;
- pruebas que demuestren que preload, bearer validation y health comparten el mismo estado.

### B-13 — PEM rompe el YAML renderizado

No insertes un PEM multilínea dentro de comillas YAML mediante reemplazo textual. Usa una representación segura, por ejemplo `stringData` con bloque literal correctamente indentado o base64 gestionado de forma consistente. El workflow debe renderizar y validar el manifiesto completo antes de copiarlo o desplegarlo. Ejecuta `kubectl apply --dry-run=client` o validación equivalente sobre dev, QA y producción con un PEM multilínea sintético. Ningún secreto real debe imprimirse.

### B-05 — prueba de concurrencia falsa positiva

La prueba de PostgreSQL no puede capturar excepciones ni retornar silenciosamente. Debe verificar el estado final: un solo consumidor inicial, detección de replay, revocación de familia, ausencia de descendientes activos y unicidad de `TokenHash`. Inclúyela como gate real de CI.

## Bloque 3 — rutas y persistencia

Después de los bloques anteriores:

- El frontend operativo debe consumir la ruta canónica `/api/operations/v1/...`; elimina el alias `/api/v1/session` solo después de comprobar consumidores o documenta una ventana de compatibilidad explícita.
- Prueba las rutas a través del ingress/proxy real, no solo llamando controladores directamente.
- Retira o archiva fuera del flujo ejecutable `scripts/validate-phase4-integration.ps1` mientras invoque `/connect/authorize` o `/connect/token`; reemplázalo por el flujo JWT real.
- Resuelve B-09: las evidencias no pueden depender del disco local de un pod. Implementa almacenamiento durable compartido o un adaptador externo; configura dev/QA/prod y demuestra persistencia entre reinicios/réplicas. No declares cerrado este punto con `LocalFileStorage` en producción.

## Hallazgos que deben conservarse visibles si no se abordan

No pierdas ni ocultes: I-04 rate limiting por sujeto y auditoría de abuso; I-06 zona horaria por tenant; I-07 analítica histórica; I-08 proxies confiables exactos; I-09 validación integral de placeholders; I-10 imágenes inmutables; I-11 migración previa y readiness de esquema; I-13 outbox/inbox; I-16 backoff JWKS; M-02 API interna de evidencias; M-03 alias de ruta.

## Gates mínimos de verificación

Ejecuta y reporta resultados reales:

```text
dotnet build D:\Proyectos\cobroup\MS.Cobroup.Backend\MS.Cobroup.Backend.slnx -c Release --no-incremental
dotnet test  D:\Proyectos\cobroup\MS.Cobroup.Backend\MS.Cobroup.Backend.slnx -c Release
dotnet build D:\Proyectos\cobroup\MS.Cobroup.BackendAdmin\MS.Cobroup.BackendAdmin.slnx -c Release --no-incremental
dotnet test  D:\Proyectos\cobroup\MS.Cobroup.BackendAdmin\MS.Cobroup.BackendAdmin.slnx -c Release
npm test -- --runInBand
npm run build
```

Ejecuta los comandos npm en ambos frontends. Además, ejecuta auditoría de dependencias, comprobación de migraciones pendientes, pruebas PostgreSQL reales, render/validación de manifiestos y escaneo textual de tokens/OIDC legado. Un build verde no sustituye las pruebas de seguridad.

## Formato obligatorio de entrega

Entrega al finalizar:

1. Tabla `hallazgo → archivos modificados → prueba que lo demuestra → estado`.
2. Lista exacta de comandos ejecutados con resultado y número de pruebas.
3. `git diff --stat` y resumen de cambios por repositorio.
4. Evidencia de headers `Set-Cookie` sanitizada y de las respuestas CSRF, sin mostrar tokens.
5. Hallazgos cerrados, parciales y abiertos. Usa `cerrado` solo si existe evidencia ejecutada.
6. Riesgos o bloqueos restantes.
7. Confirmación expresa de que no se introdujo OIDC/OpenIddict/PKCE, no se guardaron tokens en Web Storage, no se imprimieron secretos y no se hizo commit ni push.

No finalices mientras el bloque de cookies tenga pruebas omitidas, falsos positivos o comportamiento ambiguo. Si encuentras una decisión incompatible con este contrato, detén esa parte, documenta la contradicción y conserva la alternativa más segura sin debilitar los controles.
