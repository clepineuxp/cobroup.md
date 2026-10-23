# Prompt maestro para Gemini — Remediación v9: GitHub Environments, rotación HS256 y PVC durable

Actúa como **arquitecto senior .NET, especialista en seguridad JWT, GitHub Actions y Kubernetes**. Debes implementar y verificar las correcciones; no entregues únicamente recomendaciones.

## 1. Contexto vinculante

Trabaja sobre:

- `MS.Cobroup.BackendAdmin`: autoridad de identidad y único emisor lógico JWT.
- `MS.Cobroup.Backend`: API operativa, validador JWT y propietario del almacenamiento de archivos.
- `cobroup.environments`: GitHub Actions, Kubernetes, Secrets, PVC y renderer.
- `cobroup.md`: normativa y documentación.

Lee completamente antes de modificar:

1. `cobroup.md/FRAMEWORK_DESARROLLO_CSHARP.md`.
2. Los `AGENTS.md` aplicables.
3. `cobroup.md/informe/auditoria-backend-csharp-2026-10-23-v9.html`.
4. `cobroup.md/DECISION_AUTENTICACION_JWT.md`.
5. `cobroup.md/BACKLOG_SEGURIDAD_FIRMA_SIMETRICA.md`.
6. `cobroup.md/RUNBOOK_ROTACION_JWT_HS256.md`.
7. `cobroup.md/DECISION_ALMACENAMIENTO_ARCHIVOS.md`.

La decisión permanece cerrada:

- JWT exclusivo con `HS256`.
- Secreto independiente por ambiente.
- BackendAdmin es el único emisor lógico.
- Backend conoce el secreto solo para validar.
- No usar OIDC, OpenIddict runtime, RSA, PEM/PFX, JWKS, acknowledge, `alg=none` ni fallback.
- Los archivos se almacenan actualmente en PVC mediante un puerto agnóstico; AWS S3 queda previsto para el futuro, sin implementarlo ahora.

Existen cambios locales sin commit. Presérvalos y trabaja sobre el diff actual. No hagas reset, checkout destructivo ni clean. No crees commit, push, merge, rebase, tag o PR.

## 2. Objetivo y secuencia obligatoria

Remedia los tres bloques de la auditoría v9 en este orden:

1. GitHub Environments y custodia segura de secretos sin YAML persistente.
2. Rotación HS256 completa, coherente y desplegable.
3. Cierre técnico y operativo de B-09/PVC durable.

No declares el trabajo terminado si falta evidencia externa de GitHub o Kubernetes. En ese caso, deja la implementación y el runbook listos, pero reporta el control como pendiente de validación real.

---

# FASE A — GitHub Environments y creación segura de configuración Kubernetes

## A.1 Environments que creará el propietario

El propietario creará manualmente en GitHub los Environments:

- `dev`
- `qa`
- `prod`

Actualiza cada workflow para seleccionar exactamente el Environment correspondiente:

- `deploy-dev.yml` → `environment: dev`
- `deploy-qa.yml` → `environment: qa`
- `deploy-prod.yml` → `environment: prod`

No mantengas el Environment compartido `cobroup` para secretos de aplicación. Producción debe admitir required reviewers/protection rules, pero no simules que esas reglas existen si no puedes observar GitHub.

## A.2 Secrets y variables requeridos por cada Environment

Usa los mismos nombres dentro de cada Environment; los valores cambian por ambiente.

### Secrets

- `OPERATIONS_DATABASE_CONNECTIONSTRING`
- `IDENTITY_DATABASE_CONNECTIONSTRING`
- `JWT_ACTIVE_SIGNING_KEY_BASE64`
- `JWT_PREVIOUS_SIGNING_KEY_BASE64` — opcional; presente solo durante una rotación programada
- `GHCR_TOKEN`
- `SSH_HOST`
- `SSH_USER`
- `SSH_KEY`
- `SSH_PORT`

### Variables no secretas

