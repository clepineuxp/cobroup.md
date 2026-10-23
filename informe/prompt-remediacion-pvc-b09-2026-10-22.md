# Prompt para Gemini — Remediación posterior de PVC y cierre de B-09

Actúa como **arquitecto senior .NET, especialista en almacenamiento seguro y Kubernetes**. Este trabajo se ejecuta **después** de terminar y validar la migración JWT HS256. No reviertas, debilites ni mezcles los cambios de autenticación.

## Contexto y alcance

Trabaja principalmente en:

- `MS.Cobroup.Backend`: puerto y adaptador de almacenamiento, casos de uso y pruebas.
- `cobroup.environments`: PVC, mounts, permisos, probes y evidencia de despliegue.
- `cobroup.md`: decisión y documentación normativa.

Lee `DECISION_ALMACENAMIENTO_ARCHIVOS.md`, el ADR de almacenamiento durable, el informe de auditoría más reciente y los `AGENTS.md` aplicables. Respeta el framework C# y la arquitectura hexagonal.

Hay cambios locales sin commit. Presérvalos. No hagas commit, push, merge, rebase, tag ni PR.

## Objetivo obligatorio

Cerrar técnicamente el hallazgo B-09: fotografías de evidencia, imágenes y todos los archivos binarios deben persistirse actualmente en un PVC Kubernetes, con una abstracción suficientemente versátil para cambiar a AWS S3 en el futuro mediante configuración e inyección de dependencias, sin reescribir Domain/Core ni contratos HTTP.

No implementes S3 ahora. Sí deja el puerto, metadatos, claves y composición compatibles con un adaptador S3 posterior.

## Contrato de almacenamiento

- Generaliza el puerto para objetos binarios; no lo acoples solo a `PaymentEvidence` ni a rutas físicas.
- Persiste en base de datos únicamente una clave relativa canónica, nunca path absoluto, URL firmada ni nombre de PVC.
- Formato normativo: `tenants/{tenantId}/{category}/{aggregateId}/{sha256}.{ext}`.
- `category` debe provenir de una allowlist de categorías de negocio, no de texto libre del cliente.
- Conserva metadatos mínimos: hash SHA-256, tamaño, MIME detectado, extensión canónica, tenant, agregado, fecha UTC y estado cuando aplique.
- El adaptador filesystem resuelve la clave bajo un root configurado, actualmente `/var/cobroup/files`; documenta cualquier compatibilidad temporal con `/var/cobroup/evidences`.

## Seguridad filesystem obligatoria

Implementa y prueba todos estos controles:

1. Rechaza rutas absolutas, `..`, `.`, segmentos vacíos, separadores alternativos, NUL, `:`, caracteres inválidos y claves no canónicas.
2. Usa comparación de pertenencia al root con límites de segmento/path separator; un prefijo textual similar no es suficiente.
3. Rechaza symlinks y reparse points en el root y en **cada componente existente** del camino. Evita TOCTOU en la medida permitida por .NET/Linux y documenta el límite residual.
4. Crea directorios con permisos mínimos para usuario no-root y valida permisos/propietario al iniciar.
5. Escribe en un temporal dentro del mismo filesystem, hace flush cuando corresponda y publica atómicamente sin sobrescritura insegura.
6. Si el destino ya existe, verifica tamaño y SHA-256: mismo contenido = operación idempotente; contenido diferente = colisión/integridad y error explícito.
7. Soporta concurrencia real entre procesos/réplicas sobre RWX, no solo locks en memoria.
8. No ignores excepciones de move, delete, cleanup ni dispose. Informa errores accionables y logging estructurado sin datos sensibles; evita temporales huérfanos o añade reconciliación segura.
9. Streaming con límites antes y durante la copia; no cargues el archivo completo en memoria.
10. Para imágenes admitidas, valida contenido completo/decodificación segura, dimensiones, MIME y magic bytes; un encabezado parcial no basta. Rechaza archivos truncados, polyglot conocidos y decompression bombs mediante límites.

## Diseño para S3 futuro

