# Informe de avance — COBROUP

> **Versión del informe:** 2.1.0  
> **Fecha de corte:** 2026-10-22  
> **Documento maestro:** `../COBROUP.md` 1.7.0 y `../DECISION_AUTENTICACION_JWT.md`  
> **Rama de trabajo vigente:** `feature/solucion-hallazgos-iniciales` en backends; cambios de autenticación aún sin commit  
> **Alcance del corte:** la decisión vigente establece autenticación exclusiva con access y refresh JWT. La implementación se encuentra reabierta para adecuación y auditoría; no está aprobada para commit, merge ni despliegue. La evidencia histórica de identidad quedó supersedida.

## 1. Resumen ejecutivo

COBROUP conserva la funcionalidad versionada de sus nueve fases, pero la identidad fue reabierta por decisión del propietario para usar exclusivamente JWT. El nuevo contrato exige firma asimétrica, issuer/audience/tipo estrictos, access corto en memoria, refresh JWT en cookie HttpOnly con hash server-side, rotación atómica, detección de replay, revocación por familia y publicación JWKS. El código actual aún no cumple todo el contrato y la auditoría vigente registra bloqueantes; por ello no se declara 100% ni disponibilidad productiva.

| Indicador | Estado |
|---|---:|
| Fases totales de la línea base | 9 |
| Fases funcionales versionadas | 9 |
| Fases reabiertas por decisión de arquitectura | 1 (identidad JWT) |
| Auditoría JWT | NO APROBADA |
| Avance de adecuación JWT | Pendiente de evidencia |
| Repositorios funcionales involucrados | 4 |
| Bloqueos de auditoría vigentes | Ver auditoría 2026-10-22 v2 |
| Pendientes técnicos de identidad | Firma, validación, almacenamiento, rotación, ingress, migración y pruebas |

Los cierres históricos de fases no certifican el nuevo contrato JWT. La aprobación requiere una nueva evidencia completa contra `DECISION_AUTENTICACION_JWT.md` y los frameworks vigentes.

### Estado consolidado

| Fase | Nombre | Estado | Resultado principal |
|---:|---|---|---|
| 1 | Fundación y scaffolding | **COMPLETA** | Cuatro repositorios independientes, compilables, documentados y con pruebas base |
| 2 | Identidad y aislamiento tenant | **REABIERTA — JWT** | Autoridad JWT única; implementación y pruebas pendientes de conformidad |
| 3 | Configuración y rutas | **CERRADA** | Configuración organizacional, rutas, asignaciones, cupos heredables y acceso operativo filtrado |
| 4 | Clientes y alcance operativo | **FUNCIONAL; REVALIDAR IDENTIDAD** | Clientes tenant-safe y auditoría; integración debe repetirse con JWT conforme |
| 5 | Plantillas y excepciones | **CERRADA EN GIT** | Catálogo versionado, inmutabilidad, solicitudes de excepción y decisión exclusiva Admin (AC-07, AC-08, AC-09); commits `8cc1e42`, `b3b8e34`, `b6594be`, `674e62b` |
| 6 | Préstamos y cronograma | **CERRADA EN GIT** | Originación, snapshot inmutable de términos y plantilla (AC-09), cronograma decimal con ajuste legal exacto (AC-10), cupo de ruta y vistas en ambas PWA; commits `2537925`, `57d45d7`, `762847f`, `b1f7f48` |
| 7 | Pagos, evidencias y conciliación | **CERRADA EN GIT** | Pagos ordinarios/parciales en cascada (AC-11), idempotencia (AC-12), medios de pago autorizados (AC-13), soporte de fotografía auditable admin-only (AC-14), y conciliación diaria (AC-17); commits `03b2748`, `6d563c9`, `f547dd0`, `e80f442` |
| 8 | Mapas, georreferenciación y analítica | **CERRADA EN GIT** | Agenda diaria con vencimientos exactos (AC-16), optimización GPS (Haversine), mapa operativo y supervisión administrativa con OpenStreetMap, analítica ejecutiva y aging report; commits `cf65b00`, `ed609a5`, `ddfe552`, `db29c9a` |
| 9 | Endurecimiento y despliegue | **CERRADA EN GIT** | Health checks de base de datos, seguridad HTTP, Docker multi-stage, manifiestos K3s (`cobroup-dev`, `cobroup-qa`, `cobroup-prod`), CI/CD GitHub Actions con GHCR y SSH, y servicios PWA; commits `e02bd65`, `91152b1`, `c3f007f`, `d218f03` |

## 2. Repositorios y estado actual

| Repositorio | Responsabilidad actual | Fase 9 | Evidencia más reciente |
|---|---|---|---|
| `MS.Cobroup.BackendAdmin` | Autoridad JWT, administración, supervisión, analítica y auditoría | Reabierta | Build Release correcto y 61/61 pruebas; migración pendiente; emisión/refresh JWT sin commit y no conforme aún con firma asimétrica, cookie segura, hash, rotación y JWKS |
| `MS.Cobroup.Backend` | Operaciones de clientes, préstamos, cobranza, agenda Haversine, analítica, health checks y contenedor | Revalidar JWT | Build Release correcto y 106/106 pruebas; validación JWT estricta y consumo de claves públicas pendientes |
| `MF.cobroup-frontend-admin` | PWA administrativa con supervisión en mapa, dashboard de analítica, servicios PWA y Nginx | Revalidar JWT | Build productivo correcto; 30/31 pruebas Vitest; una prueba del shell falla y el refresh en Web Storage no es conforme |
| `MF.cobroup-frontend` | PWA operativa con agenda diaria, reordenamiento GPS, mapa de ruta, cobro, servicios PWA y Nginx | Revalidar JWT | Build productivo correcto; 27/28 pruebas Vitest; una prueba del shell falla y el refresh en Web Storage no es conforme |

Los backends operan en `feature/solucion-hallazgos-iniciales`; los frontends conservan su rama previa. La segunda ronda JWT está sin commit y no tiene aceptación auditora. No se ha autorizado push, merge, tag ni pull request.

## 3. Fase 1 — Fundación y scaffolding

**Estado:** COMPLETA  
**Objetivo:** preparar los cuatro repositorios con la arquitectura, herramientas, documentación y validaciones mínimas para desarrollar el producto por fases sin introducir lógica ficticia.

### 3.1 Completado