- `JWT_ACTIVE_KEY_ID`
- `JWT_PREVIOUS_KEY_ID` — vacío cuando no existe solapamiento
- `GHCR_USERNAME`
- `KUBERNETES_NAMESPACE`
- `PVC_STORAGE_CLASS`

Reglas:

- La clave JWT activa debe ser Base64 válido y decodificar a mínimo 32 bytes CSPRNG.
- `JWT_ACTIVE_KEY_ID` es obligatorio, no puede usar default y debe cumplir una expresión estricta como `[A-Za-z0-9._-]{1,64}`.
- Previous ID y previous key deben estar ambos presentes o ambos ausentes.
- Active y previous no pueden compartir `kid` ni material criptográfico.
- Dev, QA y Prod deben usar material distinto.

## A.3 Evidencia segura de separación entre ambientes

Los workflows de un Environment no pueden leer los secretos de los otros. Por tanto, no presentes como eficaz una comparación que solo mira variables opcionalmente presentes.

Implementa un mecanismo verificable sin revelar claves:

- Usa una variable de repositorio `JWT_KEY_FINGERPRINTS_JSON` con fingerprints SHA-256 no secretos para `dev`, `qa` y `prod`.
- El validador debe comprobar que existen exactamente las tres entradas y que sus fingerprints son distintos.
- En cada workflow, calcula en memoria el SHA-256 de los bytes decodificados de `JWT_ACTIVE_SIGNING_KEY_BASE64` y comprueba que coincide con la entrada del ambiente.
- No imprimas secreto, bytes, Base64, fingerprint calculado ni fragmentos reversibles.
- Añade pruebas negativas por fingerprint ausente, mal formado, duplicado y que no coincide.

Si propones otro mecanismo, debe demostrar la misma propiedad con evidencia automatizada y sin poner los tres secretos en un único job.

## A.4 Prohibición de YAML persistente con secretos

Corrige B-22. El renderer actual modifica `dev|qa|prod/**/secret.yaml` en el checkout y los workflows copian esos archivos a `/opt/cobroup/deploys/<env>`. Esto está prohibido.

Implementa estas reglas:

1. Nunca reescribas plantillas rastreadas con valores reales.
2. No copies a la VPS archivos que contengan JWT, cadenas de conexión, GHCR tokens o credenciales.
3. Retira `secret.yaml` con placeholders de los `kustomization.yaml` si su render terminaría persistido.
4. Aplica los Kubernetes Secrets directamente desde memoria/stdin durante el workflow:
   - genera JSON/YAML en memoria leyendo variables de entorno;
   - codifica el campo Kubernetes `data` correctamente;
   - envíalo por un pipe directo a `kubectl apply -f -` en la conexión remota;
   - usa `set +x`, no incluyas valores en argumentos, logs, excepciones o nombres de proceso;
   - no crees archivos temporales remotos con datos sensibles;
   - si fuera inevitable un temporal local, usa directorio de runner aislado, permisos mínimos, `finally/trap` y demuestra su eliminación antes de terminar incluso ante fallo.
5. Copia a `/opt/cobroup/deploys/<env>` exclusivamente manifiestos no sensibles.
6. El pipeline debe fallar antes de SCP/SSH si la configuración es inválida.
7. No registres el manifiesto de Secret renderizado como artifact.

Los dos backends del mismo namespace deben referenciar un Secret JWT común, por ejemplo `jwt-signing-secret`, mientras sus cadenas de conexión permanecen en Secrets separados. Los frontends nunca deben referenciar ese Secret.

## A.5 CI/CD

- Ejecuta `python -m unittest scripts/test_render_manifests.py -v` como gate previo a render, SSH o SCP.
- Fija acciones de terceros por commit SHA; documenta actualización/renovación.
- Mantén pruebas de ausencia de placeholders, residuos RSA/JWKS, exposición a frontends y persistencia de secretos.
- Añade una prueba que capture todas las escrituras del renderer y demuestre que ninguna plantilla rastreada ni carpeta SCP contiene valores sensibles.

