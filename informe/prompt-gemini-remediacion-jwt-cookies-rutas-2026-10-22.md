# Prompt para Gemini — Remediación incremental JWT, cookies y rutas COBROUP

Actúa como **ingeniero principal de seguridad .NET/Angular**, responsable de implementar un bloque acotado de correcciones sobre el workspace COBROUP. En esta iteración **no debes intentar cerrar todos los hallazgos**: el objetivo es reducir de manera verificable los hallazgos relacionados con JWT, refresh/cookies, CSRF y rutas externas, dejando el resto intacto para una auditoría posterior.

## 1. Fuentes obligatorias y orden de autoridad

Antes de modificar archivos, lee completamente:

1. `D:\Proyectos\cobroup\cobroup.md\DECISION_AUTENTICACION_JWT.md`
2. `D:\Proyectos\cobroup\cobroup.md\FRAMEWORK_DESARROLLO_CSHARP.md`
3. `D:\Proyectos\cobroup\cobroup.md\FRAMEWORK_DESARROLLO_FRONTEND.md`
4. `D:\Proyectos\cobroup\cobroup.md\COBROUP.md`
5. `D:\Proyectos\cobroup\cobroup.md\informe\auditoria-backend-csharp-2026-10-22-v3.html`
6. Los `AGENTS.md`, README, ADR, contratos y roadmaps de cada repositorio afectado.

La decisión vigente es definitiva: **COBROUP utiliza exclusivamente access y refresh JWT**. No restaures OpenIddict, OIDC, Authorization Code, PKCE, endpoints `/connect/*` ni un modo híbrido.

Repositorios en alcance:

- `D:\Proyectos\cobroup\MS.Cobroup.Backend`
- `D:\Proyectos\cobroup\MS.Cobroup.BackendAdmin`
- `D:\Proyectos\cobroup\MF.cobroup-frontend`
- `D:\Proyectos\cobroup\MF.cobroup-frontend-admin`
- `D:\Proyectos\cobroup\cobroup.environments`

## 2. Restricciones obligatorias

- Ejecuta primero `git status --short`, revisa los diffs y preserva todos los cambios tracked/untracked existentes.
- No uses `git reset --hard`, `git checkout --`, `git clean`, stash, rebase ni operaciones destructivas.
- **No hagas commit, push, merge, tag, PR ni despliegue.** Deja los cambios para reauditoría humana.
- No modifiques ni reescribas informes históricos de auditoría.
- No agregues claves privadas reales, secretos, contraseñas, JWT ni cookies a Git, consola, fixtures, snapshots o logs.
- No debilites pruebas, autorización tenant/rol/recurso, HTTPS, validación criptográfica ni fail-fast para conseguir verde.
- Respeta la arquitectura hexagonal: reglas y contratos en Core/Domain; adaptadores en Infrastructure; controladores delgados; composición en `Program.cs`.
- Si Docker, PostgreSQL real, K3s o ingress no están disponibles, implementa lo comprobable y reporta el punto como `IMPLEMENTADO, PENDIENTE DE VERIFICACIÓN EXTERNA`; no inventes evidencia.

## 3. Alcance exacto de esta iteración

Ataca únicamente estos hallazgos del informe v3:

| ID | Objetivo de esta iteración |
|---|---|
| B-05 | Rotación/replay concurrente de refresh con resultado seguro y demostrable. |
| B-06 | Cookie compatible con navegador e ingress; rutas externas y contexto operativo coherentes. |
| B-11 | Validación estricta del contrato completo del access JWT. |
| B-12 | CSRF efectivo en refresh, logout y cambio de organización. |
| B-13 | Corregir solamente la parte JWT/rutas de GitOps; no abordar storage de evidencias aquí. |
| I-02 | Ampliar pruebas específicas para este bloque. |
| I-15 | Retirar residuos runtime del mecanismo anterior asociados a autenticación. |
| I-16 | Sustituir la resolución JWKS síncrona/frágil. |
| I-17 | Eliminar revocación familiar basada en claims no validados. |
| M-01 | Dejar una única cookie refresh con semántica correcta. |

### Fuera de alcance

No implementes en este bloque:

- almacenamiento durable de evidencias/PVC/S3/MinIO (B-09);
- zona horaria por tenant (I-06);
- analítica histórica (I-07);
- rediseño general de workflows/imágenes (I-09/I-10), salvo sustitución estrictamente necesaria de variables JWT;
- job de migración/readiness de esquema (I-11);
- outbox/inbox (I-13);
- cambios funcionales de préstamos, pagos, agenda, clientes o rutas de negocio;
- refactors cosméticos amplios.

No declares esos hallazgos como corregidos. B-13 puede quedar `PARCIAL` porque la ausencia de almacenamiento durable del Backend continuará fuera de este bloque.