- Se crearon exactamente dos microservicios .NET y dos PWA Angular independientes.
- Los backend adoptaron arquitectura hexagonal con proyectos `Domain`, `Core`, `Shared`, `Infrastructure`, `Api` y pruebas.
- Los frontend adoptaron estructura feature-based, componentes standalone, SCSS, temas claro/oscuro y base PWA.
- Cada API incorporó health check y configuración por entorno.
- Se crearon `README.md`, `.gitignore`, `AGENTS.md`, contexto, contratos, ADR y roadmap en cada repositorio.
- Se establecieron las fronteras de propiedad entre BackendAdmin y Backend operativo.
- Se validaron restore/build/tests y arranque temporal de las cuatro aplicaciones.
- Se renombraron y redirigieron los backend a:
  - `https://github.com/clepineuxp/MS.Cobroup.Backend.git`
  - `https://github.com/clepineuxp/MS.Cobroup.BackendAdmin.git`
- La Fase 1 cuenta con commits de cierre autorizados.

### 3.2 Evidencia de cierre

- Backend operativo inicial: build limpio, 5/5 pruebas y `/health` 200.
- Backend administrativo inicial: build limpio, 4/4 pruebas y `/health` 200.
- Frontend operativo inicial: build PWA, 4/4 pruebas y servidor HTTP 200.
- Frontend administrativo inicial: build PWA, 4/4 pruebas y servidor HTTP 200.
- Escaneos iniciales sin secretos conocidos.

### 3.3 Qué falta relacionado con Fase 1

No falta ningún criterio funcional para considerar cerrada la fase. Permanecen tareas de madurez que no reabren su alcance:

1. Reemplazar los iconos PWA generados por Angular cuando exista una identidad visual COBROUP aprobada.
2. Revisar periódicamente las vulnerabilidades de herramientas de desarrollo Angular. En el corte inicial `npm audit` reportó cuatro altas en dependencias de desarrollo, mientras `npm audit --omit=dev` reportó cero; no se aplicó una actualización forzada incompatible.
3. Realizar una auditoría técnica independiente antes de producción. Hasta este corte no se ha ejecutado.

## 4. Fase 2 — Identidad y aislamiento tenant

**Estado:** COMPLETA  
**Objetivo:** disponer de una sola autoridad de identidad, sesiones seguras y selección de organización sin duplicar usuarios o contraseñas entre servicios.

### 4.1 Completado

- `MS.Cobroup.BackendAdmin` quedó como autoridad lógica única de identidad y emisión de tokens.
- PostgreSQL y las migraciones de identidad fueron configurados y aplicados.
- Se aprobó el contrato exclusivo de access y refresh JWT documentado en `DECISION_AUTENTICACION_JWT.md`; la implementación anterior quedó retirada de la planeación vigente.
- Los secretos locales permanecen en User Secrets; no están versionados ni fueron impresos.
- El bootstrap local creó exactamente una organización `COBROUP` y un usuario para cada rol: `Admin`, `Supervisor` y `Collector`.
- El bootstrap es transaccional, auditable y de única ejecución; la repetición devolvió conflicto.
- El usuario es una identidad global y sus roles viven en membresías por organización.
- El login solicita solo usuario y contraseña.
- La organización inicial se resuelve por la última membresía activa accedida; el primer acceso desempata por incorporación más reciente.
- El cambio de organización actualiza el último acceso, revoca la sesión anterior y reemite tokens con el nuevo tenant.
- Un usuario sin membresía activa no recibe token tenant y ve el estado “Solicitar unirme a una organización”.
- `MS.Cobroup.Backend` valida issuer/audience y confía en la autoridad central; no almacena contraseñas ni emite tokens.
- Ambos frontends deberán conservar el access JWT solo en memoria y recibir el refresh JWT únicamente en cookie HttpOnly/Secure/SameSite; Web Storage queda prohibido.
- Las entradas HTTP propias se separaron por responsabilidad en controladores; `Program.cs` quedó para composición y middleware.
- La Fase 2 cuenta con commits de cierre autorizados.

### 4.2 Evidencia de cierre

- Migraciones de identidad aplicadas y segunda ejecución sin pendientes.
- La evidencia de la autoridad anterior queda como antecedente histórico y no sirve para aceptar JWT.
- Flujo JWT pendiente de validar para los tres roles: login, `me`, refresh rotatorio, cambio de organización, logout, replay y revocación familiar.
- Integración de ambos frontend con la autoridad real.
- Backend operativo protegido con política de usuario tenant.

### 4.3 Qué falta relacionado con Fase 2

La autenticación y el cambio de organización comprometidos para la fase están completos. Quedan decisiones o capacidades posteriores:

1. Diseñar e implementar el flujo real de “Solicitar unirme a una organización”, incluyendo solicitud, aprobación/rechazo, notificación y auditoría. Actualmente existe correctamente el estado sin inventar el proceso.
2. Completar la normalización y política del número telefónico de usuario, parte aún abierta de `ADR-011`.
3. Definir el aprovisionamiento productivo de organizaciones y del primer administrador. El bootstrap actual es únicamente el mecanismo local controlado.
4. Proveer clave privada asimétrica externa exclusiva de BackendAdmin, publicación JWKS y rotación por `kid` para cada ambiente.
5. Completar pruebas negativas, replay/concurrencia, penetración, carga, recuperación y aislamiento tenant.

Estos puntos no invalidan el cierre funcional de Fase 2; se encuentran diferidos explícitamente o pertenecen al endurecimiento productivo.

## 5. Fase 3 — Configuración y rutas

**Estado funcional:** COMPLETA  
**Estado Git:** CERRADA EN CUATRO COMMITS LOCALES  
**Objetivo:** permitir al administrador configurar su organización y gestionar rutas con asignaciones, cupos y auditoría, exponiendo al sistema operativo solo las rutas autorizadas.

### 5.1 Completado en BackendAdmin

- Configuración por organización de zona horaria IANA, moneda ISO y cupos generales opcionales.
- Semántica explícita de `null`: no existe límite configurado; no se inventan valores de negocio.
- Rutas con nombre único dentro del tenant, descripción, estado activo/inactivo, supervisor responsable y uno o más cobradores.
- Alta, consulta, edición, activación/desactivación y reasignación.
- Cupos de clientes y préstamos configurables por ruta como `Inherited` u `Overridden`.
- Resolución dinámica del valor efectivo heredado desde la organización.
- Catálogo de asignación limitado a membresías activas `Supervisor` y `Collector` del tenant del token.
- Escrituras restringidas a `Admin` tanto en API como en Core.
- Auditoría atómica con actor, fecha y JSON antes/después para configuración, estado, asignaciones y cupos.
- Lectura operativa filtrada: todas las rutas activas para `Admin`, rutas supervisadas para `Supervisor` y rutas asignadas para `Collector`.
- Controladores dedicados para rutas, usuarios asignables, configuración y acceso operativo.
- Migraciones `CompleteRouteConfiguration` y `AddAdministrativeAuditDetails` aplicadas.

