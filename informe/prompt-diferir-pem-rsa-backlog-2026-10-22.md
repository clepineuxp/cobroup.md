# Prompt para diferir el uso de claves PEM y formalizar el proveedor RSA futuro

Actúa como ingeniero principal de .NET, seguridad JWT, Kubernetes y documentación técnica. Debes modificar la solución COBROUP para formalizar una nueva decisión arquitectónica: **en la fase actual no se suministrará ni utilizará una clave privada RSA en formato PEM administrada externamente**. La integración de claves persistentes mediante PEM, PFX, KMS o un servicio de gestión de claves queda como backlog obligatorio para una fase futura de endurecimiento productivo.

Debes basarte en:

- `D:\Proyectos\cobroup\cobroup.md\FRAMEWORK_DESARROLLO_CSHARP.md`.
- Los `AGENTS.md` de cada repositorio afectado.
- La documentación vigente de autenticación JWT.
- `D:\Proyectos\cobroup\cobroup.md\informe\auditoria-backend-csharp-2026-10-22-v6.html`.

## Restricciones de trabajo

1. Trabaja únicamente en ramas `feature/*`. `cobroup.environments` está actualmente en `dev`: **no edites ese repositorio hasta crear o cambiar a una rama `feature/*` con autorización del usuario**.
2. Preserva todos los cambios locales existentes.
3. No hagas commit, push, merge, rebase, tag ni PR sin autorización expresa.
4. No reincorpores OIDC, OpenIddict, Authorization Code ni PKCE. COBROUP continúa usando exclusivamente JWT.
5. No sustituyas RS256 por HS256 ni distribuyas un secreto simétrico entre BackendAdmin y el backend operativo.
6. No escribas claves privadas, JWT, cookies, secretos ni cadenas de conexión en logs, pruebas, documentación o respuestas.
7. Actualiza `AGENTS.md` únicamente mediante una nueva entrada append-only con resultados reales.

## Decisión obligatoria

Incorpora textualmente esta decisión en la documentación normativa y de planeación pertinente:

> COBROUP continuará usando JWT firmados. En la fase actual de desarrollo y validación no se configurará una clave privada externa en formato PEM. BackendAdmin utilizará un proveedor RSA efímero generado en memoria, habilitado de forma explícita y restringido a entornos no productivos. La clave y el `kid` cambian al reiniciar la instancia; en consecuencia, los JWT emitidos antes del reinicio pueden dejar de ser válidos. Este modo no se considera apto para producción, alta disponibilidad ni múltiples réplicas emisoras. La custodia de una clave persistente, su rotación, solapamiento de claves y recuperación operativa mediante Kubernetes Secret, PFX, PEM, AWS KMS u otro proveedor aprobado quedan registrados como backlog obligatorio previo a producción.

Esta decisión significa:

- Los JWT **siguen requiriendo firma criptográfica**.
- Se difiere el uso de una clave privada persistente suministrada mediante PEM.
- No se debe afirmar que producción está aprobada mientras solo exista el proveedor efímero.
- La ausencia de PEM deja de ser un error en Development/Test cuando el modo efímero está habilitado explícitamente.
- En QA/Production debe existir un bloqueo explícito o una aceptación temporal de riesgo igualmente explícita. No habilites silenciosamente claves efímeras fuera de Development.

## Implementación requerida

### 1. Abstracción del proveedor de firma

Refactoriza la autoridad JWT de BackendAdmin para que dependa de un puerto claramente definido, por ejemplo:

```csharp
public interface IJwtSigningKeyProvider
{
    string ActiveKeyId { get; }
    SigningCredentials GetSigningCredentials();
    IReadOnlyCollection<SecurityKey> GetValidationKeys();
    JsonWebKeySet GetPublicJwks();
}
```

Los nombres pueden adaptarse al proyecto, pero el servicio de emisión de tokens no debe conocer PEM, rutas físicas, Kubernetes ni detalles del proveedor.

### 2. Proveedor actual: RSA efímero