## 4. Bloque A — Validación estricta del access JWT

Corrige B-11 en Backend y BackendAdmin.

### Contrato obligatorio

Un access JWT solo es válido si cumple simultáneamente:

- firma asimétrica `RS256` —o `ES256` únicamente si ya se adopta de forma integral—;
- header `kid` presente y correspondiente a una clave conocida;
- issuer exacto HTTPS configurado por ambiente;
- audience exacta `cobroup-api`;
- vigencia válida para `exp` y `nbf`, con skew máximo documentado;
- claims presentes y bien formados: `iss`, `aud`, `sub`, `exp`, `iat`, `nbf`, `jti`, `typ=access`, `organization_id`, `organization_code` y `role`;
- `sub`, `organization_id` y `jti` no vacíos; GUID válidos donde aplique;
- `organization_code` no vacío;
- `role` limitado a los roles COBROUP permitidos;
- nunca aceptar refresh JWT como access JWT.

### Implementación esperada

- Centraliza la validación adicional para evitar reglas divergentes entre políticas/controladores.
- Configura explícitamente `MapInboundClaims`, `RoleClaimType` y `NameClaimType`; no dependas de mapeos implícitos diferentes entre servicios.
- Mantén autorización por tenant/rol/recurso después de validar el token.
- Fuera de Development, issuer, audience, JWKS/configuración pública y material del emisor deben ser obligatorios; elimina defaults silenciosos de audience o URLs que oculten configuración faltante.
- Un token inválido debe producir 401 consistente, sin excepción 500 ni información sensible.

### Pruebas mínimas del bloque A

Crea pruebas negativas independientes para:

1. firma incorrecta;
2. algoritmo incorrecto o `none`;
3. `kid` ausente;
4. `kid` desconocido;
5. issuer incorrecto;
6. audience incorrecta;
7. expiración vencida;
8. `nbf` futuro;
9. `typ` ausente o distinto de `access`;
10. cada claim COBROUP obligatorio ausente o malformado;
11. refresh JWT presentado como bearer a una API protegida;
12. token válido aceptado por Backend y BackendAdmin.

No basta con comprobar que el emisor genera los claims: debes demostrar que los validadores rechazan su ausencia o alteración.

## 5. Bloque B — JWKS, claves y retiro de residuos

Corrige I-15 e I-16 sin introducir otro protocolo de identidad.

- Retira de `MS.Cobroup.Backend.Api.csproj` los paquetes `OpenIddict.Validation.*` si ya no son usados por runtime.
- Retira scripts activos, páginas callback sin ruta y código ejecutable del flujo anterior. Conserva migraciones históricas, informes y entradas append-only de `AGENTS.md` como evidencia.
- Reemplaza el HTTP síncrono de `JwksValidationService` (`GetAwaiter().GetResult()`) por un mecanismo asíncrono y cacheado que no bloquee el hilo de validación.
- Elimina el defecto por el cual se ignora el resultado de `SemaphoreSlim.Wait(...)` y se ejecuta `Release()` aunque no se haya adquirido el semáforo.
- Precarga o valida disponibilidad de al menos una clave pública antes de declarar ready fuera de Development.
- Implementa caché con expiración y actualización controlada. Ante `kid` desconocido, refresca una sola vez de forma coordinada y después rechaza; no elijas otra clave por conveniencia.
- Conserva solapamiento de clave activa/anterior y prueba ambos estados.
- Backend nunca debe recibir la clave privada.

Si la librería elegida exige un adaptador propio para JWKS, mantenlo detrás de un puerto/servicio testeable y documenta su política de cache/refresh.

## 6. Bloque C — Refresh, replay y concurrencia PostgreSQL

Corrige B-05 e I-17.

- La rotación debe ejecutarse en una única transacción PostgreSQL con control real de concurrencia.
- Usa bloqueo de fila (`SELECT ... FOR UPDATE`), actualización condicional atómica con filas afectadas o una estrategia equivalente demostrable; no dependas solo de una lectura previa con `RepeatableRead`.
- Dos requests simultáneos con el mismo refresh deben producir exactamente un éxito controlado.
- El segundo intento debe responder 401/409 según contrato documentado, nunca 500.
- Un replay debe revocar toda la familia comprometida y registrar un evento de seguridad sin token completo.
- Revalida usuario, organización y membresía antes de emitir descendientes.
- No confíes en `family_id` leído de un JWT no validado. En `RevokeFamilyByTokenAsync`, revoca usando el registro localizado por hash o valida criptográficamente el token antes de usar claims.
- Logout siempre expira la cookie local, incluso si el token no existe o ya fue revocado.
- El cambio de organización revoca primero la familia anterior y luego emite una familia nueva; prueba que el refresh del tenant anterior no vuelve a emitir.