### 5.2 Completado en Backend operativo

- `GET /api/operations/v1/routes` como contrato operativo propio.
- Consumo del directorio de BackendAdmin reenviando el bearer token.
- Sin base compartida ni duplicación de la autoridad de escritura de rutas.
- Conservación de respuestas 401/403.
- Conversión de timeout, red o fallo inesperado del servicio propietario a 503, evitando presentar una lista vacía engañosa.

### 5.3 Completado en Frontend administrativo

- Puerto de desarrollo `0.0.0.0:5300`.
- Pantalla `/configuracion` para zona horaria, moneda y cupos generales anulables.
- Pantalla `/rutas` para alta, listado, edición, estado, supervisor, cobradores y cupos heredados/sobrescritos.
- Visualización de valores efectivos y estados de carga, vacío, éxito y error.
- Bearer token en memoria y un reintento después de refresh ante 401.

### 5.4 Completado en Frontend operativo

- Puerto de desarrollo `0.0.0.0:5200`.
- Pantalla `/rutas` con rutas activas autorizadas, rol de acceso y cupos efectivos.
- Consumo exclusivo del Backend operativo; no consulta BackendAdmin directamente.
- Estados de carga, vacío y error sin datos simulados ni caché de negocio.

### 5.5 Evidencia de cierre

| Validación | Resultado |
|---|---|
| BackendAdmin Release | 0 warnings, 0 errores |
| BackendAdmin Core | 22/22 pruebas |
| BackendAdmin Infrastructure | 3/3 pruebas |
| Backend operativo Release | 0 warnings, 0 errores |
| Backend operativo Core | 6/6 pruebas |
| Backend operativo Infrastructure | 5/5 pruebas |
| Frontend administrativo | Build productivo y 8/8 pruebas |
| Frontend operativo | Build productivo y 5/5 pruebas |
| Integración de identidad histórica | Supersedida; repetir bajo contrato JWT |
| Lecturas administrativas/operativas | HTTP 200 |
| Migraciones EF | 5 migraciones listadas, sin pendientes |
| Script idempotente EF | Generado correctamente |
| Escaneo de secretos del diff | 0 coincidencias de alta confianza |
| Servicios temporales | 0 listeners en 5198, 5199 y 7120 |

### 5.6 Qué falta relacionado con Fase 3

No falta funcionalidad dentro del alcance aprobado. Quedan estas acciones de cierre:

1. Decidir y autorizar por separado push, pull request y merge; ninguno se ha ejecutado.
2. Ejecutar una auditoría independiente si se requiere una segunda revisión antes de integrar.
3. Agregar datos funcionales de rutas únicamente cuando el usuario lo solicite. La validación actual evitó crear rutas ficticias.

### 5.7 Elementos diferidos que no son faltantes de Fase 3

- La selección, traslado y orden de clientes requiere implementar `Customer`, teléfonos y ubicaciones en Fase 4.
- La asistencia por georreferenciación requiere clientes con coordenadas y cerrar la decisión de proveedor de mapas en Fase 8.
- Plantillas, préstamos, pagos, mora y reportes pertenecen a las fases 5–8.

## 6. Fases 1 a 5 completadas y fases 6 a 9 pendientes

### 6.1 Fase 4 — Clientes y alcance operativo

**Estado:** COMPLETA  
**Dependencias satisfechas:** fases 2 y 3 completas. El tratamiento futuro de préstamos activos al trasladar permanece en ADR-005 y no aplica todavía porque no existe originación.  
**Repositorios principales:** Backend operativo y Frontend operativo; BackendAdmin/Frontend Admin para consultas o administración autorizada.

Completado:

- Clientes por tenant y ruta, nombres, comercio/trabajo y observaciones.
- Entre 1 y 6 teléfonos, sin duplicados y con principal.
- Entre 1 y 4 ubicaciones, principal, origen manual/GPS y consentimiento.
- Cupo transaccional de clientes activos por ruta.
- Alta, consulta, búsqueda, edición y traslado dentro del alcance del actor.
- Aislamiento por tenant, ruta, ID directo y búsqueda; exportación no fue expuesta en esta fase.
- Base para etiquetas y selección/orden de clientes dentro de la ruta.
- Auditoría atómica de creación, edición, traslado y cambio de estado con actor, fecha y resumen antes/después.
- Fachada administrativa sin duplicar tablas ni autoridad de escritura.
- Pantallas operativa y administrativa con estados de carga, vacío, éxito y error.

Evidencia de aceptación funcional: AC-01, AC-02, AC-04, AC-05 y AC-20 cubiertos por pruebas de dominio/servicio, aislamiento cross-tenant, roles, PostgreSQL real y dos altas concurrentes con cupo 1. La integración de identidad debe repetirse bajo el contrato JWT antes de considerar vigente la aceptación end-to-end.

### 6.2 Fase 5 — Plantillas y excepciones

**Estado:** COMPLETA FUNCIONALMENTE  
**Dependencias funcionales:** fases 1 a 4 versionadas. La identidad multi-organización debe revalidarse con JWT conforme; el directorio de clientes activos permanece vinculado a ruta. ADR-003 permanece abierta deliberadamente para Fase 6.  
**Repositorios involucrados:** los cuatro repositorios (`MS.Cobroup.BackendAdmin`, `MS.Cobroup.Backend`, `MF.cobroup-frontend-admin`, `MF.cobroup-frontend`).

Completado:

- **Catálogo versionado e inmutabilidad:**
  - Plantillas de crédito predefinidas `LOW`, `HIGH` y personalizadas con control de versiones estrictamente inmutables.
  - Reglas de negocio con límites exactos en `decimal(18,2)` para montos y `decimal(5,2)` para tasas, duración en días/meses, frecuencia y estados borrador/publicada/inactiva con nombre único por tenant.
  - Publicación protegida y auditable; las plantillas publicadas son de solo lectura y nunca se modifican destructivamente.