---

# FASE B — Rotación HS256 operacional y coherente

## B.1 Contrato único de configuración

Elimina la contradicción entre código, runbook y GitOps. Usa exactamente este contrato .NET/Kubernetes:

- `Jwt__ActiveKeyId`
- `Jwt__SigningKeyBase64`
- `Jwt__PreviousValidationKeys__0__KeyId`
- `Jwt__PreviousValidationKeys__0__SigningKeyBase64`

La entrada `0` es opcional y existe solo durante una rotación. Esta fase soportará una rotación serial con una clave anterior; no inventes múltiples rotaciones simultáneas. El código puede conservar colección interna, pero GitOps y runbook deben usar la forma anterior sin ambigüedad.

El Environment mapea:

- `JWT_ACTIVE_KEY_ID` → `Jwt__ActiveKeyId`
- `JWT_ACTIVE_SIGNING_KEY_BASE64` → `Jwt__SigningKeyBase64`
- `JWT_PREVIOUS_KEY_ID` → `Jwt__PreviousValidationKeys__0__KeyId`
- `JWT_PREVIOUS_SIGNING_KEY_BASE64` → `Jwt__PreviousValidationKeys__0__SigningKeyBase64`

## B.2 Fail-fast obligatorio en ambos servicios y renderer

Rechaza al iniciar/renderizar:

- `ActiveKeyId` ausente, vacío, placeholder o con formato inválido;
- clave activa ausente, placeholder, Base64 inválido o menor de 32 bytes;
- previous ID sin previous key o viceversa;
- previous key inválida/débil;
- `kid` activo igual al previous;
- material activo igual al previous aunque tenga otro `kid`;
- hijos incompletos de `PreviousValidationKeys` — no usar `continue` silencioso;
- claves duplicadas o propiedades desconocidas que oculten un error tipográfico;
- diferencias entre el conjunto cargado por Backend y BackendAdmin dentro del mismo ambiente.

No permitas default `k1` en runtime. La configuración debe ser explícita.

## B.3 Procedimiento de rotación programada

Implementa y documenta una rotación reproducible:

### Etapa 0 — estado estable

- Activa `k1`.
- Sin clave anterior.

### Etapa 1 — prepublicación

- Activa `k1`.
- Previous slot contiene `k2`.
- Reiniciar primero Backend y después BackendAdmin.
- Todos los pods pueden validar k1 y k2; todavía se emite únicamente con k1.

### Etapa 2 — activación

- Activa `k2`.
- Previous slot contiene `k1`.
- Reiniciar primero Backend y después BackendAdmin.
- Durante el rollout, pods antiguos y nuevos deben aceptar ambos `kid`; solo los nuevos emisores firman con k2.

### Etapa 3 — retiro

- Esperar TTL máximo de access token + skew: 15 + 5 = 20 minutos.
- Los refresh usados durante la ventana deben rotar y persistirse bajo k2.
- Persistir `SigningKeyId` en cada refresh token o introducir evidencia equivalente que permita identificar sesiones aún asociadas a k1.
- Al finalizar la ventana, revocar atómicamente refresh tokens/familias todavía asociados a k1 y registrar auditoría. Esos usuarios deberán autenticarse nuevamente.
- Retirar previous k1 de ambos servicios y reiniciar primero Backend, luego BackendAdmin.
- Demostrar que k1 es rechazado y k2 continúa operativo.

No declares “cero desconexiones” si se revocan refresh inactivos. Documenta el comportamiento exacto.

### Emergencia

- Generar nuevo secreto/kid.
- No conservar la clave comprometida como previous.
- Revocar todas las familias afectadas.
- Desplegar coordinadamente ambos backends.
- Emitir alerta y evidencia auditable sin revelar material.

## B.4 Pruebas obligatorias

Añade pruebas que cubran:

- configuración activa/previous válida;
- todas las combinaciones inválidas de fail-fast indicadas;
- token k1 y k2 aceptados en prepublicación/activación;
- nuevos tokens emitidos solo con active kid;
- retiro de k1;
- un token real emitido por BackendAdmin validado por Backend;
- dos instancias de cada servicio con el mismo Secret;
- rollout simulado con pods viejos/nuevos;
- refresh k1 rotado a k2 y persistencia de `SigningKeyId`;
- revocación de refresh/familias restantes de k1;
- emergencia sin solapamiento;
- manifiestos y workflow generan exactamente la estructura esperada sin persistir secretos.

Actualiza `RUNBOOK_ROTACION_JWT_HS256.md` para que coincida literalmente con el contrato implementado.

---

# FASE C — Cierre de B-09 y PVC durable

## C.1 Puerto agnóstico y clave lógica

Refactoriza el puerto para archivos binarios generales; no debe estar nominalmente limitado a Evidence/Payment.

- Expón operaciones equivalentes a `Put`, `OpenRead`, `Delete` y `Exists/GetMetadata` mediante streams y claves lógicas.
- Domain/Core no deben conocer paths, PVC, `FileInfo` ni SDK AWS.
- Mantén una fachada temporal si los casos de uso de evidencias la requieren, pero la infraestructura base debe ser genérica.
- Formato canónico obligatorio:
  `tenants/{tenantId}/{category}/{aggregateId}/{sha256}.{ext}`
- `category` procede de una allowlist definida por servidor.
- Migra o preserva compatibilidad de claves anteriores explícitamente; no cambies claves persistidas sin plan de datos.
- AWS S3 sigue sin implementarse. La clave debe ser directamente reutilizable como object key futuro.

## C.2 Seguridad filesystem

Implementa y prueba:

1. Rechazo de rutas absolutas, segmentos vacíos, `.`, `..`, separadores alternativos, NUL, `:`, claves no canónicas y categorías no autorizadas.
2. Pertenencia al root usando límites reales de segmento/path separator, no simple prefijo textual.
3. Rechazo de symlinks/reparse points en root y cada componente existente; documenta el riesgo TOCTOU residual.
4. Directorios/archivos con permisos mínimos para usuario no-root.
5. Temporal en el mismo filesystem, flush cuando corresponda y publicación atómica con `overwrite: false`.
6. Si el destino existe: mismo tamaño+SHA256 = idempotencia; contenido diferente = error de integridad, nunca overwrite.
7. Concurrencia multiproceso/RWX basada en atomicidad del filesystem, no locks estáticos en memoria.
8. No silencies errores de move, delete o cleanup. Usa excepciones accionables y logs estructurados; añade reconciliación segura de temporales huérfanos.
9. Streaming con límite antes y durante la copia.
10. Para JPEG/PNG/WebP, validación de decodificación completa, estructura, MIME, dimensiones y límites contra decompression bombs; magic bytes aislados no son suficientes.

## C.3 Kubernetes

- Dev/QA RWO solo con una réplica; Prod con múltiples réplicas requiere RWX.
- `PVC_STORAGE_CLASS` debe producir `storageClassName` explícita y válida; el pipeline falla si falta en QA/Prod.
- Configura `runAsNonRoot`, `runAsUser`, `runAsGroup`, `fsGroup`, seccomp y filesystem raíz read-only cuando sea compatible.
- Readiness debe comprobar montaje y escritura segura sin dejar basura; liveness no debe reiniciar pods por fallos transitorios del volumen.
- El renderer rechaza réplica múltiple con RWO, claim/mount/path inconsistente, capacidad inválida o PVC ausente.

## C.4 Pruebas y evidencia obligatoria

Pruebas automatizadas:

- clave canónica exacta y compatibilidad/migración;
- aislamiento tenant/categoría/agregado;
- traversal, prefijo hermano, symlink/reparse y root enlazado;
- colisión diferente, idempotencia y dos escritores concurrentes desde procesos distintos;
- fallos de move, disco lleno simulado, delete/cleanup fallido y temporal huérfano;
- reinicio de servicio y lectura posterior;
- imágenes válidas completas y archivos truncados/polyglot/MIME falso/tamaño o dimensiones excesivas;
- sustitución del adaptador mediante DI sin dependencias AWS en Domain/Core;
- pruebas GitOps de StorageClass, RWO/RWX, réplicas, mounts, permisos y probes.

Evidencia Kubernetes real:

1. desplegar dos pods contra un PVC RWX de prueba;
2. escribir desde A, leer/verificar hash desde B;
3. reiniciar ambos y volver a leer;
4. ejecutar escritores concurrentes sobre la misma clave;
5. demostrar rechazo de traversal/symlink y aislamiento tenant;
6. registrar StorageClass, access modes, eventos, UID/GID y comandos sin secretos;
7. limpiar únicamente los objetos de prueba identificados.

Si no hay clúster disponible, entrega script/runbook reproducible y deja B-09 en estado parcial. No sustituyas esta evidencia con dos instancias .NET apuntando al mismo directorio temporal local.

---

# 3. Verificación final obligatoria

Ejecuta y reporta resultados reales:

## Backend

```powershell
dotnet restore MS.Cobroup.Backend.slnx
dotnet build MS.Cobroup.Backend.slnx -c Release --no-restore
dotnet test MS.Cobroup.Backend.slnx -c Release --no-build
dotnet list MS.Cobroup.Backend.slnx package --vulnerable --include-transitive
```

## BackendAdmin

```powershell
dotnet restore MS.Cobroup.BackendAdmin.slnx
dotnet build MS.Cobroup.BackendAdmin.slnx -c Release --no-restore
dotnet test MS.Cobroup.BackendAdmin.slnx -c Release --no-build
dotnet list MS.Cobroup.BackendAdmin.slnx package --vulnerable --include-transitive
```

## GitOps

```powershell
python -m unittest scripts/test_render_manifests.py -v
kubectl kustomize dev
kubectl kustomize qa
kubectl kustomize prod
```

Además:

- ejecutar `dotnet ef migrations has-pending-model-changes` en ambos backends con configuración auditora no persistida;
- ejecutar `git diff --check` en los tres repositorios;
- buscar secretos reales, PEM, RSA/JWKS runtime, placeholders y YAML renderizados;
- revisar el diff completo y confirmar que ningún frontend recibe secretos;
- demostrar que el renderer no modifica el checkout y que las carpetas SCP no contienen Secrets sensibles.

# 4. Documentación obligatoria

- Corrige las secciones activas de ambos `AGENTS.md` que todavía describen RSA/JWKS, conservando el historial append-only.
- Alinea ADR HS256, README, contratos, backlog, runbook y guía VPS.
- Elimina callbacks/configuración OIDC activa huérfana si no existe consumidor.
- Alinea la decisión PVC, ADR y README con la clave canónica real.
- Registra resultados efectivos, limitaciones y evidencia externa pendiente; no inventes despliegues ni protecciones de GitHub.

# 5. Entrega requerida

Entrega un informe final con:

1. archivos modificados por repositorio;
2. diseño final de GitHub Environments, nombres de Secrets/Variables y checklist para que el propietario los cree;
3. prueba de que ningún secreto quedó en archivos persistentes;
4. contrato exacto de rotación y evidencia k1→k2→retiro;
5. tratamiento de refresh tokens durante rotación;
6. controles B-09 implementados y evidencia PVC real o pendiente explícita;
7. resultados exactos de build, tests, EF, dependencias y GitOps;
8. estado de cada SYM-01 a SYM-10 y de B-09, B-21 y B-22;
9. riesgos residuales y cualquier paso manual requerido.

No hagas commit. No declares producción aprobada mientras falte un control bloqueante o evidencia real de GitHub/Kubernetes.
