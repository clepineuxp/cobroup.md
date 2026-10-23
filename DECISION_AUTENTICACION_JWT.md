# Decisión de arquitectura — Autenticación exclusiva con JWT y firma simétrica por ambiente

> **Estado:** APROBADA por el propietario del producto  
> **Fecha:** 2026-10-22  
> **Vigencia:** inmediata para planeación, implementación, pruebas y auditoría  
> **Autoridad:** este documento complementa y prevalece sobre referencias históricas de identidad en `COBROUP.md`, `AGENTS.md`, ADR anteriores, roadmaps e informes de avance.  
> **Implementación actual:** pendiente de adecuación y reauditoría; esta decisión no aprueba el código existente.  
> **Decisión reemplazada:** quedan retirados definitivamente RSA, PEM/PFX, firma asimétrica, JWKS, proveedores efímeros/persistentes y cualquier bandera de *acknowledge* asociada.

## 1. Decisión cerrada

COBROUP utilizará exclusivamente tokens JSON Web Token (JWT) firmados con el algoritmo simétrico `HS256`. `MS.Cobroup.BackendAdmin` será el único servicio autorizado para autenticar credenciales, emitir, rotar y revocar tokens. `MS.Cobroup.Backend` será consumidor/validador y nunca almacenará contraseñas ni emitirá tokens, aunque recibirá el secreto simétrico necesario para validar la firma localmente.

No se usarán OIDC, OpenIddict, Authorization Code + PKCE, RSA, certificados, archivos `.pem`, `.pfx`, JWKS ni `alg=none`. No habrá modo híbrido, fallback de algoritmo ni compatibilidad paralela. Las referencias conservadas en auditorías y ADR previos son evidencia histórica, no requisitos vigentes.

## 2. Contrato obligatorio de tokens

### 2.1 Access token

- JWT firmado exclusivamente con `HS256`; cualquier otro valor de `alg`, incluso otro HMAC, se rechaza.
- Vida recomendada entre 5 y 15 minutos; máximo 15 minutos salvo decisión de riesgo documentada.
- Claims registrados obligatorios: `iss`, `aud`, `sub`, `exp`, `iat`, `nbf`, `jti` y `typ=access`.
- Claims COBROUP obligatorios: `organization_id`, `organization_code` y `role`; deben proceder de una membresía activa consultada en servidor.
- Audiencia exacta: `cobroup-api`. Emisor exacto, HTTPS y distinto por ambiente.
- La API valida firma, algoritmo, identificador de versión de clave, emisor, audiencia, expiración, `nbf`, tipo y claims obligatorios. Se prohíben `ValidateIssuer=false`, `ValidateAudience=false`, claves predeterminadas y fallbacks silenciosos.

### 2.2 Refresh token

- JWT firmado exclusivamente con `HS256`, con `typ=refresh`, audiencia `cobroup-refresh`, `sub`, `organization_id`, `jti`, `family_id`, `iat`, `nbf` y `exp`.
- Vida máxima de 30 días y uso único. Cada refresh rota el token y revoca atómicamente el anterior.
- El servidor persiste solo el hash criptográfico del identificador/token, familia, usuario, organización, expiración, revocación y reemplazo; se prohíbe almacenar el token completo.
- El navegador lo recibe únicamente en cookie `HttpOnly`, `Secure`, `SameSite=Strict` —o `Lax` con justificación funcional documentada— y alcance mínimo. Se prohíbe `localStorage`, `sessionStorage`, IndexedDB, URL, logs o estado JavaScript persistente.
- Refresh, logout y cambio de organización requieren protección CSRF, límites de frecuencia y auditoría.
- Un replay revoca toda la familia y genera un evento de seguridad. Dos refresh concurrentes del mismo token no pueden producir dos descendientes válidos.

## 3. Secreto simétrico por ambiente

### 3.1 Custodia y distribución

- Dev, QA y Prod usan secretos distintos, aleatorios e independientes. Está prohibido reutilizar un secreto entre ambientes.
- Cada secreto debe contener como mínimo 256 bits aleatorios generados con CSPRNG. Se transportará codificado en Base64, sin convertir una frase humana en clave.
- Los valores reales se administran en secretos de GitHub/Kubernetes desde `cobroup.environments`. Nunca se versionan en Git, ConfigMap, imagen OCI, archivo de aplicación, log, frontend ni documentación.
- Nombres de secretos de GitHub: `DEV_JWT_SIGNING_KEY_BASE64`, `QA_JWT_SIGNING_KEY_BASE64` y `PROD_JWT_SIGNING_KEY_BASE64`.
- Clave de Kubernetes/.NET: `Jwt__SigningKeyBase64`. Debe inyectarse, con el mismo valor dentro de un ambiente, tanto en `MS.Cobroup.BackendAdmin` como en `MS.Cobroup.Backend`.
- El inicio falla inmediatamente si el valor falta, no es Base64 válido o decodifica a menos de 32 bytes. No existen secretos por defecto, desarrollo automático ni fallback.

### 3.2 Rotación

