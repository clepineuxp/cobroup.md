# Prompt para Gemini — Corrección integral de hallazgos COBROUP

Actúa como **ingeniero principal de seguridad y arquitectura**, responsable de implementar las correcciones, no como auditor. Trabaja directamente sobre el workspace COBROUP y corrige los hallazgos con evidencia reproducible. No basta con compilar ni con ocultar síntomas.

## 1. Workspace y fuentes obligatorias

Repositorios:

- `D:\Proyectos\cobroup\MS.Cobroup.Backend`
- `D:\Proyectos\cobroup\MS.Cobroup.BackendAdmin`
- `D:\Proyectos\cobroup\MF.cobroup-frontend`
- `D:\Proyectos\cobroup\MF.cobroup-frontend-admin`
- `D:\Proyectos\cobroup\cobroup.environments`

Antes de modificar cualquier archivo, lee completamente y aplica esta jerarquía:

1. `D:\Proyectos\cobroup\cobroup.md\DECISION_AUTENTICACION_JWT.md`
2. `D:\Proyectos\cobroup\cobroup.md\COBROUP.md`
3. `D:\Proyectos\cobroup\cobroup.md\FRAMEWORK_DESARROLLO_CSHARP.md`
4. `D:\Proyectos\cobroup\cobroup.md\FRAMEWORK_DESARROLLO_FRONTEND.md`
5. `D:\Proyectos\cobroup\cobroup.md\informe\auditoria-backend-csharp-2026-10-22-v2.html`
6. `D:\Proyectos\cobroup\cobroup.md\informe\linea-base-auditoria-jwt-2026-10-22.html`
7. Los `AGENTS.md`, README, ADR, contratos y roadmaps de cada repositorio afectado.

La decisión vigente es definitiva: **COBROUP utilizará exclusivamente access y refresh tokens JWT**. No restaures el mecanismo de autenticación anterior ni implementes un modo híbrido.

## 2. Restricciones de trabajo

- Primero ejecuta `git status`, identifica cambios tracked/untracked y revisa los diffs existentes en cada repositorio.
- El árbol contiene cambios sin commit. Presérvalos e intégralos; no uses `git reset --hard`, `git checkout --`, `git clean`, `stash`, rebase ni comandos destructivos.
- No hagas commit, push, merge, tag, PR ni despliegue. Deja todos los cambios para revisión humana.
- No modifiques ni reescribas informes históricos de auditoría.
- Actualiza `AGENTS.md` de forma append-only y la documentación técnica únicamente para reflejar lo realmente implementado y probado.
- No incluyas secretos, claves privadas, JWT, cookies o contraseñas en Git, fixtures, snapshots, consola o logs.
- Si una prueba requiere Docker, PostgreSQL, K3s, ingress, object storage o infraestructura no disponible, implementa todo lo verificable, deja el comando exacto pendiente y marca el hallazgo como **NO VERIFICADO**; no lo declares resuelto.
- Respeta la arquitectura hexagonal del framework C#: reglas de dominio/Core sin dependencias de infraestructura, puertos explícitos, adaptadores en Infrastructure, controladores delgados y composición en `Program.cs`.

## 3. Resultado esperado

Corrige todos los bloqueantes B-02 a B-10 y los importantes I-01 a I-14. B-01, en su componente de gobierno, ya fue resuelto mediante la decisión JWT; falta retirar de forma coherente los componentes técnicos heredados.

Trabaja en el orden indicado. No avances al siguiente bloque si las pruebas del bloque actual fallan, salvo que documentes con precisión un bloqueo externo.

### 3.1 Mapa mínimo de cobertura