Implementa un adaptador como `EphemeralRsaSigningKeyProvider` con estas condiciones:

- Genera una clave RSA de al menos 2048 bits mediante APIs criptográficas seguras de .NET.
- La clave privada permanece únicamente en memoria del proceso.
- Publica exclusivamente los parámetros públicos mediante JWKS.
- Usa RS256 para access y refresh JWT.
- Genera un `kid` no ambiguo por ciclo de vida del proceso; no reutilices un `kid` fijo con material criptográfico diferente.
- Se registra como singleton para que emisión, validación local y JWKS compartan exactamente la misma instancia.
- No escribe ni exporta la clave privada a archivos, logs, base de datos, respuestas HTTP o variables de entorno.
- Libera correctamente los recursos criptográficos al detenerse la aplicación.

### 3. Selección explícita y fail-fast

Define configuración explícita, por ejemplo:

```text
Jwt:SigningProvider=EphemeralRsa
Jwt:AllowEphemeralSigning=true
```

Reglas obligatorias:

- En `Development` y pruebas se permite `EphemeralRsa` únicamente si `AllowEphemeralSigning=true`.
- Si el flag no está presente, el arranque debe fallar de forma clara; no uses un fallback silencioso.
- En `Staging`, QA o `Production`, `EphemeralRsa` debe rechazarse por defecto.
- Si el negocio decide aceptar temporalmente el riesgo fuera de Development, debe existir otro flag explícito y alarmante, acompañado de warning estructurado y documentación de la excepción. No lo actives en este trabajo sin autorización expresa.
- Mantén preparado el punto de extensión para un futuro proveedor persistente, pero no implementes AWS KMS, Secrets Manager, S3 ni infraestructura ficticia.

### 4. Retiro del PEM del flujo actual

Después de crear una rama válida para environments:

- Retira `DEV_JWT_PRIVATE_KEY_PEM`, `QA_JWT_PRIVATE_KEY_PEM` y `PROD_JWT_PRIVATE_KEY_PEM` de los workflows y del renderer actual.
- Elimina la sustitución, normalización y validación PEM del flujo activo.
- Retira los placeholders de clave privada PEM de los Secrets que pertenezcan exclusivamente a este mecanismo.
- Configura el modo efímero solo donde esté permitido por la decisión anterior.
- No dejes secretos vacíos ni claves de configuración muertas.
- Conserva la validación integral de YAML, placeholders, PVC, mounts y `kubectl kustomize`; no deshagas las mejoras independientes de B-13.
- Ajusta las pruebas del renderer para que ya no utilicen un PEM falso como fixture válido.

### 5. Comportamiento operativo obligatorio

Documenta y demuestra que:

- Un reinicio de BackendAdmin genera una nueva clave y un nuevo `kid`.
- Los tokens firmados por la instancia anterior pueden ser rechazados después del reinicio.
- El backend operativo actualiza JWKS según su política, pero el modo efímero no garantiza continuidad de sesiones durante reinicios.
- Dos réplicas de BackendAdmin con claves efímeras distintas no son compatibles como emisores activos detrás de balanceo.
- Por lo anterior, el modo actual es exclusivamente de desarrollo/validación y no constituye una solución productiva.

No ocultes estas limitaciones mediante tiempos de caché, un `kid` fijo, sincronización improvisada o aceptación de cualquier clave.

## Backlog obligatorio de seguridad productiva

Crea o actualiza el backlog/roadmap con una tarea prioritaria previa a producción que incluya, como mínimo:

1. Seleccionar proveedor persistente: Kubernetes Secret con PEM/PFX, AWS KMS u otro HSM/KMS aprobado.
2. Mantener el puerto `IJwtSigningKeyProvider` sin modificar casos de uso ni contratos HTTP.
3. Custodia, permisos mínimos, cifrado en reposo y procedimiento de recuperación.
4. Rotación de claves con `kid` único y ventana de solapamiento de clave activa/anterior.
5. Publicación JWKS de claves públicas únicamente.
6. Prueba de tokens emitidos antes, durante y después de la rotación.
7. Compatibilidad con múltiples réplicas de BackendAdmin.
8. Procedimiento de rollback y revocación de emergencia.
9. Alertas por clave próxima a expirar, fallo de carga y uso de proveedor efímero fuera de Development.
10. Gate de despliegue que impida producción mientras el proveedor persistente no esté configurado y validado.