- **Solicitudes de excepción vinculadas a cliente:**
  - Entidad de dominio `LoanExceptionRequest` en `MS.Cobroup.Backend` con estados `Pending`, `Approved`, `Rejected` y `Cancelled`.
  - Persistencia tenant-safe en esquema `operations.loan_exception_requests` con foreign key referencial a `customers(id)`, índices compuestos y migración EF Core `20261021200640_AddLoanExceptionRequests` aplicada en PostgreSQL.
  - Solicitud parametrizada con cliente visible en ruta, plantilla de referencia opcional, montos propuestos, tasa, plazo y justificación obligatoria.
  - Cancelación idempotente disponible para el solicitante, supervisor de la ruta o administrador mientras permanezca en estado pendiente.
- **Aprobación/rechazo exclusiva de Admin (AC-08):**
  - Endpoint de decisión protegido por política `tenant-admin` en API y verificado estrictamente en Core por rol (`actor.Role is UserRole.Admin`).
  - Prohibición verificada por pruebas: cobradores o supervisores no pueden aprobar ni rechazar solicitudes de excepción bajo ninguna circunstancia.
  - Trazabilidad y notas de auditoría obligatorias ante rechazo.
  - Registro de auditoría administrativa transaccional (`AdministrativeAuditRecord.LoanExceptionDecided`) en `MS.Cobroup.BackendAdmin`.
  - Integración HTTP desacoplada mediante `IOperationalLoanExceptionDirectory` y `OperationalLoanExceptionDirectoryClient` reenviando el Bearer token con manejo de `401/403/503`.
- **Experiencia de usuario en frontend:**
  - PWA Administrativa (`MF.cobroup-frontend-admin`): módulo `/excepciones` con filtros por estado, modal interactivo de decisión con notas, badges temáticos, toasts reactivos y presupuesto de estilos optimizado (< 4 kB).
  - PWA Operativa (`MF.cobroup-frontend`): módulo `/plantillas` para consulta de condiciones vigentes y módulo `/excepciones` para visualización de solicitudes, formulario de solicitud asistida seleccionando cliente visible, y cancelación de solicitudes pendientes.

Evidencia de aceptación: AC-07, AC-08 y AC-09 cubiertos:
- **Builds:** 0 warnings y 0 errores en los cuatro repositorios (Release en .NET, production bundle en Angular).
- **Pruebas unitarias/integración:** 102 pruebas backend (42 Core + 16 Infra en Backend; 36 Core + 8 Infra en BackendAdmin) y 22 pruebas frontend (12 en Admin; 10 en Operativo) pasando al 100%.
- **Base de datos:** Migración EF Core aplicada exitosamente sobre PostgreSQL.

### 6.3 Fase 6 — Préstamos y cronograma

**Estado:** COMPLETA FUNCIONALMENTE  
**Dependencias satisfechas:** Fases 1 a 5 completas. Catálogo inmutable de plantillas publicado, circuito de excepciones aprobado y clientes activos por ruta.  
**Repositorios involucrados:** Los cuatro repositorios (`MS.Cobroup.Backend`, `MS.Cobroup.BackendAdmin`, `MF.cobroup-frontend`, `MF.cobroup-frontend-admin`).

Completado:

- **Dominio y motor financiero de cronograma:**
  - Entidad de dominio `Loan` con términos inmutables (`LoanTermsSnapshot`), estados `Active`, `PaidOff`, `Cancelled` y colección de `Installment`.
  - Calculador financiero `LoanScheduleCalculator` garantizando la regla matemática estricta: `sum(installments) == totalAmount` mediante ajuste por residuo de centavos en la última cuota (AC-10).
  - Soporte de frecuencias de pago `Daily`, `Weekly` y `EveryNDays`, proyectando fechas con exclusión opcional de domingos si se parametriza.
  - Validación de cupo de préstamos activos por ruta: se comprueba contra el cupo efectivo de la ruta antes de originar.
  - Validación de coincidencia con plantilla de crédito publicada o excepción de crédito en estado `Approved` (AC-06, AC-07, AC-08, AC-09).
- **Persistencia y migración en PostgreSQL:**
  - Mapeo EF Core en esquema `operations` para `loans` (con `ComplexProperty` de `LoanTermsSnapshot`) e `installments` (con clave foránea e índice `loan_id, number`).
  - Migración EF Core `20261021210019_AddLoansAndInstallments` generada y aplicada exitosamente en PostgreSQL.
  - Repositorio `EfLoanRepository` implementado con estrategia desacoplada en dos fases para compatibilidad total con PostgreSQL e InMemory.
- **Microservicio Backend Admin:**
  - Fachada de auditoría y supervisión `/api/v1/loans` con búsqueda paginada, filtros por ruta/cliente/estado y detalle con cuotas.
  - Integración desacoplada vía `IOperationalLoanDirectory` y `OperationalLoanDirectoryClient`.
  - Protección estricta exigiendo rol `UserRole.Admin`.
- **Experiencia de usuario en PWA Operativa (`MF.cobroup-frontend`):**
  - Módulo `/prestamos` con listado de cartera, filtros por ruta y estado.
  - Modal de originación asistida con simulación en tiempo real (`previewSchedule`), selector de cliente activo, modo plantilla vs modo excepción.
  - Modal de inspección de cronograma de amortización con tabla de cuotas, montos esperados y pagados.
- **Experiencia de usuario en PWA Administrativa (`MF.cobroup-frontend-admin`):**
  - Módulo `/prestamos` con auditoría centralizada multi-ruta, filtros y modal de cronograma.
  - Estilos optimizados cumpliendo el budget de Angular (< 4.00 kB).

Evidencia de aceptación: AC-06, AC-09 y AC-10 cubiertos:
- **Builds:** 0 warnings y 0 errores en los cuatro repositorios (Release en .NET, production bundle en Angular).
- **Pruebas automatizadas:** 162 pruebas pasando en verde (Backend: 81; BackendAdmin: 46; Frontend: 17; FrontendAdmin: 18).
- **Base de datos:** Migración aplicada y tablas operativas indexadas.

### 6.4 Fase 7 — Pagos, evidencias y conciliación

**Estado:** COMPLETA FUNCIONALMENTE  
**Rama activa:** `feature/fase-7-pagos-conciliacion`  
**Dependencias:** préstamo y cronograma estables (Fase 6); integración desacoplada de medios de pago.

#### Entregables completados por repositorio:

- **Backend Administrativo (`MS.Cobroup.BackendAdmin`):**
  - Dominio: Entidad `PaymentMethod` (`OrganizationId`, `Name`, `NormalizedName`, `DisplayOrder`, `IsActive`) y auditoría en `AdministrativeAuditRecord`.
  - Core: `IPaymentMethodRepository`, `PaymentMethodContracts`, `PaymentMethodService` gestionando alta, actualización, ordenamiento y activación/desactivación reactiva con validación estricta de `UserRole.Admin`.
  - Persistencia: Mapeo en `identity.payment_methods` con restricción de unicidad `(OrganizationId, NormalizedName)` y clave foránea a `organizations`. Implementación `EfPaymentMethodRepository`.
  - Base de datos: Migración EF Core `20261021215708_AddPaymentMethods` generada y aplicada exitosamente en PostgreSQL.
  - API HTTP: `PaymentMethodsController` exponiendo endpoints administrativos (`/api/v1/payment-methods`) y directorio disponible (`/api/v1/payment-methods/available`).
  - Pruebas: 56 pruebas automatizadas pasando al 100% (47 en `Core.Tests`, 9 en `Infrastructure.Tests`).
- **Backend Operativo (`MS.Cobroup.Backend`):**
  - Dominio: Métodos de negocio `ApplyPayment(decimal amount)` en `Loan` e `Installment` con validaciones de importe positivo, reducción de saldo y actualización de estado. Entidades `Payment`, `PaymentAllocation`, `PaymentEvidence` en `Domain/Payments`.
  - Core: `PaymentService` implementando:
    - Imputación en cascada sobre cuotas pendientes/parciales en orden cronológico exacto (AC-11).
    - Prevención estricta de pagos dobles mediante clave de idempotencia única por tenant (AC-12).
    - Validación de medio de pago activo y autorizado para el tenant mediante directorio desacoplado (AC-13).
    - Soporte de fotografía auditable con retiro lógico restringido exclusivamente a rol `Admin` (`IsDeleted`, motivo de borrado, usuario auditor) (AC-14).
    - Conciliación diaria consolidada por ruta y medio de pago (AC-17).
  - Persistencia: `EfPaymentRepository`, mapeo en `operations.payments`, `operations.payment_allocations`, `operations.payment_evidences` con índice único `(TenantId, IdempotencyKey)`.
  - Base de datos: Migración EF Core `20261021220458_AddPaymentsAndEvidences` generada y aplicada en PostgreSQL.
  - API HTTP: `PaymentsController` (`POST /api/operations/v1/payments`, `GET by-loan/{loanId}`, `GET daily-reconciliation`, `POST {id}/evidence`, `DELETE {id}/evidence/{evidenceId}`, `GET payment-methods`).
  - Pruebas: 94 pruebas automatizadas pasando al 100% (71 en `Core.Tests`, 23 en `Infrastructure.Tests`).
- **Frontend Operativo (`MF.cobroup-frontend`):**
  - Módulo de pagos y recaudos: `payments.service.ts` y contratos tipados `payment.models.ts`.
  - Modal de cobro en `/prestamos`: simulación de imputación en cascada en vivo sobre cuotas (`previewCascade`), selector de medios de pago autorizados, captura/adjunto de evidencia fotográfica, clave UUID de idempotencia generada en cliente, recibo digital de confirmación e historial de pagos por préstamo con visualización de fotos.
  - Conciliación diaria `/conciliacion`: selector de fecha y ruta, tarjetas de métricas (total cobrado, cuotas cubiertas, ticket promedio), tabla por medio de pago con barras de distribución porcentual y fila de totales.
  - Estilos y presupuestos: SCSS refactorizado estrictamente con tokens de variables semánticas (`var(--color-...)`); presupuesto de estilos < 4.00 kB cumplido con 0 advertencias.
  - Pruebas: 25 pruebas Vitest pasando al 100%. Build productivo limpio (0w/0e).
- **Frontend Administrativo (`MF.cobroup-frontend-admin`):**
  - Módulo de medios de pago: `payment-methods.service.ts` y modelos tipados.
  - Página `/medios-pago`: catálogo completo de medios de pago por organización, modal reactivo de creación y edición, orden de visualización, conmutador (toggle) de activación/desactivación reactivo con confirmación visual.
  - Estilos y presupuestos: SCSS adaptado con tokens semánticos globales; presupuesto de componentes < 4.00 kB cumplido con 0 advertencias.
  - Pruebas: 24 pruebas Vitest pasando al 100%. Build productivo limpio (0w/0e).

Evidencia de aceptación: AC-11, AC-12, AC-13, AC-14 y AC-17 cubiertos al 100%:
- **Builds:** 0 warnings y 0 errores en los cuatro repositorios (Release en .NET 10, production bundle en Angular 21).
- **Pruebas automatizadas:** 199 pruebas pasando en verde (Backend: 94; BackendAdmin: 56; Frontend: 25; FrontendAdmin: 24).
- **Base de datos:** Migraciones `AddPaymentMethods` y `AddPaymentsAndEvidences` aplicadas en PostgreSQL.

### 6.5 Fase 8 — Mapas, georreferenciación y analítica

**Estado:** COMPLETA FUNCIONALMENTE  
**Rama activa:** `feature/fase-8-mapas-analitica`  
**Dependencias:** Préstamos y pagos estabilizados (Fases 6 y 7); integración de mapas interactivos sin API keys privadas (`ADR-006` resuelto con OpenStreetMap raster tiles).

#### Entregables completados por repositorio:

- **Backend Operativo (`MS.Cobroup.Backend`):**
  - Dominio:
    - `AgendaStatus.cs`: enumeración tipada de estados de cobranza para agenda y mapas (`Overdue`, `DueToday`, `UpToDate`, `CollectedToday`).
    - `HaversineGeoCalculator.cs`: algoritmo geodésico puro en dominio (fórmula del semiverseno con radio terrestre de 6.371 km) para cálculo exacto de distancias entre coordenadas lat/lon, ordenamiento voraz de paradas de cobro (Nearest Neighbor / traveling salesman heurístico) y cálculo de distancias acumuladas.
    - `IAgendaRepository.cs`: puerto de persistencia para consultas de agenda, ubicaciones activas de clientes y métricas de cartera.
  - Core:
    - `AgendaContracts.cs`: DTOs inmutables para agenda diaria, paradas georreferenciadas, optimización de secuencia y analítica de cartera.
    - `AgendaService.cs`:
      - Cálculo exacto de vencimientos, cuotas del día y mora según la fecha local (AC-16).
      - Optimización de secuencia de visitas por cercanía geográfica reduciendo tiempos de desplazamiento del cobrador.
      - Extracción de ubicaciones georreferenciadas de clientes de la ruta con semáforo de colores (`#ef4444` mora, `#eab308` vence hoy, `#22c55e` al día/cobrado, `#3b82f6` sin cuota hoy).
      - Motor analítico de cartera en tiempo real: cálculo de colocación total, recaudado histórico, saldo activo, mora vencida, índice PAR% (Portfolio at Risk: `overdue / balance * 100`), tasa de efectividad diaria y desglose de envejecimiento de deuda (Aging Report) en tramos canónicos (0 días al día, 1-7 días, 8-15 días, 16-30 días y >30 días).
  - Persistencia:
    - `EfAgendaRepository.cs`: implementación sobre `OperationsDbContext` y PostgreSQL consultando `loans`, `installments`, `customers` y `customer_addresses`.
  - API HTTP:
    - `AgendaController.cs`: endpoints operativos protegidos por rol: `GET /api/operations/v1/agenda/daily`, `POST /api/operations/v1/agenda/optimize-route`, `GET /api/operations/v1/agenda/map-locations`, `GET /api/operations/v1/agenda/portfolio-analytics`.
  - Pruebas automatizadas:
    - 106 pruebas pasando al 100% (81 en `Core.Tests`, 25 en `Infrastructure.Tests`), incluyendo pruebas unitarias del algoritmo de Haversine, optimización secuencial, tramos de aging report y persistencia PostgreSQL.