### Prueba obligatoria del bloque C

Implementa una prueba de integración contra PostgreSQL real con barrera de sincronización para lanzar dos refresh simultáneos. Verifica:

- exactamente una respuesta exitosa;
- ninguna respuesta 500;
- no existen dos descendientes activos;
- el padre queda revocado;
- si se clasifica como replay, toda la familia termina revocada conforme al contrato;
- el resultado es repetible en varias ejecuciones.

Una prueba secuencial con fake/in-memory repository no satisface este criterio.

## 7. Bloque D — Una sola cookie y CSRF real

Corrige B-06, B-12 y M-01 en BackendAdmin y ambos frontends.

### Cookie refresh

- Usa **una sola cookie**, no dupliques el mismo refresh bajo nombres distintos.
- Estrategia preferida: `__Host-cobroup-refresh`, `Secure=true`, `HttpOnly=true`, `SameSite=Strict`, sin atributo `Domain` y obligatoriamente `Path=/`.
- Si eliges una cookie de path restringido, no uses el prefijo `__Host-`; documenta el nombre y garantiza que el path se evalúa contra la URL pública `/identity/...`, no contra la ruta interna posterior al rewrite.
- Las opciones usadas al borrar deben coincidir con nombre, Path, Secure, SameSite y dominio de la cookie emitida.
- No devuelvas refresh JWT en JSON ni lo expongas a JavaScript.
- Access JWT permanece exclusivamente en memoria; solo el tema visual puede seguir usando Web Storage.

### Antiforgery/CSRF

- Protege efectivamente `POST /api/v1/auth/refresh`, `POST /api/v1/auth/logout` y `PUT /api/v1/auth/organizations/current`.
- Registrar `AddAntiforgery` y `UseAntiforgery` no basta: agrega metadata/filtro de validación a cada endpoint mutable.
- Mantén o corrige `GET /api/v1/auth/antiforgery` para emitir el par necesario, con `Cache-Control: no-store`.
- Ambos frontends deben solicitar el token antiforgery con `credentials: include` y enviar `X-XSRF-TOKEN` en cada operación protegida.
- En arranque de la SPA: obtener antiforgery → intentar refresh → cargar contexto. Evita loops y tormentas concurrentes.
- Si el token antiforgery expira, permite un único reacondicionamiento controlado y luego falla a login.
- Mantén single-flight para refresh.

### Pruebas mínimas del bloque D

- El navegador acepta una sola cookie con atributos correctos.
- Refresh/logout/cambio de organización sin cabecera CSRF fallan.
- Cookie correcta + cabecera incorrecta fallan.
- Cookie y cabecera correctas funcionan.
- Logout elimina la cookie con exactamente el mismo Path.
- Ningún refresh aparece en body, Web Storage o logs.
- Dos llamadas frontend concurrentes que reciben 401 producen un solo refresh.

## 8. Bloque E — Rutas API, frontend e ingress

Corrige la parte de rutas de B-06 sin rediseñar toda la plataforma.

### Contrato externo requerido

- Identidad same-origin: `/identity/api/v1/auth/*`.
- El ingress NGINX debe reescribir únicamente `/identity(/|$)(.*)` a `/$2` hacia Admin Backend.
- APIs operativas: `/api/operations/v1/*`.
- El contexto del frontend operativo debe ser alcanzable como `/api/operations/v1/session/me`.
- El contexto del frontend administrativo debe seguir el contrato de identidad: `/identity/api/v1/auth/me`.

### Cambios esperados

- Alinea `SessionController` y `AUTH_CONFIG` operativo para que el endpoint de contexto use `/api/operations/v1/session/me`; no agregues una regla ingress ambigua para todo `/api/v1`.
- Conserva el ingress de identidad separado para evitar que su anotación de rewrite afecte frontend o API operativa.
- Verifica que dev, qa y prod usan `ingressClassName: nginx` y anotaciones NGINX coherentes.
- Asegura que login, antiforgery, refresh, me, organizaciones, cambio y logout resuelven por `/identity` desde ambos hosts PWA.
- Evita URLs absolutas hardcodeadas en bundles si el contrato es same-origin.
- Mantén el dominio dedicado de auth como ruta adicional solo si no crea un segundo contrato divergente.

### Pruebas de rutas

- Renderiza `kubectl kustomize dev`, `qa` y `prod`.
- Agrega pruebas de contrato que verifiquen rutas frontend y rutas de controladores.
- Si existe cluster: ejecuta el flujo real desde los hosts PWA y captura códigos/headers, nunca los valores de tokens/cookies.
- Si no existe cluster: deja comandos exactos para la prueba externa y marca únicamente ese punto como pendiente.

## 9. Bloque F — Configuración JWT en GitOps