- El puerto debe expresar `Put/Get/Delete/Exists` o equivalentes mediante clave lógica y streams, sin `FileInfo`, paths locales ni tipos AWS.
- La selección `Storage:Provider=FileSystem|S3` debe ocurrir en composición/infraestructura.
- Domain/Core no dependen de SDK AWS.
- Define semántica estable de idempotencia, integridad, errores y metadatos que un adaptador S3 pueda cumplir.
- Documenta cómo la clave relativa se mapeará directamente a object key S3 y cómo se sustituirá el adaptador; no agregues credenciales ni código S3 prematuro.

## Kubernetes y GitOps

- Define un PVC por ambiente y monta exactamente el mismo path configurado por la aplicación.
- Dev/QA pueden usar RWO si mantienen una sola réplica; producción con dos o más réplicas exige RWX y una StorageClass que realmente lo soporte.
- Ejecuta como no-root y establece `runAsUser`, `runAsGroup`/`fsGroup`, `readOnlyRootFilesystem` cuando sea viable y permisos mínimos del volumen.
- Readiness debe comprobar que el root está montado, accesible y escribible sin dejar basura; liveness no debe provocar ciclos por fallos transitorios de almacenamiento.
- El render debe fallar si hay réplicas múltiples con RWO, mount/path inconsistente, PVC ausente, provider no soportado o capacidad inválida.
- No versiones credenciales S3. S3 sigue fuera de alcance de esta iteración.

## Pruebas automatizadas obligatorias

Cubre como mínimo:

- store/read/delete/idempotencia y hash correcto;
- aislamiento por tenant/categoría/agregado;
- traversal, absoluto, separadores, NUL, prefijo hermano y clave malformada;
- symlink/reparse en root y en cada nivel intermedio;
- colisión con contenido diferente;
- dos escritores concurrentes desde instancias separadas;
- fallo de move/rename, disco lleno simulado, cleanup fallido y temporal huérfano;
- reinicio del proceso y lectura posterior;
- imágenes JPEG/PNG/WebP válidas y truncadas, MIME falso, exceso de tamaño/dimensiones;
- sustitución del adaptador mediante DI con un fake compatible con futura implementación S3;
- render GitOps positivo/negativo para RWO/RWX, réplicas, mounts y paths.

Evita tests que pasen únicamente porque usan un lock estático dentro del mismo proceso.

## Evidencia operativa obligatoria

Además de pruebas locales, prepara y ejecuta cuando exista clúster autorizado una validación Kubernetes reproducible:

1. desplegar dos pods contra el PVC RWX de prueba;
2. escribir desde el pod A y leer/verificar hash desde B;
3. reiniciar ambos pods y volver a leer;
4. demostrar aislamiento tenant y rechazo de traversal/symlink;
5. capturar comandos, resultados, StorageClass, access modes, UID/GID y eventos;
6. limpiar solo los objetos de prueba identificados.

Si no hay clúster disponible, no simules ni marques esta evidencia como aprobada: deja script/runbook reproducible y B-09 en estado parcial pendiente de prueba real.

## Documentación y consistencia

- Alinea `DECISION_ALMACENAMIENTO_ARCHIVOS.md`, ADR, README, configuración y manifests con el formato de clave exacto y el root real.
- Elimina contradicciones como `{tenantId}/{paymentId}/...` si la norma vigente usa `tenants/{tenantId}/{category}/{aggregateId}/...`.
- Conserva explícita la estrategia futura S3 sin presentarla como implementada.

## Verificación final y entrega

Ejecuta build, suite completa, pruebas de integración/filesystem, pruebas GitOps, `git diff --check` y revisión del diff. Reporta:

1. archivos modificados;
2. controles B-09 implementados y evidencia asociada;
3. resultados exactos de pruebas;
4. evidencia Kubernetes real o declaración explícita de pendiente;
5. riesgos residuales y limitaciones del filesystem/RWX;
6. preparación para S3 sin afirmar que S3 está implementado.

No hagas commit. No declares B-09 cerrado sin evidencia Kubernetes real y reproducible.