- **Backend Administrativo (`MS.Cobroup.BackendAdmin`):**
  - Core:
    - `IOperationalAgendaDirectory.cs`: interfaz desacoplada para consumo de ubicaciones y analítica operativa.
    - `MapSupervisionService.cs`: orquestación de mapa de supervisión multi-ruta con agregación de saldos y morosidad.
    - `AnalyticsAdministrationService.cs`: consolidación de analítica ejecutiva, rendimiento comparativo entre rutas del tenant y reporte global de envejecimiento de cartera.
  - Infraestructura:
    - `OperationalAgendaDirectoryClient.cs`: cliente HTTP proxy autenticado reenviando Bearer token tenant-safe hacia el backend operativo.
  - API HTTP:
    - `MapSupervisionController.cs`: endpoint administrativo `/api/v1/map/route-locations`.
    - `AnalyticsController.cs`: endpoints `/api/v1/analytics/overview`, `/api/v1/analytics/routes-performance`, `/api/v1/analytics/aging`.
  - Pruebas automatizadas:
    - 60 pruebas pasando al 100% (51 en `Core.Tests`, 9 en `Infrastructure.Tests`).

- **Frontend Operativo (`MF.cobroup-frontend`):**
  - Componente UI reutilizable: `map-view.component.ts`: visualización interactiva de mapas con renderizado de cuadrícula/raster OpenStreetMap, marcadores interactivos codificados por color con popups informativos de saldo, cuota y mora.
  - Página `/agenda` (`agenda.page.ts`):
    - Tarjetas de resumen diario (total paradas, cuota esperada, clientes en mora, cobros completados).
    - Lista secuenciada de paradas con indicación de dirección, teléfono y saldo.
    - Acción reactiva "Reordenar por GPS": geolocalización del navegador y reordenamiento voraz por distancia mínima.
    - Acción directa "Cobrar": acceso inmediato al modal de pago con datos pre-cargados.
  - Página `/mapa` (`mapa.page.ts`): mapa de ruta con visualización geográfica de todos los clientes asignados, selección de parada y tarjeta flotante de contacto/cobro.
  - Rutas y layout: integrados en `app.routes.ts` y barra de navegación de `main-layout`.
  - Estilos y presupuestos: SCSS con variables semánticas (`var(--color-...)`), < 4.00 kB por componente.
  - Pruebas automatizadas: 28 pruebas Vitest pasando al 100%. Build de producción limpio (0w/0e).

- **Frontend Administrativo (`MF.cobroup-frontend-admin`):**
  - Componente UI reutilizable: `map-view.component.ts`.
  - Página `/mapa` (`map-supervision.page.ts`):
    - Supervisión geográfica multi-ruta con selector de ruta y selector de fecha.
    - Filtros dinámicos por estado: "Todos", "En mora", "Vencen hoy", "Cobrados".
    - Panel flotante con detalle de cliente, saldo activo, mora y datos comerciales.
  - Página `/analitica` (`analytics.page.ts`):
    - 6 tarjetas de métricas clave (Cartera colocada, Recaudado histórico, Saldo por cobrar, Cartera en mora, PAR% y Efectividad diaria).
    - Tabla visual de envejecimiento de cartera (Aging Report) con montos por tramo, porcentaje de cartera y barras de progreso proporcionales.
    - Tabla comparativa de rendimiento y ranking de rutas con indicador visual de riesgo (Bajo, Moderado, Alto).
  - Rutas y layout: registradas en `app.routes.ts` y enlaces en `main-layout`.
  - Estilos y presupuestos: SCSS desacoplado respetando budget de 4.00 kB con 0 advertencias.
  - Pruebas automatizadas: 31 pruebas Vitest pasando al 100%. Build de producción limpio (0w/0e).

Evidencia de aceptación: AC-15, AC-16, AC-18 y AC-19 cubiertos al 100%:
- **Builds:** 0 warnings y 0 errores en los cuatro repositorios (.NET Release y Angular production).
- **Pruebas automatizadas:** 225 pruebas pasando en verde (Backend: 106; BackendAdmin: 60; Frontend: 28; FrontendAdmin: 31).
- **Commits locales de Fase 8:** `MS.Cobroup.Backend` (`cf65b00`), `MS.Cobroup.BackendAdmin` (`ed609a5`), `MF.cobroup-frontend` (`ddfe552`), `MF.cobroup-frontend-admin` (`db29c9a`).

---

## 7. Fase 9 — Endurecimiento y despliegue K3s/Kubernetes

**Estado:** COMPLETA  
**Rama activa:** `feature/fase-9-endurecimiento-despliegue`  
**Objetivo:** dotar a la plataforma de observabilidad técnica (health checks liveness/readiness), seguridad HTTP perimetral, empaquetado OCI inmutable con Docker multi-stage, orquestación declarativa en K3s / Kubernetes con separación estricta de ambientes (`cobroup-dev`, `cobroup-qa`, `cobroup-prod`), canal de CI/CD automatizado vía GitHub Actions con publicación en GHCR y despliegue por SSH, y servicios PWA de ciclo de vida.

### 7.1 Entregables y arquitectura técnica implementada

