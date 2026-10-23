# Prompt para remediar los 3 bloqueantes de la auditoría v5

Actúa como ingeniero principal de .NET, seguridad, PostgreSQL y Kubernetes. Debes implementar y demostrar la corrección de los tres hallazgos bloqueantes documentados en:

`D:\Proyectos\cobroup\cobroup.md\informe\auditoria-backend-csharp-2026-10-22-v5.html`

## Reglas de trabajo

1. Lee completamente antes de modificar:
   - Todos los `AGENTS.md` aplicables.
   - `D:\Proyectos\cobroup\cobroup.md\FRAMEWORK_DESARROLLO_CSHARP.md`.
   - `D:\Proyectos\cobroup\cobroup.md\DECISION_AUTENTICACION_JWT.md`.
   - Los contratos, ADR, README, contexto y manifiestos relacionados con identidad, archivos y despliegue.
2. Revisa `git status`, `git diff` y los últimos commits de cada repositorio. Hay trabajo sin commit: consérvalo y no reviertas cambios ajenos.
3. Trabaja únicamente en las ramas `feature/*` actuales.
4. No hagas `commit`, `push`, `merge`, `rebase`, tag ni PR.
5. No reintroduzcas OIDC, OpenIddict, Authorization Code ni PKCE. La decisión vigente es JWT exclusivo.
6. No declares un hallazgo cerrado por compilación, documentación o mocks aislados. Cada cierre requiere prueba ejecutada del comportamiento real.
7. No expongas claves privadas, JWT, cookies, cadenas de conexión ni secretos en consola, logs, archivos de evidencia o respuesta final.

Repositorios involucrados:

- `D:\Proyectos\cobroup\MS.Cobroup.Backend`
- `D:\Proyectos\cobroup\MS.Cobroup.BackendAdmin`
- `D:\Proyectos\cobroup\MF.cobroup-frontend`
- `D:\Proyectos\cobroup\MF.cobroup-frontend-admin`
- `D:\Proyectos\cobroup\cobroup.environments`
- `D:\Proyectos\cobroup\cobroup.md`

---

## Decisión nueva obligatoria: archivos en PVC con evolución futura a AWS S3

Debes dejar explícita y consistentemente registrada en los Markdown pertinentes la siguiente decisión aprobada:

> Todas las fotografías de evidencias, imágenes cargadas y cualquier archivo binario administrado por COBROUP se almacenarán actualmente en un PersistentVolumeClaim de Kubernetes. La lógica de negocio, los controladores y la base de datos no dependerán de rutas físicas de Kubernetes. El acceso se realizará mediante un puerto de almacenamiento y claves lógicas independientes del proveedor, de forma que el adaptador basado en PVC pueda sustituirse en el futuro por AWS S3 mediante configuración e inyección de dependencias, sin reescribir Core, Domain ni los contratos HTTP.

La decisión implica:

- **Proveedor actual:** volumen persistente montado mediante PVC de Kubernetes.
- **Proveedor futuro previsto:** AWS S3, sin implementarlo ahora salvo que ya exista un requerimiento autorizado.
- No instalar el SDK de AWS ni crear buckets, credenciales o infraestructura S3 ficticia en esta remediación.
- Las clases deben ser suficientemente desacopladas para agregar un adaptador S3 posteriormente.
- La base de datos debe guardar una clave lógica relativa y segura, no una ruta absoluta del pod, una URL pública ni detalles del StorageClass.
- Ningún archivo puede depender del filesystem efímero de la imagen o del pod.
- El volumen debe ser compartido por todas las réplicas que atienden el mismo conjunto de archivos. Para producción con dos réplicas, el PVC debe soportar el modo de acceso requerido —normalmente `ReadWriteMany`—. No declares conformidad si el StorageClass real no ofrece ese modo.
- Las fotografías y archivos permanecen privados; se entregan únicamente a través de endpoints autorizados. No se publican URLs directas del volumen.

Actualiza, como mínimo y según corresponda tras inspeccionar la estructura real:

- `cobroup.md/COBROUP.md`.
- Un ADR/decisión específica, por ejemplo `cobroup.md/DECISION_ALMACENAMIENTO_ARCHIVOS.md`, enlazada desde el índice de contexto.
- `MS.Cobroup.Backend/README.md`.
- `MS.Cobroup.Backend/docs/context/INDEX.md`.
- Contratos y decisiones de evidencias/archivos del backend operativo.
- Roadmap o propuesta de fase donde se defina almacenamiento y despliegue.
- Documentación de ambientes en `cobroup.environments`.
- Todos los `AGENTS.md` afectados, exclusivamente con una nueva entrada append-only que registre esta solicitud, decisiones, archivos y resultados reales.