El backlog debe tener criterio de aceptación, responsable pendiente de asignación, dependencia y condición explícita: **bloquea salida a producción**.

## Documentación que debe actualizarse

Revisa y alinea, sin borrar el historial:

- Decisión maestra de autenticación JWT.
- ADR de identidad y firma JWT.
- Roadmap y backlog.
- README y contexto de BackendAdmin.
- Guía de environments y despliegue.
- Matriz de ambientes.
- Todos los `AGENTS.md` afectados, con entrada append-only.
- Informes de avance que presenten PEM persistente como requisito actual.

No reescribas auditorías históricas. Añade una adenda o nueva versión que indique que B-13 cambia de alcance por una decisión aprobada: la validación PEM deja de aplicar al flujo actual, pero el sistema queda **no apto para producción** hasta implementar el proveedor persistente del backlog.

## Pruebas obligatorias

Agrega y ejecuta pruebas que demuestren:

1. BackendAdmin inicia en Development sin PEM cuando `EphemeralRsa` está autorizado explícitamente.
2. BackendAdmin falla si se selecciona modo efímero sin el flag de aceptación.
3. BackendAdmin falla fuera de Development cuando se intenta usar modo efímero sin autorización excepcional.
4. Access y refresh JWT se firman con RS256 y contienen el `kid` activo.
5. Los tokens se validan con la clave pública publicada por JWKS.
6. JWKS no contiene parámetros privados (`d`, `p`, `q`, `dp`, `dq`, `qi`).
7. Emisión y JWKS comparten el mismo singleton/proveedor.
8. Dos instancias nuevas generan claves y `kid` diferentes.
9. Un token emitido por la primera instancia no se valida con la segunda; documenta esta limitación como comportamiento esperado del modo efímero.
10. Los manifiestos renderizados no contienen placeholders ni variables PEM.
11. `kubectl kustomize` funciona para los ambientes permitidos.
12. Los workflows se detienen antes de transferencia si la configuración del proveedor es incompatible con el ambiente.

Ejecuta también:

```text
dotnet build MS.Cobroup.BackendAdmin.slnx -c Release --no-incremental
dotnet test MS.Cobroup.BackendAdmin.slnx -c Release
dotnet build MS.Cobroup.Backend.slnx -c Release --no-incremental
dotnet test MS.Cobroup.Backend.slnx -c Release
dotnet ef migrations has-pending-model-changes ...
dotnet list <solución> package --vulnerable --include-transitive
python scripts/test_render_manifests.py -v
kubectl kustomize dev
kubectl kustomize qa
kubectl kustomize prod
git diff --check
```

No declares aprobado un comando que no hayas ejecutado. Si QA/producción quedan intencionalmente bloqueados por no tener proveedor persistente, repórtalo como resultado esperado y pendiente de backlog, no como error corregido.

## Entrega esperada

Entrega una matriz:

```text
Área | archivos modificados | decisión aplicada | pruebas | resultado | limitación/backlog
```

Incluye además:

- Resumen de la nueva decisión.
- Evidencia de que el runtime actual no consume PEM.
- Evidencia de RS256/JWKS sin material privado.
- Resultado real por ambiente.
- Lista exacta de documentación actualizada.
- Backlog productivo con criterios de aceptación.
- Estado Git final y confirmación de que no se realizó commit/push/merge/tag/PR.

## Criterio final

La tarea se considera completa únicamente cuando el modo actual puede operar sin PEM de forma explícita y segura para desarrollo, la documentación deja de afirmar que el PEM es un requisito inmediato, y queda inequívocamente registrado que una clave persistente y rotatable es condición obligatoria antes de QA productiva, alta disponibilidad o producción.