1. **Observabilidad y Health Checks en Backend (.NET 10):**
   - Implementación de `OperationsDatabaseHealthCheck` en `MS.Cobroup.Backend` e `IdentityDatabaseHealthCheck` en `MS.Cobroup.BackendAdmin` verificando conectividad activa contra PostgreSQL.
   - Configuración en `Program.cs` de probes nativas:
     - `/health/live`: comprobación básica de liveness para reinicio de contenedor si el proceso no responde.
     - `/health/ready`: comprobación de readiness que valida la base de datos antes de enrutar tráfico de clientes.
   - Configuración de `UseForwardedHeaders` para soportar reverse proxy e Ingress Traefik sin pérdida de IP cliente ni protocolo HTTPS.
   - Middleware de seguridad HTTP con inyección de cabeceras de protección: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, `X-XSS-Protection: 0`.
   - Trazabilidad distribuida con middleware de Correlation ID (`X-Correlation-ID`).

2. **Contenedorización multi-stage y seguridad de imágenes:**
   - Creación de `.dockerignore` y `Dockerfile` optimizados para los cuatro repositorios:
     - `MS.Cobroup.Backend`: .NET SDK 10 multi-etapa -> ASP.NET Core 10 runtime, ejecución con usuario no root (`1000:1000`), puerto unificado 8080.
     - `MS.Cobroup.BackendAdmin`: .NET SDK 10 multi-etapa -> ASP.NET Core 10 runtime, usuario no root, puerto 8080.
     - `MF.cobroup-frontend`: Node 22-alpine para compilación productiva -> Nginx 1.27-alpine como servidor web unprivileged en puerto 8080 con compresión gzip, fallback SPA `try_files` y endpoint `/health`.
     - `MF.cobroup-frontend-admin`: Node 22-alpine -> Nginx 1.27-alpine unprivileged en puerto 8080.

3. **Manifiestos declarativos para K3s / Kubernetes (`k8s/`):**
   - `k8s/base/namespace.yaml`: creación dinámica del namespace según el ambiente activo (`${ENVIRONMENT}`).
   - `k8s/base/backend-admin.yaml`: Deployment y Service ClusterIP con límites de recursos (CPU 200m–1000m, Memoria 256Mi–512Mi) y health probes (`/health/live`, `/health/ready`).
   - `k8s/base/backend-operations.yaml`: Deployment y Service ClusterIP con límites de recursos y health probes.
   - `k8s/base/frontend-collector.yaml`: Deployment y Service ClusterIP para la PWA de cobradores con health probe (`/health`).
   - `k8s/base/frontend-admin.yaml`: Deployment y Service ClusterIP para la PWA de administradores con health probe (`/health`).
   - `k8s/base/ingress.yaml`: Ingress Traefik con soporte de TLS cert-manager y enrutamiento por hostname para aplicaciones y APIs.
   - `k8s/base/configmap.yaml` y `k8s/base/secrets-template.yaml`: plantillas de configuración y credenciales.
   - `k8s/deploy-k3s.sh`: script bash automatizado para renderizado con `envsubst`, aplicación atómica y verificación progresiva (`kubectl rollout status`).

4. **Automatización CI/CD con GitHub Actions (`.github/workflows/ci-cd.yml`):**
   - Configurado en los cuatro repositorios con disparadores automáticos en push a ramas `dev`, `qa`, `prod`, `main` y ejecución manual `workflow_dispatch`.
   - **Mapeo automático de ambientes:**
     - Push a `dev` -> despliegue en namespace `cobroup-dev`.
     - Push a `qa` -> despliegue en namespace `cobroup-qa`.
     - Push a `prod` o `main` -> despliegue en namespace `cobroup-prod`.
   - **Flujo de pipeline:**
     - Etapa 1: ejecución obligatoria de suite de pruebas automatizadas (cero tolerancia a fallos).
     - Etapa 2: compilación de imagen Docker y publicación en GitHub Container Registry (`ghcr.io`) con etiquetado por SHA, rama y latest.
     - Etapa 3: conexión SSH al servidor remoto donde corre K3s, inyección de variables (`vars.*`) y secretos (`secrets.*`) de GitHub según el ambiente, actualización de imagen en el cluster y verificación de `kubectl rollout status`.

5. **Servicios PWA en Frontend:**
   - `PwaUpdateService` en ambas aplicaciones para detección automática de nuevas versiones mediante Angular Service Worker (`SwUpdate`) y recarga controlada.
   - `PwaInstallService` para captura reactiva del evento `beforeinstallprompt` y habilitación de instalación nativa en dispositivos móviles y de escritorio.

---

## 8. Validación del framework de desarrollo

Se realizó una auditoría y verificación exhaustiva de los cambios de Fase 9 contra los estándares canónicos del proyecto:

### 8.1 Framework de Backend (`FRAMEWORK_DESARROLLO_CSHARP.md` v2.0)
| Criterio | Requisito | Estado | Evidencia |
|---|---|---|---|
| **Rama Git** | Nombre bajo patrón `feature/*` | **CUMPLE** | Rama activa `feature/fase-9-endurecimiento-despliegue` en ambos backend |
| **Arquitectura hexagonal** | Proyectos independientes `Domain`, `Core`, `Shared`, `Infrastructure`, `Api` | **CUMPLE** | Estructura hexagonal pura sin carpeta `src/`; health checks en capa `Api/Health` |
| **Separación de responsabilidades** | Observabilidad desacoplada de reglas de negocio | **CUMPLE** | `OperationsDatabaseHealthCheck` e `IdentityDatabaseHealthCheck` implementan `IHealthCheck` estándar de ASP.NET Core |
| **Seguridad de cabeceras** | Protección contra XSS, clickjacking y MIME sniffing | **CUMPLE** | Middleware de seguridad configurado en pipeline HTTP de ambos microservicios |
| **Ejecución en contenedor** | Contenedores seguros sin permisos de superusuario | **CUMPLE** | Dockerfile ejecuta bajo usuario unprivileged `1000:1000` en puerto 8080 |
| **Pruebas automatizadas** | 100% pruebas en verde sin advertencias | **NO CUMPLE EN EL CORTE JWT** | Backends: 167/167; frontends: 57/59, con 2 pruebas fallidas |