Corrige únicamente la parte JWT de B-13.

- Elimina `Jwt__Secret` y variables `*_JWT_SECRET` de dev/qa/prod y workflows.
- BackendAdmin debe recibir por Secret/configuración externa los nombres necesarios para clave privada y `kid`; usa placeholders, nunca valores reales.
- Backend operativo debe recibir solo issuer, audience y JWKS/public key necesarios; jamás la clave privada.
- Define explícitamente por ambiente:
  - `Identity__Issuer`
  - `Identity__Audience=cobroup-api`
  - `Identity__JwksUri` en Backend
  - `Jwt__KeyId` en BackendAdmin
  - `Jwt__PrivateKeyPem` o referencia segura equivalente únicamente en BackendAdmin
  - variables de clave anterior solo cuando exista rotación planificada.
- El pipeline debe fallar si queda un placeholder JWT sin resolver o una variable obligatoria está vacía.
- El cluster dev tampoco debe depender de una clave RSA efímera generada al reiniciar el pod.
- No agregues `Storage:EvidencePath` local como supuesto cierre de B-09; ese hallazgo queda fuera de alcance.

Valida que los manifests rendericen y que la clave privada no aparezca en ConfigMap, Backend operativo, frontend, logs o salida de pruebas.

## 10. Gates obligatorios de esta iteración

Ejecuta, como mínimo:

```powershell
# Backend operativo
dotnet restore
dotnet build MS.Cobroup.Backend.slnx -c Release --no-restore
dotnet test MS.Cobroup.Backend.slnx -c Release --no-build

# Backend administrativo
dotnet restore
dotnet build MS.Cobroup.BackendAdmin.slnx -c Release --no-restore
dotnet test MS.Cobroup.BackendAdmin.slnx -c Release --no-build
dotnet dotnet-ef migrations has-pending-model-changes `
  --project MS.Cobroup.BackendAdmin.Infrastructure/MS.Cobroup.BackendAdmin.Infrastructure.csproj `
  --startup-project MS.Cobroup.BackendAdmin.Api/MS.Cobroup.BackendAdmin.Api.csproj `
  --configuration Release

# Frontends, en cada repositorio
npm test -- --watch=false
npm run build

# GitOps
kubectl kustomize dev
kubectl kustomize qa
kubectl kustomize prod
```

También ejecuta búsquedas que demuestren:

- ausencia runtime de `OpenIddict`, `/connect/`, OIDC y PKCE fuera de historia/migraciones;
- ausencia de `Jwt__Secret`, HMAC y fallbacks conocidos;
- ausencia de access/refresh en `localStorage`, `sessionStorage` o IndexedDB;
- una sola cookie refresh;
- ausencia de `GetAwaiter().GetResult()` en resolución JWKS;
- ningún endpoint cookie-mutable sin CSRF;
- ningún refresh JWT persistido en claro.

No elimines pruebas para obtener verde. Build con warnings debe reportarse como no conforme aunque el proceso retorne código 0.

## 11. Definición de terminado para este bloque

Este bloque solo puede declararse terminado cuando:

- B-06, B-11, B-12, I-15, I-16, I-17 y M-01 tienen implementación y pruebas locales reproducibles;
- B-05 tiene prueba PostgreSQL concurrente real o queda explícitamente pendiente de verificación externa;
- B-13 queda cerrado solo en su componente JWT/rutas y se reconoce como parcial en el resultado global;
- las cuatro suites y builds terminan sin fallos;
- Kustomize renderiza los tres ambientes;
- no se creó ningún commit ni operación remota.

No afirmes que la auditoría global queda aprobada: B-09 y otros hallazgos deliberadamente permanecen abiertos.

## 12. Formato obligatorio de entrega

Entrega una matriz final:

| Hallazgo | Estado | Archivos modificados | Pruebas ejecutadas | Evidencia o limitación |
|---|---|---|---|---|

Usa solo:

- `CORREGIDO Y VERIFICADO`
- `IMPLEMENTADO, PENDIENTE DE VERIFICACIÓN EXTERNA`
- `PARCIAL`
- `ABIERTO`

Incluye después:

1. resumen técnico de la solución JWT/cookie/rutas;
2. contrato final de cookie y CSRF;
3. contrato final de rutas internas/externas;
4. nombres de variables y Secrets requeridos por ambiente, sin valores;
5. resultado exacto de cada build, suite, prueba de concurrencia, migración y Kustomize;
6. pruebas no ejecutadas y razón concreta;
7. hallazgos fuera de alcance que permanecen abiertos;
8. `git status --short` final de cada repositorio;
9. confirmación explícita de que no se hizo commit, push, merge, tag, PR ni despliegue.

La meta es **reducir hallazgos con correcciones profundas y auditables**, no producir una declaración artificial de cierre total.