| ID | Resultado que debe quedar demostrado |
|---|---|
| B-01 | Decisión JWT aplicada y componentes heredados fuera del runtime, sin borrar evidencia histórica. |
| B-02 | Firma asimétrica, sin secreto compartido ni fallback; claves compatibles entre emisor y validadores. |
| B-03 | Issuer y audience estrictos con pruebas negativas. |
| B-04 | Refresh solo en cookie segura y únicamente hash server-side. |
| B-05 | Rotación atómica, replay familiar y cambio de tenant seguro. |
| B-06 | Rutas frontend/API/ingress compatibles y probadas externamente. |
| B-07 | Modelo, migración y snapshot sin cambios pendientes. |
| B-08 | Evidencia creada solo desde un archivo real validado. |
| B-09 | Descarga autorizada por ruta y almacenamiento durable compartido. |
| B-10 | Plazo originado exactamente igual al aprobado. |
| I-01 | Suites frontend totalmente verdes. |
| I-02 | Cobertura JWT, HTTP, replay, concurrencia y tenant. |
| I-03 | Organización solicitada no autorizada rechazada explícitamente. |
| I-04 | Rate limit, telemetría y protección de login/refresh/logout. |
| I-05 | Upload streaming y compensación storage/DB. |
| I-06 | Zona IANA por tenant e intervalos diarios correctos. |
| I-07 | Analítica histórica basada en eventos/snapshots. |
| I-08 | Forwarded headers limitados a proxies/redes conocidos. |
| I-09 | Workflows fail-fast y despliegue acoplado a revisión inmutable. |
| I-10 | Imágenes inmutables y pull secret estable de mínimo privilegio. |
| I-11 | Job de migración y readiness consciente de versión de esquema. |
| I-12 | Correlation ID propagado extremo a extremo. |
| I-13 | Outbox/inbox e idempotencia para decisión remota/auditoría. |
| I-14 | Un único controlador ingress documentado y validado en clúster limpio. |

## 4. Bloque A — Autoridad JWT asimétrica y eliminación del legado

### A.1 Retirar componentes incompatibles

En `MS.Cobroup.BackendAdmin`:

- Elimina del runtime controladores, seeders, paquetes, servicios y configuración OpenIddict que ya no tengan función.
- Elimina endpoints `/connect/*`, discovery anterior y flujos de callback que no pertenezcan al contrato JWT actual.
- No edites migraciones históricas ya aplicables. Crea una migración nueva y auditable para retirar tablas/índices heredados que ya no se utilicen, preservando datos de usuarios, organizaciones y membresías.
- En ambos frontends elimina rutas, páginas, servicios, estado y código muerto de callback/verifier/redirección del flujo anterior.
- Confirma con búsqueda global que no quedan dependencias runtime ni rutas activas del mecanismo retirado. Las menciones en historial `AGENTS.md` e informes fechados deben conservarse.

### A.2 Firma y claves

- BackendAdmin debe ser el único emisor y único custodio de la clave privada.
- Usa firma asimétrica `RS256` o `ES256`; no uses HMAC ni secretos compartidos.
- Carga la clave privada desde configuración externa/Secret en formato seguro, con permisos mínimos. Nunca incluyas una clave real en Git.
- Fuera de Development, falla durante el arranque si faltan clave privada válida, `kid`, issuer HTTPS o audiences.
- No permitas claves, issuer, audience ni secretos predeterminados. Elimina todo fallback conocido.
- Publica únicamente claves públicas activas y en solapamiento en `GET /api/v1/auth/.well-known/jwks.json`.
- Cada clave debe tener `kid`. Implementa rotación con clave activa y anterior hasta expirar los access tokens emitidos; luego la clave retirada desaparece del JWKS.
- Backend operativo valida mediante claves públicas/JWKS y nunca recibe la clave privada.
- Define caché y refresco seguro de JWKS: ante `kid` desconocido refresca una vez y luego rechaza; no aceptes silenciosamente otra clave o algoritmo.

### A.3 Access JWT

- Vida entre 5 y 15 minutos; nunca superior a 15 sin una decisión adicional.
- Claims obligatorios: `iss`, `aud`, `sub`, `exp`, `iat`, `nbf`, `jti`, `typ=access`, `organization_id`, `organization_code` y `role`.
- Audiencia exacta: `cobroup-api`; issuer HTTPS exacto y diferente por ambiente.
- Backend y BackendAdmin deben validar firma, algoritmo permitido, `kid`, issuer, audience, expiración, `nbf`, tipo y claims obligatorios.
- Elimina `ValidateIssuer=false`, `ValidateAudience=false` y cualquier relajación equivalente.
- Rechaza algoritmos no permitidos, `none`, token refresh usado como access, claims ausentes y tenant/rol inválidos.