### 8.2 Framework de Frontend (`FRAMEWORK_DESARROLLO_FRONTEND.md` v1.0)
| Criterio | Requisito | Estado | Evidencia |
|---|---|---|---|
| **Identidad de repositorios** | Patrón `MF.<kebab-case>` | **CUMPLE** | `MF.cobroup-frontend` y `MF.cobroup-frontend-admin` |
| **Servicios de infraestructura PWA** | Ubicación centralizada en `core/pwa` | **CUMPLE** | `PwaUpdateService` y `PwaInstallService` en `src/app/core/pwa/` |
| **Ejecución en contenedor Nginx** | Servidor web liviano sin privilegios | **CUMPLE** | Nginx alpine configurado en puerto 8080 con health endpoint `/health` y compresión gzip |
| **Presupuestos de estilos (Budget)** | Component SCSS < 4.00 kB en bundle de producción | **CUMPLE** | Ambas aplicaciones compilan con 0 advertencias de budget en build de producción |
| **Pruebas automatizadas** | Pruebas unitarias de servicios y componentes en Vitest | **CUMPLE** | Frontend: 28 pruebas; FrontendAdmin: 31 pruebas (59 pruebas Vitest) pasando con `--watch=false` |

---

## 9. Decisiones de arquitectura cerradas en Fase 9

| Decisión | Estado/impacto |
|---|---|
| `ADR-001` Aprovisionamiento productivo | Resuelto mediante orquestación K3s y variables de entorno por namespace |
| `ADR-003` Amortización y ajuste legal | Resuelto y probado en Fase 6 con ajuste decimal exacto |
| `ADR-006` Mapas y georreferenciación | Resuelto en Fase 8 con OpenStreetMap / raster grid libre de costos y Haversine puro |
| `ADR-007` Pagos, cascada e idempotencia | Resuelto e implementado en Fase 7 con PostgreSQL |
| `ADR-008` Operación PWA | Resuelto con Service Worker, cache de assets estáticos y servicios PWA de actualización |
| `ADR-010` Retención de evidencias | Retiro administrativo auditable implementado en Fase 7 (AC-14) |
| `ADR-012` Infraestructura K3s / CI-CD | Resuelto en Fase 9 con GitHub Actions, GHCR, SSH y manifiestos K3s para `cobroup-dev`, `cobroup-qa`, `cobroup-prod` |
| Autenticación exclusiva JWT | Aprobada el 2026-10-22; implementación reabierta y sujeta a `DECISION_AUTENTICACION_JWT.md` |

---

## 10. Resumen de aseguramiento de calidad (QA)

1. **Estado Git:** funcionalidad histórica versionada; segunda ronda JWT sin commit y sin aceptación auditora.
2. **Ambientes soportados:** `cobroup-dev`, `cobroup-qa` y `cobroup-prod` totalmente modelados y desacoplados.
3. **Seguridad de identidad:** contrato asimétrico aprobado; la implementación actual aún contiene hallazgos que deben corregirse.
4. **Calidad de software:** builds pasan; backends 167/167 y frontends 57/59. El gate global no está verde.

---

## 11. Hito completado

Commits locales de cierre de Fase 9 realizados exitosamente bajo autorización explícita:
- `MS.Cobroup.Backend`: `e02bd65` (`feat(hardening): implement database health checks, security headers, dockerfile and k3s ci-cd`)
- `MS.Cobroup.BackendAdmin`: `91152b1` (`feat(hardening): implement identity health checks, security headers, dockerfile and k3s ci-cd`)
- `MF.cobroup-frontend`: `c3f007f` (`feat(pwa-deploy): implement pwa update services, nginx container and k3s ci-cd`)
- `MF.cobroup-frontend-admin`: `d218f03` (`feat(pwa-deploy): implement pwa update services, nginx container and k3s ci-cd`)

**Próximo paso operativo:**
Adecuar la implementación a `DECISION_AUTENTICACION_JWT.md`, dejar todas las pruebas verdes y solicitar una nueva auditoría sobre un commit inmutable. No desplegar antes de ese cierre.

---

## 12. Fuentes de evidencia

- `COBROUP.md`, `DECISION_AUTENTICACION_JWT.md`, `FRAMEWORK_DESARROLLO_CSHARP.md` y `FRAMEWORK_DESARROLLO_FRONTEND.md` en `cobroup.md`.
- Manifiestos K3s en `k8s/base/` y script `k8s/deploy-k3s.sh`.
- Workflows CI/CD `.github/workflows/ci-cd.yml` en los cuatro repositorios.
- Dockerfiles y `.dockerignore` en los cuatro repositorios.
- Resultados del corte auditor 2026-10-22: backends 167/167; frontends 57/59; migración Admin pendiente.

---

**Conclusión:** las nueve fases conservan funcionalidad versionada, pero identidad está reabierta. La decisión definitiva es JWT exclusivo y la implementación no se considera conforme hasta cerrar los criterios criptográficos, de sesión, tenant, migración, ingress y pruebas descritos en el contrato.

---

## 13. Adenda de redefinición: Deferir PEM RSA y Formalización de Backlog Criptográfico (2026-10-22)

Conforme a la decisión formal aprobada e incorporada en `DECISION_AUTENTICACION_JWT.md`:

> COBROUP continuará usando JWT firmados. En la fase actual de desarrollo y validación no se configurará una clave privada externa en formato PEM. BackendAdmin utilizará un proveedor RSA efímero generado en memoria, habilitado de forma explícita y restringido a entornos no productivos. La clave y el `kid` cambian al reiniciar la instancia; en consecuencia, los JWT emitidos antes del reinicio pueden dejar de ser válidos. Este modo no se considera apto para producción, alta disponibilidad ni múltiples réplicas emisoras. La custodia de una clave persistente, su rotación, solapamiento de claves y recuperación operativa mediante Kubernetes Secret, PFX, PEM, AWS KMS u otro proveedor aprobado quedan registrados como backlog obligatorio previo a producción.

**Implicaciones operativas y de despliegue:**
1. **Desarrollo y pruebas locales:** BackendAdmin usa `EphemeralRsaSigningKeyProvider` condicionado a `Jwt:SigningProvider=EphemeralRsa` y `Jwt:AllowEphemeralSigning=true`. Si no está autorizado explícitamente, el arranque falla de inmediato (*fail-fast*).
2. **Desacoplamiento arquitectónico:** Se introdujo la abstracción `IJwtSigningKeyProvider` para que los controladores y servicios de emisión no conozcan detalles de archivos, PEM ni Kubernetes.
3. **Ambientes QA y Producción:** El uso de claves efímeras está bloqueado por defecto. Los workflows y el script `render_manifests.py` detienen el despliegue hacia QA y Producción antes de la transferencia SCP (código de salida 2).
4. **Condición de producción:** El sistema se declara formalmente **NO APTO PARA PRODUCCIÓN** hasta que se ejecute y apruebe el backlog de seguridad documentado en `cobroup.md/BACKLOG_SEGURIDAD_PRODUCTIVA.md`.