No copies información obsoleta ni escribas que S3 ya está implementado. Debe quedar claro: **PVC implementado ahora; S3 es una sustitución futura prevista por arquitectura**.

---

## Bloqueante 1 — B-09: almacenamiento durable de evidencias y archivos

### Problema comprobado

`MS.Cobroup.Backend.Api/Program.cs` exige `Storage:EvidencePath` fuera de Development, pero los manifiestos dev/QA/prod no configuran la ruta, no montan un PVC y producción tiene dos réplicas. El runtime registra `LocalFileStorageService`, por lo que actualmente el pod puede fallar al iniciar o almacenar archivos de manera efímera/no compartida.

### Implementación obligatoria

1. Conserva la arquitectura hexagonal:
   - El puerto debe permanecer en Domain, por ejemplo `IFileStorageService`.
   - Core solo opera con streams y claves lógicas.
   - Infrastructure implementa el adaptador de filesystem montado en PVC.
   - API únicamente selecciona/configura el proveedor mediante DI.
2. Revisa el puerto actual y hazlo agnóstico del proveedor:
   - Operaciones asíncronas de guardar, abrir/leer y eliminar.
   - `CancellationToken` obligatorio.
   - Entrada mediante `Stream`; no cargar archivos completos en memoria.
   - Resultado con clave lógica, tamaño, MIME validado y hash; nunca ruta absoluta interna.
   - Evitar nombres o tipos como `PvcStorageService` dentro de Core/Domain. La dependencia del PVC pertenece al adaptador/configuración de infraestructura.
3. Implementa un adaptador filesystem durable, por ejemplo `FileSystemObjectStorage`, cuyo root procede de opciones tipadas y apunta al mount del PVC.
4. Diseña la selección del proveedor mediante configuración explícita, por ejemplo:
   - `Storage:Provider=FileSystem` para el PVC actual.
   - Una extensión futura `Storage:Provider=S3` debe poder registrar otro adaptador sin cambiar casos de uso/controladores.
   - Un proveedor desconocido debe provocar fail-fast fuera de pruebas.
5. Las claves persistidas deben ser normalizadas, relativas y libres de traversal. Rechaza `..`, rutas absolutas, separadores inválidos, symlinks que escapen del root y nombres suministrados directamente por el cliente.
6. Mantén los controles actuales: límite durante streaming, magic bytes, MIME permitido, hash incremental, compensación si falla la persistencia y autorización tenant/ruta antes de descargar o borrar.
7. Define comportamiento ante colisión, escritura parcial y cancelación. Escribe primero a un archivo temporal dentro del mismo volumen y publica mediante operación atómica cuando sea posible. Limpia temporales sin borrar archivos válidos.
8. Configura en Kubernetes para dev, QA y producción:
   - `PersistentVolumeClaim` por ambiente.
   - Mount consistente en el Deployment del backend operativo.
   - `Storage__Provider` y `Storage__EvidencePath`/root coherentes con el mount.
   - Capacidad solicitada, access mode y StorageClass configurables/documentados.
   - Producción debe soportar acceso compartido por sus dos réplicas.
   - SecurityContext y permisos mínimos compatibles con el usuario no-root del contenedor.
9. No almacenes la clave privada JWT, cadenas de conexión u otros secretos dentro de este PVC.

### Pruebas obligatorias de B-09

- Unitarias de normalización y rechazo de traversal/rutas absolutas.
- Integración de streaming, límites, MIME/magic bytes, hash, cancelación, colisión y limpieza tras fallo.
- Escribir mediante una instancia del adaptador y leer mediante otra apuntando al mismo root, simulando réplicas distintas.
- Demostrar persistencia después de recrear el servicio/adaptador.
- Validar que la base de datos guarda una clave relativa, no el path físico.
- `kubectl kustomize` de dev/QA/prod y validación post-render de los PVC, mounts y variables.
- Si existe cluster de prueba accesible, crear archivo, reiniciar pod y leerlo desde otra réplica. Si no existe, reportar esa prueba como **no ejecutada**, nunca como aprobada.

### Criterio de cierre de B-09

Solo se cierra cuando el backend puede iniciar en los tres ambientes, todas las réplicas usan almacenamiento durable compartido y el código de Core/API no queda acoplado a Kubernetes. La posibilidad futura de S3 debe demostrarse por sustitución del adaptador en una prueba de DI o doble de almacenamiento, no por instalar AWS prematuramente.

