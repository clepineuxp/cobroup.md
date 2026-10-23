# Decisión de arquitectura — Almacenamiento durable de archivos en Kubernetes PVC con evolución futura a AWS S3

> **Estado:** APROBADA por el propietario del producto  
> **Fecha:** 2026-10-22  
> **Vigencia:** inmediata para arquitectura, backend operativo (`MS.Cobroup.Backend`), manifiestos de Kubernetes y auditoría.  
> **Autoridad:** este documento complementa y rige sobre las decisiones de almacenamiento en `COBROUP.md`, `README.md`, manifiestos de `cobroup.environments` y `AGENTS.md`.

---

## 1. Decisión cerrada aprobada

> **Todas las fotografías de evidencias, imágenes cargadas y cualquier archivo binario administrado por COBROUP se almacenarán actualmente en un PersistentVolumeClaim de Kubernetes. La lógica de negocio, los controladores y la base de datos no dependerán de rutas físicas de Kubernetes. El acceso se realizará mediante un puerto de almacenamiento y claves lógicas independientes del proveedor, de forma que el adaptador basado en PVC pueda sustituirse en el futuro por AWS S3 mediante configuración e inyección de dependencias, sin reescribir Core, Domain ni los contratos HTTP.**

---

## 2. Principios y reglas de implementación

### 2.1 Proveedor actual y proveedor futuro previsto
- **Proveedor actual:** Volumen persistente montado mediante un `PersistentVolumeClaim` (PVC) de Kubernetes en el contenedor del microservicio operativo `MS.Cobroup.Backend`.
- **Proveedor futuro previsto:** Almacenamiento de objetos en la nube con **AWS S3**, sin implementarlo ni instalar su SDK en la presente fase.
- **Regla estricta:** No instalar el SDK de AWS ni crear buckets, credenciales o infraestructura S3 ficticia en esta remediación. El desacoplamiento debe garantizarse mediante la abstracción del puerto de dominio y la inyección de dependencias.

### 2.2 Desacoplamiento hexagonal y clave lógica
- **Puerto de Dominio:** `IFileStorageService` reside en `Domain.Interfaces.Services`. Core y Domain operan exclusivamente con streams de datos (`Stream`), metadatos (`ContentType`, `ByteSize`, `FileHashSha256`) y una **clave lógica relativa** (`StorageKey`), nunca con rutas físicas del host, nombres de pods ni detalles de Kubernetes.
- **Sin acoplamiento nominal:** Ninguna clase o interfaz de Domain o Core contendrá nombres o prefijos dependientes del proveedor como `PvcStorageService` o `S3StorageService`. El adaptador concreto reside en `Infrastructure.Storage`.
- **Persistencia en Base de Datos:** Las tablas de base de datos (como `payment_evidences.storage_path`) almacenan únicamente una clave lógica relativa normalizada (por ejemplo: `tenants/{tenantId}/payments/{paymentId}/{fileHash}.jpg`), libre de rutas absolutas del pod (`/var/cobroup/...`, `C:\...`), URLs públicas o identificadores de StorageClass.
- **Privacidad estricta:** Todos los archivos permanecen privados. Ningún archivo binario o fotografía se expone mediante URLs públicas directas del volumen. La entrega se realiza única y exclusivamente a través de endpoints autorizados de la API que validan tenant, pertenencia a la ruta y rol del usuario solicitante.

### 2.3 Concurrencia y réplicas en Kubernetes
- Ningún archivo puede depender del filesystem efímero del contenedor o del pod.
- El volumen debe ser compartido entre todas las réplicas del microservicio que procesan archivos.
- En **producción**, dado que el despliegue cuenta con dos (2) réplicas activas (`replicas: 2`), el PVC debe soportar el modo de acceso compartido concurrente: **`ReadWriteMany` (RWX)** provisto por un StorageClass compartido (e.g., NFS, AWS EFS, Azure Files o CephFS).
- En **desarrollo (dev)** y **QA**, donde opera una única réplica (`replicas: 1`), el modo `ReadWriteOnce` (RWO) es admitido operativamente, aunque se recomienda alineación con RWX.

---

## 3. Matriz de configuración por ambiente

La selección del adaptador se realiza por configuración tipada mediante `Storage:Provider`:

```json
{
  "Storage": {
    "Provider": "FileSystem",
    "EvidencePath": "/var/cobroup/evidences",
    "MaxFileSizeBytes": 10485760
  }
}
```

| Ambiente | Proveedor (`Storage:Provider`) | Montaje de Volumen (`Storage:EvidencePath`) | Capacidad PVC | Modo de Acceso | Réplicas |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Development (local)** | `FileSystem` | `storage/evidences` (relativo a app) | N/A (local dev) | Local | 1 |
| **Kubernetes Dev** | `FileSystem` | `/var/cobroup/evidences` | `1Gi` | `ReadWriteOnce` | 1 |
| **Kubernetes QA** | `FileSystem` | `/var/cobroup/evidences` | `5Gi` | `ReadWriteOnce` | 1 |
| **Kubernetes Prod** | `FileSystem` | `/var/cobroup/evidences` | `10Gi` | `ReadWriteMany` | 2 |
| **Futuro (S3)** | `S3` *(previsto)* | `s3://[bucket-name]/[prefix]` | N/A (S3 Bucket) | Cloud Managed | N réplicas |

Cualquier valor desconocido para `Storage:Provider` (o la ausencia de `Storage:EvidencePath` fuera de Development) provoca **fail-fast inmediato** al iniciar el microservicio. Si se configura `Storage:Provider=S3` en la versión actual, el contenedor fallará explícitamente indicando que el adaptador S3 está previsto para una fase futura.

---

## 4. Controles de seguridad de archivos en el adaptador

1. **Prevención de Directory Traversal:** Se rechaza de forma estricta cualquier clave que contenga secuencias `..`, caracteres `:`, barras iniciales absolutas, caracteres nulos o nombres suministrados directamente por el cliente. La ruta resultante se resuelve canónicamente con `Path.GetFullPath` y debe pertenecer estrictamente al subárbol del `RootPath`.
2. **Validación de Magic Bytes y Formato:** Se inspeccionan los bytes iniciales del stream en el primer fragmento (sin cargar el archivo completo en memoria) para validar firmas válidas de imagen:
   - JPEG: `FF D8 FF`
   - PNG: `89 50 4E 47 0D 0A 1A 0A`
   - WebP: `RIFF....WEBP`
   Formatos ejecutables, scripts o documentos ajenos son rechazados con error de validación.
3. **Escritura Atómica y Limpieza:** El stream se escribe primero a un archivo temporal dentro del mismo volumen persistente (`.tmp/`). Una vez completado el hashing SHA-256 y verificados los límites de tamaño (máximo 10 MB), se realiza una operación atómica `File.Move(..., overwrite: true)` a la ubicación final indexada por hash (`{hashHex}.{ext}`). Ante cualquier error o cancelación, el archivo temporal es eliminado en el bloque `finally`.
4. **Seguridad del Contenedor:** Los manifiestos de Kubernetes configuran `securityContext: fsGroup: 1000` y `runAsUser: 1000` (compatible con el usuario no-root `appuser` definido en el Dockerfile) para asegurar permisos de lectura/escritura en el volumen persistente.
5. **Aislamiento de Secretos:** Queda prohibido almacenar claves privadas JWT, credenciales o cadenas de conexión dentro de este volumen de evidencias.