### Pruebas obligatorias del bloque A

- Token válido aceptado por Backend y BackendAdmin.
- HTTP 401 para firma, clave, algoritmo, `kid`, issuer, audience, `typ`, expiración o `nbf` incorrectos.
- Ausencia de configuración criptográfica fuera de Development impide iniciar.
- Rotación de clave: tokens no expirados de la clave anterior siguen válidos durante la ventana y dejan de serlo después de retirarla.
- El backend operativo funciona únicamente con material público.

## 5. Bloque B — Refresh JWT, cookie, atomicidad y tenant

### B.1 Contrato del refresh JWT

- Claims obligatorios: `typ=refresh`, `aud=cobroup-refresh`, `sub`, `organization_id`, `jti`, `family_id`, `iat`, `nbf` y `exp`.
- Vida máxima de 30 días y uso único.
- No devuelvas ni aceptes refresh tokens mediante JSON. Elimina `refresh_token` de responses, request DTO y modelos frontend.
- Entrega el refresh exclusivamente en cookie `HttpOnly`, `Secure`, `SameSite=Strict` —o `Lax` solo con justificación documentada—, `Path` mínimo y expiración coherente.
- El access token se devuelve en el cuerpo y se conserva únicamente en memoria del frontend.

### B.2 Persistencia y rotación

- Sustituye la columna que guarda el token completo por hash criptográfico no reversible del token o `jti`; agrega índice único.
- Conserva como mínimo: hash, `jti`/hash, `family_id`, usuario, organización, creación, expiración, uso/revocación, razón y reemplazo.
- La rotación debe ser una transacción atómica con control real de concurrencia en PostgreSQL. Leer, emitir descendiente y revocar el padre no pueden quedar como operaciones independientes.
- Dos refresh simultáneos sobre el mismo token deben producir exactamente un éxito.
- Cualquier reutilización/replay revoca toda la familia y genera evento de seguridad sin registrar el token.
- Refresh revalida usuario, organización y membresía activos antes de emitir.
- Logout revoca como mínimo la familia actual y expira la cookie incluso ante token inválido/ausente.

### B.3 Cambio de organización y login

- El cambio de organización debe revocar de forma atómica la familia anterior antes de emitir tokens del nuevo tenant.
- Ningún refresh del tenant anterior puede volver a emitir tokens.
- Si `OrganizationId` solicitado en login no corresponde a una membresía activa, responde explícitamente 400/403; no selecciones silenciosamente otra organización.
- Login no recibe ni confía en rol, tenant, issuer o claims enviados por el cliente.

### B.4 CSRF, rate limit y telemetría

- Protege refresh, logout y cambio de organización contra CSRF porque usan cookie. Implementa un patrón compatible con las SPA y pruébalo.
- Aplica rate limiting a login, refresh y logout por IP confiable y sujeto/token cuando corresponda.
- Agrega bloqueo progresivo o protección equivalente sin permitir enumeración de usuarios.
- Logs y auditoría solo incluyen identificadores parciales/hasheados, actor, organización, resultado, IP confiable y correlation ID.

### Pruebas obligatorias del bloque B

- Login válido/inválido, usuario inactivo, organización suspendida y membresía inactiva.
- Refresh exitoso rota el token y el padre no puede reutilizarse.
- Prueba concurrente PostgreSQL: dos refresh simultáneos dejan un único descendiente válido.
- Replay revoca toda la familia.
- Cambio de organización invalida la familia anterior.
- Token/cookie ausente o CSRF inválido produce rechazo seguro.
- Base de datos, responses, Web Storage y logs no contienen refresh JWT en claro.

## 6. Bloque C — Frontends y enrutamiento real

En ambos frontends:

- Elimina cualquier uso de `localStorage`, `sessionStorage` o IndexedDB para access o refresh tokens.
- Mantén el access token únicamente en memoria. Al recargar, recupera sesión mediante refresh cookie; si falla, vuelve a login.
- Usa `withCredentials`/credenciales de cookie solo en las llamadas necesarias.
- Obtén y envía el token CSRF según el contrato implementado.
- Implementa un único refresh ante 401, evita tormentas concurrentes con single-flight y no cree loops de reintento.
- Login, `me`, refresh, logout, listado y cambio de organización deben usar los endpoints definidos en `DECISION_AUTENTICACION_JWT.md`.
- El logout limpia estado de memoria aunque falle la red.
- Corrige las dos pruebas del shell que no encuentran el `h1`; no elimines expectativas útiles solo para poner la suite en verde.

En ingress y configuración:

- Conserva un único contrato externo coherente. Para el acceso same-origin `/identity`, crea una regla Ingress-NGINX separada con rewrite explícito para que `/identity/api/v1/auth/...` llegue como `/api/v1/auth/...`; no apliques una anotación de rewrite accidentalmente a rutas `/api` o frontend.
- Alinea `ingressClassName`, anotaciones y documentación con el controlador realmente instalado. No mezcles Traefik y NGINX.
- Configura correctamente dominio, `Secure`, `SameSite`, Path de cookie, forwarded HTTPS y CORS si llegara a existir origen diferente.

Prueba desde ambos hosts PWA, pasando realmente por ingress: login, `me`, refresh, cambio de organización y logout. Una llamada directa al servicio no sustituye esta prueba.

## 7. Bloque D — Migraciones, Kubernetes y despliegue seguro

- Reconciliación completa de modelo, migraciones y snapshot de BackendAdmin.
- `dotnet ef migrations has-pending-model-changes` debe finalizar sin pendientes.
- Prueba migración arriba/abajo en PostgreSQL limpio y migración hacia arriba con datos representativos existentes.
- Genera y revisa script SQL idempotente; no ejecutes migraciones automáticas dentro de cada réplica de aplicación.
- Agrega Job de migración idempotente previo al rollout y haz que readiness compruebe la versión de esquema esperada, no solo `SELECT 1`.
- Actualiza `cobroup.environments` para inyectar la clave privada únicamente en BackendAdmin y la configuración pública necesaria en Backend. Usa valores placeholder, nunca secretos reales.
- Separa claves por dev/qa/prod. Incluye `kid`, issuer y audiences por ambiente.
- Los workflows remotos deben usar `set -euo pipefail`, validar precondiciones y detenerse ante cualquier error.
- Despliega imágenes por digest o SHA inmutable, no por tags mutables.
- El pull secret debe usar una credencial de lectura estable, de mínimo privilegio y rotación documentada; no un token efímero creado durante el job.
- Restringe forwarded headers a CIDR/IP conocidos del ingress; no limpies indiscriminadamente las listas de proxies/redes confiables.
- Propaga correlation ID Backend→BackendAdmin y respuesta→llamante; agrega prueba end-to-end.

## 8. Bloque E — Evidencias de pago

Corrige B-08, B-09 e I-05:

- Elimina el endpoint legado que permite registrar evidencia solo con JSON/metadatos/ruta suministrada por el cliente.
- La evidencia solo puede crearse desde contenido binario realmente recibido y verificado.
- Usa upload streaming con límite aplicado durante lectura; no cargues el archivo completo en memoria antes de validar tamaño.
- Valida tamaño, MIME permitido, extensión coherente y firma/magic bytes. Genera nombre y ubicación server-side.
- Calcula hash del contenido real. Rechaza archivo vacío y metadatos falsificados.
- Si falla persistencia de base de datos después de subir, elimina/compensa el objeto; si falla el almacenamiento, no crees el registro.
- Antes de descargar, valida tenant, rol, ruta y relación con el pago/evidencia. Un usuario de ruta A no puede leer evidencia de ruta B por conocer el ID.
- Sustituye disco local del pod por almacenamiento de objetos privado y compartido compatible con S3/MinIO, detrás de un puerto de infraestructura. Development puede usar un adaptador local explícito; QA/producción deben fallar si no existe configuración durable.
- Usa bucket privado, cifrado en tránsito, credenciales por Secret, nombres no predecibles y descarga autorizada/proxy o URL firmada de vida corta.
- Agrega configuración/manifiestos o plantillas necesarias sin inventar credenciales reales.