- Cada token incluye un identificador de versión de clave (`kid`) que no expone el secreto.
- El emisor usa una única clave activa; ambos backends pueden cargar temporalmente la clave activa y una o más claves anteriores solo para validación.
- Una clave anterior se conserva como máximo durante la ventana necesaria para vencer los access tokens emitidos con ella. Los refresh tokens deben reemitirse o revocarse según el procedimiento de rotación.
- La rotación debe actualizar de forma coordinada los Secrets de ambos backends y todas sus réplicas, con procedimiento probado de despliegue, rollback y revocación de emergencia.

### 3.3 Riesgo aceptado y límites

La firma simétrica implica que ambos backends conocen material capaz de crear una firma válida. Por tanto, el compromiso de `MS.Cobroup.Backend` podría permitir falsificar tokens. El propietario acepta este modelo, condicionado a mínimo privilegio del Secret, aislamiento por namespace/ambiente, acceso exclusivo de las dos cargas autorizadas, rotación, auditoría y respuesta a incidentes. Backend operativo no implementará endpoints ni servicios de emisión.

El backlog productivo aplicable es `BACKLOG_SEGURIDAD_FIRMA_SIMETRICA.md`. No existe requisito ni proyección futura de migrar a RSA/JWKS.

## 4. Endpoints de identidad

| Método y ruta | Responsabilidad |
|---|---|
| `POST /api/v1/auth/login` | Validar credenciales y membresía; emitir access JWT y cookie de refresh JWT. |
| `POST /api/v1/auth/refresh` | Validar cookie, revalidar contexto, rotar atómicamente y devolver un access JWT nuevo. |
| `POST /api/v1/auth/logout` | Revocar token/familia y expirar cookie. |
| `GET /api/v1/auth/me` | Devolver contexto derivado de un access JWT válido. |
| `GET /api/v1/auth/organizations` | Listar únicamente membresías activas del usuario autenticado. |
| `PUT /api/v1/auth/organizations/current` | Revocar la familia anterior y emitir una sesión nueva para la membresía seleccionada. |

El endpoint JWKS debe retirarse. Las rutas externas e internas deben coincidir; cualquier `PathBase` o reescritura de ingress se prueba por ambiente.

## 5. Autorización y tenant

- El tenant procede exclusivamente del token validado; nunca de un parámetro confiado del cliente.
- Cada operación vuelve a comprobar tenant, rol y alcance del recurso. Poseer un JWT no sustituye autorización por ruta, cliente, préstamo, pago o evidencia.
- Login, refresh y cambio de organización revalidan los estados de usuario, organización y membresía.
- El cambio de organización revoca primero la familia del tenant anterior. Ningún refresh anterior puede reemitir tokens.

## 6. Seguridad operacional

- Login y refresh tienen rate limit por IP y sujeto, bloqueo progresivo y respuestas resistentes a enumeración.
- No se registran contraseñas, JWT completos, cookies ni secretos. Los logs usan identificadores parciales/hasheados, actor, tenant, resultado, IP confiable y correlation ID.
- Los frontends conservan el access token solo en memoria y aplican CSP estricta.
- La configuración de proxies confía únicamente en los CIDR/IP del ingress.
- Las migraciones de sesión se aplican mediante job idempotente antes del rollout; readiness valida el esquema esperado.

## 7. Pruebas mínimas para aprobación

1. Firma `HS256` válida y rechazo de `none`, RSA, otros HMAC, firma, `kid`, issuer, audience, tipo, expiración o `nbf` incorrectos.
2. Falta de secreto, Base64 inválido o menos de 32 bytes impiden el arranque de ambos backends en todos los ambientes.
3. Secretos diferentes entre Dev, QA y Prod; valor idéntico dentro de cada ambiente para emisor y validador; ninguna exposición en Git, render, logs o frontend.
4. Rotación coordinada: la clave activa emite; la anterior solo valida durante el solapamiento; después se rechaza.
5. Login correcto/incorrecto, usuario inactivo, organización suspendida y membresía inactiva.
6. Refresh, replay y concurrencia dejan un único descendiente y revocan la familia comprometida.
7. Cambio de organización invalida todos los refresh del tenant anterior.
8. Refresh ausente de Web Storage, cuerpo JSON, logs y base de datos en claro.
9. Aislamiento cross-tenant, autorización cross-route y flujo end-to-end por ingress en Dev/QA/Prod.
10. Build sin warnings, todas las suites verdes y escaneos de dependencias y secretos.

## 8. Criterio de auditoría

Una implementación JWT se considera conforme únicamente cuando cumple todos los requisitos anteriores con evidencia reproducible. Compilar, emitir un token o declarar un Secret no constituye aprobación. El código y los manifiestos actuales deben adecuarse y reauditarse antes de commit, merge o despliegue.

Las auditorías no deberán abrir hallazgos por ausencia de RSA, PEM, JWKS u OIDC. Sí deberán abrirlos por claves débiles/reutilizadas/expuestas, algoritmo no restringido, falta de rotación, distribución excesiva, emisión desde el backend operativo o incumplimiento del contrato de sesión.