---

## Bloqueante 2 — B-17: cambio de organización sin revocación confirmada

### Problema comprobado

`OrganizationSessionsController` permite que la cookie refresh sea opcional. `RevokeFamilyByTokenAsync` no devuelve resultado y oculta token inválido/no encontrado; acto seguido se emite una nueva familia. Un access JWT válido puede convertirse en una sesión refresh nueva sin demostrar que la familia anterior fue validada y revocada.

### Implementación obligatoria

1. `PUT /api/v1/auth/organizations/current` debe exigir simultáneamente:
   - Access JWT válido.
   - Cookie `__Host-cobroup-refresh` presente.
   - Antiforgery válido.
   - Refresh JWT criptográficamente válido, `typ=refresh`, audience `cobroup-refresh`, RS256 y `kid` autorizado.
   - Registro hash existente y activo en PostgreSQL.
   - Coincidencia entre `sub` del access, `sub` del refresh y usuario solicitado.
   - Coincidencia entre tenant actual del access, organización del refresh y familia persistida.
   - Membresía destino activa y organización destino activa.
2. Si falta o falla cualquiera de esas condiciones:
   - No emitir access ni refresh nuevos.
   - No crear una familia nueva.
   - Responder `401` o `403` según corresponda, sin filtrar detalles sensibles.
   - Expirar la cookie cuando sea inseguro conservarla.
3. Cambia el contrato de revocación para devolver un resultado tipado verificable. No uses `void` lógico ni ignores “no encontrado”.
4. La revocación de la familia anterior y la persistencia del primer refresh de la familia nueva deben ser una operación consistente en PostgreSQL. Implementa transacción en Infrastructure; no pongas SQL ni reglas de sesión en el controller.
5. Genera los JWT fuera o dentro de la orquestación según convenga, pero no envíes la cookie/respuesta hasta confirmar commit de base de datos.
6. Ante excepción o rollback, la familia anterior no debe quedar en un estado ambiguo y no debe existir una nueva familia utilizable.
7. Registra evento de seguridad/auditoría del cambio exitoso y fallido sin token completo, cookie ni hash sensible.

### Pruebas obligatorias de B-17

- Cambio correcto revoca toda la familia anterior y crea exactamente una familia nueva.
- El refresh anterior ya no puede rotar después del cambio.
- Access válido sin cookie refresh no obtiene sesión nueva.
- Cookie refresh inválida, expirada, revocada, de otra familia, usuario u organización no obtiene sesión nueva.
- Antiforgery ausente/inválido no muta ninguna familia.
- Membresía destino inactiva u organización suspendida no muta ninguna familia.
- Fallo simulado entre revocación y creación produce rollback verificable.
- Dos cambios concurrentes no crean dos familias activas.
- Prueba HTTP completa con `WebApplicationFactory` y prueba de integración PostgreSQL real. No retornar silenciosamente ni contaminar una base compartida.
- El test debe usar base efímera/contenedor dedicado o cleanup garantizado en `finally`/fixture. Corrige también la contaminación detectada en `AuthLifecycleHttpTests`.

### Criterio de cierre de B-17

El endpoint no debe poder convertir un access token aislado en una sesión persistente. Debe existir evidencia de atomicidad, revocación anterior y unicidad de la nueva familia.

---

## Bloqueante 3 — B-13: workflow de manifiestos depende de PyYAML no instalado

### Problema comprobado

Los workflows `deploy-dev.yml`, `deploy-qa.yml` y `deploy-prod.yml` ejecutan `import yaml` después de `actions/setup-python`, pero no instalan ni fijan PyYAML. En un Python limpio se obtiene `ModuleNotFoundError`.

### Implementación obligatoria

1. Elimina la duplicación de tres scripts inline. Extrae un script versionado y probado en `cobroup.environments`, por ejemplo `scripts/render_manifests.py`.
2. Declara sus dependencias reproducibles:
   - Preferencia: archivo de requirements dedicado con versión fijada y, si el flujo actual lo permite, hashes verificados.
   - Instala con `python -m pip`, fail-fast y sin depender de paquetes preinstalados del runner.
   - También es válido eliminar PyYAML y usar una alternativa ya declarada, siempre que valide realmente YAML y esté demostrada en CI.