Pruebas: archivo vacío, tamaño excedido por streaming, MIME/extensión/firma falsa, compensación por fallo DB, aislamiento cross-route, reinicio y acceso desde otra réplica/adaptador compartido.

## 9. Bloque F — Excepción de crédito

Corrige B-10:

- En `LoanService`, una aprobación de excepción debe coincidir exactamente con monto, tasa, plazo y unidad contractual aprobados.
- Rechaza plazo menor y mayor; solo el plazo exacto puede originar el préstamo.
- No normalices silenciosamente días/semanas/cuotas. Usa la unidad persistida por la aprobación y prueba sus bordes.

## 10. Bloque G — Hallazgos importantes restantes

- **I-01/I-02:** deja todas las suites verdes y agrega pruebas de JWT service, controladores, expiración, replay, concurrencia, tenant y flujos HTTP.
- **I-06:** usa zona horaria IANA configurada por tenant; convierte a UTC en bordes y usa intervalos diarios semiabiertos `[inicio, fin)`. Prueba cambio de día y DST en una zona que lo aplique.
- **I-07:** la analítica con fecha de corte no puede calcularse con saldos actuales. Implementa eventos/snapshots/proyección histórica consistente y pruebas retroactivas antes/después de pagos y cambios.
- **I-09/I-10/I-11/I-14:** aplica los controles CI/CD, imagen inmutable, migración, readiness e ingress definidos arriba. Valida Docker y un clúster limpio si están disponibles.
- **I-13:** usa outbox/inbox para la decisión remota de excepciones y su auditoría local, con identificador idempotente, reintentos y reconciliación. Prueba fallo entre persistencia y publicación/consumo duplicado.

## 11. Comandos mínimos de validación

Ejecuta desde cada repositorio, adaptando únicamente rutas reales:

```powershell
dotnet restore
dotnet build -c Release --no-restore
dotnet test -c Release --no-build
dotnet ef migrations has-pending-model-changes --project MS.Cobroup.BackendAdmin.Infrastructure --startup-project MS.Cobroup.BackendAdmin.Api

npm ci
npm test -- --watch=false
npm run build

npm audit --omit=dev
dotnet list package --vulnerable --include-transitive
```

Además:

- Renderiza Kustomize de dev, qa y prod.
- Escanea secretos y verifica que no exista ninguna clave privada o fallback conocido en Git.
- Busca usos de `localStorage`, `sessionStorage`, token completo en DB/logs, validaciones issuer/audience desactivadas y claves HMAC.
- Si Docker está disponible, construye las cuatro imágenes y ejecuta smoke tests como usuario no root.
- Si K3s/ingress está disponible, ejecuta el flujo E2E completo por los hosts reales.

No reduzcas cobertura ni elimines pruebas para obtener verde.

## 12. Entrega obligatoria

Al terminar, responde con una matriz:

| Hallazgo | Estado | Archivos modificados | Pruebas ejecutadas | Evidencia/limitación |
|---|---|---|---|---|

Usa solo estos estados:

- `CORREGIDO Y VERIFICADO`
- `IMPLEMENTADO, PENDIENTE DE VERIFICACIÓN EXTERNA`
- `PARCIAL`
- `ABIERTO`

Incluye también:

1. Resumen de arquitectura final.
2. Migraciones creadas y estrategia de rollback.
3. Variables/Secrets requeridos por ambiente, solo nombres y formato, nunca valores.
4. Resultado exacto de builds, pruebas, migraciones, Kustomize, auditorías de dependencias, Docker e ingress.
5. Lista de bloqueos externos reales.
6. `git status --short` final por repositorio.
7. Confirmación explícita de que no hiciste commit ni operación Git remota.

No declares el trabajo terminado mientras exista un bloqueante sin prueba o una suite fallida. No cambies el contrato para adaptar los requisitos al código existente: corrige el código para cumplir `DECISION_AUTENTICACION_JWT.md`.