3. El render debe aceptar PEM multilínea o base64, normalizarlo y colocarlo bajo bloque literal correctamente indentado.
4. Valida antes de copiar/desplegar:
   - Todas las variables requeridas no vacías.
   - Todos los placeholders de cualquier tipo, no solo `Jwt__*`.
   - Todos los YAML del ambiente, hayan cambiado o no durante sustitución.
   - `kubectl kustomize` del ambiente completo.
   - Estructura Kubernetes post-render y presencia de PVC/mount/configuración de almacenamiento.
5. Usa archivos temporales seguros y evita imprimir valores sustituidos. Los mensajes solo deben mostrar nombre de variable/archivo y estado.
6. El pipeline debe detenerse antes de SCP/SSH ante secreto ausente, PEM inválido, YAML inválido, placeholder pendiente, PVC/mount ausente o kustomization inválida.
7. Mantén comportamiento equivalente para dev, QA y producción mediante parámetros, no tres implementaciones copiadas.
8. Fija versiones de Actions por SHA o documenta la excepción; no empeores la trazabilidad de supply chain.

### Pruebas obligatorias de B-13

- Unitarias del renderer con PEM multilínea, PEM base64, valores vacíos, placeholder desconocido y YAML malformado.
- Caso válido produce YAML parseable sin exponer el PEM.
- Caso sin PyYAML instalado instala sus dependencias declaradas y funciona desde cero.
- `kubectl kustomize dev`, `qa` y `prod` después del render sintético.
- Prueba negativa confirma que el workflow se detiene antes del paso remoto.
- Si no puede ejecutarse GitHub Actions localmente, ejecuta el mismo script y comandos en un entorno Python limpio y registra esa limitación.

### Criterio de cierre de B-13

El workflow debe ser autosuficiente, reproducible y fallar antes de cualquier transferencia remota cuando el render no sea válido. Corregir solo el bloque PEM sin arreglar la dependencia y la validación integral no cierra el hallazgo.

---

## Gates de verificación obligatorios

Ejecuta y registra resultados reales:

```text
dotnet build D:\Proyectos\cobroup\MS.Cobroup.Backend\MS.Cobroup.Backend.slnx -c Release --no-incremental
dotnet test D:\Proyectos\cobroup\MS.Cobroup.Backend\MS.Cobroup.Backend.slnx -c Release
dotnet build D:\Proyectos\cobroup\MS.Cobroup.BackendAdmin\MS.Cobroup.BackendAdmin.slnx -c Release --no-incremental
dotnet test D:\Proyectos\cobroup\MS.Cobroup.BackendAdmin\MS.Cobroup.BackendAdmin.slnx -c Release
dotnet ef migrations has-pending-model-changes ...
dotnet list <solución> package --vulnerable --include-transitive
npm test -- --watch=false
npm run build
npm audit --omit=dev
kubectl kustomize dev
kubectl kustomize qa
kubectl kustomize prod
git diff --check
```

Ejecuta npm en ambos frontends si modificas contratos de sesión. Ejecuta pruebas PostgreSQL contra una base de integración aislada. No ejecutes build y test simultáneamente sobre la misma solución/`obj`.

Además debes comprobar:

- Ningún token en `localStorage`, `sessionStorage`, IndexedDB, URL o logs.
- Ningún refresh token en JSON.
- Solo una cookie `__Host-cobroup-refresh` con atributos correctos al crear, rotar y eliminar.
- Ningún paquete o endpoint OIDC/OpenIddict/PKCE activo.
- Ninguna clave privada o secreto versionado.
- Ningún fixture de pruebas abandonado en bases compartidas.
- El backend operativo inicia con la configuración renderizada de cada ambiente.

---

## Formato obligatorio de la entrega

Entrega una tabla con:

`Hallazgo | archivos modificados | cambio implementado | pruebas ejecutadas | resultado | estado`

Incluye además:

1. Resumen por repositorio.
2. Comandos exactos y número real de pruebas.
3. Evidencia sanitizada de PVC, mounts, claves lógicas y headers de cookie.
4. Evidencia PostgreSQL del cambio de organización atómico, sin tokens ni hashes completos.
5. Evidencia de ejecución del renderer en entorno Python limpio.
6. Lista de Markdown actualizados y texto de la decisión PVC→S3 incorporado.
7. `git diff --stat` y `git status --short` finales.
8. Hallazgos cerrados, parciales y abiertos. No llames “cerrado” a un punto con pruebas no ejecutadas.
9. Confirmación expresa: sin OIDC/OpenIddict/PKCE, sin secretos expuestos y sin commit/push.

Prioriza cerrar completamente estos tres bloqueantes. No aproveches la tarea para reestructuraciones generales ni para intentar resolver todos los hallazgos importantes de la v5.
