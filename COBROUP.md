# COBROUP — Documento maestro del proyecto

> **Identificador:** COBROUP  
> **Versión:** 1.10.0 — JWT `HS256` con secreto simétrico independiente por ambiente y custodia GitHub/Kubernetes  
> **Fecha:** 2026-10-22  
> **Estado:** fases funcionales versionadas; almacenamiento en PVC formalizado; migración a firma simétrica por ambiente aprobada y pendiente de implementación/reauditoría  
> **Naturaleza:** plataforma multi-tenant para gestión de clientes, préstamos, rutas y recaudos  
> **Documento principal:** `COBROUP.md` (nombre canónico; conservar exactamente `cobroup` en los cuatro identificadores de repositorio)  
> **Informe de avance:** `informe/avance.md` y `informe/avance.html`  
> **Rama de trabajo vigente:** cambios locales sin commit; verificar la rama real de cada repositorio antes de implementar  
> **Alcance vigente:** JWT firmado exclusivamente con `HS256`; secreto CSPRNG Base64 de 32 bytes o más, distinto para Dev/QA/Prod y compartido solo por BackendAdmin y Backend dentro del mismo ambiente. Se retiran RSA, PEM/PFX, JWKS, proveedores efímeros/persistentes y acknowledge.

## 1. Propósito y alcance

COBROUP permite que organizaciones administren operaciones de préstamos y sus cobros mediante rutas asignadas a cobradores, supervisión operativa y un panel administrativo. Cada organización es un **tenant aislado**: sus usuarios, clientes, préstamos, rutas, plantillas de cuantías, medios de pago, etiquetas, pagos, fotografías y estadísticas no deben ser visibles ni modificables por otra organización.

Se divide en **dos aplicaciones frontend** y **dos microservicios backend** independientes. La aplicación operativa sirve a cobradores y supervisores, y habilita al administrador las operaciones que le corresponden; la aplicación administrativa gestiona configuración, asignaciones y monitoreo de la organización.

### 1.1 Objetivos

1. Gestionar clientes, ubicaciones y datos de contacto desde una PWA con acceso a cámara, galería, mapas y geolocalización con consentimiento.
2. Crear préstamos vinculados a clientes conforme a plantillas de cuantía, períodos y porcentajes autorizados.
3. Generar calendarios de cuotas reproducibles, mostrar saldos, vencimientos y mora, registrar cobros y adjuntar evidencia.
4. Configurar rutas con cupos, asignación de cobradores y supervisor, de forma manual o asistida por georreferenciación.
5. Permitir al administrador configurar la organización y sus rutas, visualizar el mapa operativo y resumir recaudo diario por medio de pago.
6. Mantener trazabilidad, seguridad multi-tenant, autorización por rol y alcance, integridad de dinero y privacidad de datos.

### 1.2 Fuera de alcance inicial

- Desembolsos bancarios automáticos, pasarelas de pago, conciliación bancaria y comprobantes fiscales automáticos.
- Seguimiento permanente de la posición de los cobradores, recopilación de ubicación en segundo plano sin autorización y geocercas activas.
- Cobranza automática intimidatoria, acceso a contactos del teléfono de terceros o extracción invasiva de información.
- Aprobación automática de excepciones individuales; corresponde exclusivamente al administrador.
- Operación completamente offline, reestructuración, refinanciación, castigos, intereses moratorios o comisiones no definidos explícitamente.
- Una interfaz o rol global de plataforma no solicitado. El aprovisionamiento inicial de organizaciones queda pendiente de decisión.

Los puntos fuera de alcance podrán convertirse en requisitos mediante decisión registrada, sin asumirse silenciosamente.

## 2. Repositorios y responsabilidades

| Repositorio exacto | Tipo | Responsabilidad | Estándar obligatorio |
|---|---|---|---|
| `MF.cobroup-frontend` | Angular / PWA | Aplicación operativa: clientes, ubicaciones, préstamos, rutas disponibles, cobros, agenda diaria y mora | `FRAMEWORK_DESARROLLO_FRONTEND.md` |
| `MF.cobroup-frontend-admin` | Angular / PWA | Administración tenant: parámetros, rutas, asignaciones, mapa, aprobaciones, auditoría y reportes | `FRAMEWORK_DESARROLLO_FRONTEND.md` |
| `MS.Cobroup.Backend` | .NET / API | Casos de uso operativos: clientes, préstamos, cronograma, recaudos, evidencias y consulta de agenda | `FRAMEWORK_DESARROLLO_CSHARP.md` |
| `MS.Cobroup.BackendAdmin` | .NET / API | Casos de uso administrativos: configuración, rutas, asignaciones, etiquetas, medios, plantillas, aprobaciones, supervisión y resúmenes | `FRAMEWORK_DESARROLLO_CSHARP.md` |

En ambos backend, toda entrada HTTP propia debe residir en un controlador dentro del proyecto API y cada responsabilidad tendrá su controlador dedicado (`AuthController`, `UsersController`, `CollectionsController`, etc.). `Program.cs` se limita a composición, configuración, middleware y `MapControllers`; no contiene handlers de negocio. Login, refresh, logout y contexto se implementan en controladores/servicios explícitos bajo el contrato JWT vigente; no existe endpoint JWKS.

**Regla fundamental:** dos backend no significan dos versiones de las mismas reglas. Son fronteras de capacidades; contratos, propiedad de datos, eventos y responsabilidades deben documentarse para evitar inconsistencias y escritura simultánea de las mismas entidades. Los permisos se verifican **siempre en el servidor**, nunca solamente en la interfaz.

### 2.1 Organización de trabajo recomendada

```text
cobroup/                              # Carpeta de trabajo contenedora; no es un quinto repositorio
├── COBROUP.md                         # Este contrato maestro
├── informe/                           # Informe de avance Markdown y HTML mantenible
├── FRAMEWORK_DESARROLLO_CSHARP.md     # Estándar local canónico aplicable a MS
├── FRAMEWORK_DESARROLLO_FRONTEND.md   # Estándar local canónico aplicable a MF
├── MF.cobroup-frontend/              # Repositorio Git autónomo
├── MF.cobroup-frontend-admin/        # Repositorio Git autónomo
├── MS.Cobroup.Backend/               # Repositorio Git autónomo
└── MS.Cobroup.BackendAdmin/          # Repositorio Git autónomo
```

- Los nombres externos anteriores son **fijos**. Los identificadores npm, Angular CLI, ensamblados y soluciones internas deben respetar los estándares y la sintaxis de cada tecnología.
- Si los estándares se encuentran en otra ubicación, referenciarlos sin reemplazarlos por una variante inventada. Si existe una versión local más reciente que una copia archivada, inspeccionar y utilizar la canónica del espacio de trabajo.
- Los proyectos frontend son independientes; no crear una aplicación única que simule dos frontend.
- No añadir aplicaciones móviles nativas ni un quinto servicio por defecto.

## 3. Marcos de desarrollo y jerarquía normativa

### 3.1 Frontend

Leer el archivo real `FRAMEWORK_DESARROLLO_FRONTEND.md` antes de generar código. Se exige Angular + TypeScript + SCSS, arquitectura **feature-based**, componentes standalone, carpetas `src/app/features`, `core`, `shared`, `layouts`, `themes`, diseño responsive, tokens para Light/Dark y destino PWA instalable. Nombre externo `MF.<kebab-case>`. Elegir versiones LTS y fijarlas con arreglo al estándar. Para esta fase se recomienda habilitar el armazón PWA desde la creación y registrar el resultado; si alguna característica se pospone, documentar motivo y obligación anterior a producción.

### 3.2 Backend

Leer el archivo real `FRAMEWORK_DESARROLLO_CSHARP.md` antes de generar código. Los proyectos COBROUP son **exclusivamente microservicios `MS`** en arquitectura hexagonal: Domain, Core, Shared, Infrastructure y host Api en proyectos independientes, con organización por responsabilidades, interfaces/puertos, casos de uso y pruebas. No colocar las capas productivas bajo un `src/` si el estándar lo prohíbe. Usar el .NET LTS admitido por la versión canónica del marco.

**Compatibilidad de versiones:** si en el espacio de trabajo conviven una versión antigua del estándar backend que admite WPF/WinForms con una actualización destinada únicamente a `MS`/`WS`, se debe aplicar la versión vigente indicada por el propietario del proyecto para COBROUP, solo en su parte `MS`; nunca agregar WPF, WinForms, MAUI o un frontend a los microservicios. Registrar en `AGENTS.md` la versión efectiva leída y cualquier conflicto.

### 3.3 Reglas comunes de los cuatro repositorios

- `AGENTS.md` en raíz, creado y mantenido desde el primer cambio, con historial cronológico de prompts, decisiones, archivos, pruebas, bloqueos y próximos pasos.
- `README.md`, `.gitignore`, documentación de contexto/decisiones, pruebas mínimas y configuración de ejemplo sin secretos.
- Trabajar exclusivamente en ramas `feature/*`, sujetas a las reglas de autorización del marco. No hacer commits, pushes, merges, tags ni PR sin solicitud expresa.
- Toda API, DTO y autorización debe estar respaldada por contratos compartidos **como especificaciones/versiones**, sin acoplar directamente un repositorio a clases internas del otro.
- Las decisiones por confirmar se documentan, no se convierten arbitrariamente en reglas de negocio definitivas.

## 4. Modelo multi-tenant

### 4.1 Organización (tenant)

Cada tenant representa una organización que cuenta con configuración propia:

- Identidad, nombre, estado (activa/suspendida), zona horaria IANA, moneda ISO, preferencias y parámetros operativos.
- Plantillas de cuantía baja, alta y personalizadas preconfiguradas.
- Medios de pago y etiquetas, límites generales de rutas y reglas susceptibles de heredarse.
- Usuarios con roles dentro de esa organización, rutas, clientes, préstamos, cuotas, pagos, evidencias, auditoría y reportes.

**Aislamiento obligatorio:** toda entidad de negocio tiene pertenencia a tenant (directa o verificable mediante una relación segura). La identidad autenticada determina el `organization_id`; **no confiar en un `organization_id` que el cliente envíe para acceder a recursos**. Validar tenant y alcance de cada relación (`route_id`, `customer_id`, `loan_id`, `payment_id`, `attachment_id`) y en consultas, comandos, medios y exportaciones. Aplicar filtrado y restricciones en almacenamiento más validación en Core; probar accesos cruzados.

### 4.2 Configuraciones heredadas por ruta

Cada ruta obtiene sus valores base de la organización, pero puede sobreescribir los atributos permitidos:

- Por cada parámetro heredable, usar `INHERITED` o `OVERRIDDEN` y almacenar un valor local únicamente en el segundo caso.
- En `INHERITED`, el valor efectivo proviene **siempre** de la configuración vigente de la organización (los cambios futuros se reflejan).
- En `OVERRIDDEN`, el valor efectivo es el de la ruta y no cambia si varía el global.
- Ofrecer acción administrativa para volver a herencia sin perder auditoría.
- Límites de clientes/préstamos por ruta y otras reglas configurables deberán seguir este esquema donde sea pertinente.
- **No modificar condiciones financieras de préstamos vigentes por cambiar plantillas o herencias.** Cada préstamo conserva una instantánea de las reglas pactadas en su originación.

Ejemplo de resolución:

```text
MaxActiveCustomersRoute =
  route.maxCustomers.mode == INHERITED
  ? organization.maxCustomersPerRoute
  : route.maxCustomers.value
```

## 5. Usuarios, autenticación y autorización

### 5.1 Datos mínimos de usuario

- El usuario es una identidad global con username normalizado único en la plataforma, contraseña cifrada con algoritmo de hash resistente y sal, número de teléfono, estado de habilitación, fechas de creación/actualización y trazabilidad.
- La pertenencia a tenant y el rol no son atributos globales del usuario: viven en una membresía usuario–organización. Un mismo usuario puede pertenecer a varias organizaciones y tener un rol diferente en cada una.
- El login solicita únicamente usuario y contraseña. Selecciona la última membresía activa utilizada según `LastAccessedAtUtc`; si ninguna tiene acceso previo, usa como inicio la membresía incorporada más recientemente.
- Un usuario sin membresías activas no recibe token tenant. La interfaz muestra el estado **“Solicitar unirme a una organización”**; el contrato de solicitud/aprobación debe definirse antes de habilitarlo.
- El usuario puede cambiar a otra membresía activa. El servidor actualiza el último acceso y emite una sesión nueva; el cliente revoca el refresh token anterior y no reutiliza tokens del tenant previo.
- Roles internos exactos: `ADMIN`, `SUPERVISOR`, `COLLECTOR`.
- Jamás almacenar contraseñas en texto plano ni incluirlas en logs; usar mecanismo seguro de restablecimiento y bloqueo/limitación de intentos de login.
- Un usuario no accede a otra organización por modificar URLs, IDs, tokens o parámetros.

### 5.2 Contrato JWT obligatorio

La decisión vigente es usar exclusivamente JWT. `MS.Cobroup.BackendAdmin` es el único emisor; `MS.Cobroup.Backend` valida tokens y jamás recibe la clave privada. El contrato detallado y auditable está en `DECISION_AUTENTICACION_JWT.md` y prevalece sobre referencias históricas.

- Access JWT simétrico `HS256`, máximo 15 minutos, con `iss`, `aud`, `sub`, `exp`, `iat`, `nbf`, `jti`, `typ=access`, tenant y rol.
- Refresh JWT `HS256`, `typ=refresh`, audiencia separada, uso único, rotación atómica, familia/replay y máximo 30 días.
- Refresh únicamente en cookie `HttpOnly`, `Secure` y `SameSite`; access token solo en memoria. Prohibido guardar tokens en Web Storage o texto claro en base de datos.
- Validación estricta de firma, algoritmo, `kid`, issuer, audience, tiempo y tipo. Se prohíben claves/defaults de respaldo y desactivar issuer/audience.
- Cada ambiente usa un secreto simétrico distinto administrado desde `cobroup.environments`; el mismo secreto del ambiente se inyecta solo en BackendAdmin (emisión/validación) y Backend (validación). Faltante, Base64 inválido o menos de 32 bytes detienen el arranque. No se usan RSA, PEM/PFX, JWKS ni acknowledge.
- Cambio de organización revoca la familia anterior antes de emitir la nueva sesión.
- El cierre exige las pruebas negativas, de concurrencia, cross-tenant, rotación de clave e integración por ingress definidas en la decisión.

### 5.3 Alcance por rol

**Cobrador (`COLLECTOR`)**: solo accede a rutas expresamente asignadas y a clientes pertenecientes a esas rutas; puede crear clientes indicando una de sus rutas, consultar y actualizar los datos permitidos, registrar ubicaciones, crear préstamos asociados a clientes visibles y cuotas válidas, registrar pagos, adjuntar foto de sus cobros y finalizar **exclusivamente** cuando la deuda haya sido cubierta por pagos ordinarios o anticipados. No puede crear rutas, asignarse rutas, crear etiquetas/medios/plantillas, aprobar excepciones, borrar evidencia ni condonar deuda.

**Supervisor (`SUPERVISOR`)**: posee las operaciones del cobrador **dentro de las rutas que supervisa o tenga formalmente habilitadas**. Además consulta actividad, pagos, cambios, moras, cronogramas y acciones de los cobradores de esas rutas, con filtros y auditoría. No configura globalmente la organización ni aprueba excepciones individuales. La visibilidad de todas las rutas de la organización no se presupone para este rol.

**Administrador (`ADMIN`)**: visibilidad y operación sobre toda su organización; posee capacidades operativas y de supervisión; es el único autorizado a crear/editar/desactivar rutas, definir cupos, asignar o reasignar cobradores y supervisor de ruta, configurar medios de pago, etiquetas, plantillas de cuantías y valores base, aprobar/rechazar condiciones excepcionales individuales, retirar evidencias de acuerdo con procedimiento y consultar todos los reportes y mapas de su tenant. El administrador no puede acceder a otro tenant.

### 5.4 Matriz de permisos mínimos

| Acción | Cobrador | Supervisor | Administrador |
|---|:---:|:---:|:---:|
| Ver clientes, préstamos y agenda de rutas en alcance | Sí | Sí | Sí |
| Crear clientes y modificar sus datos permitidos | Sí | Sí | Sí |
| Crear préstamos dentro de plantilla autorizada | Sí | Sí | Sí |
| Solicitar excepción individual de préstamo | Sí | Sí | Sí |
| Aprobar/rechazar excepción individual | No | No | Sí |
| Registrar cobros / adelantar pago | Sí | Sí | Sí |
| Cerrar préstamo por saldo completamente pagado | Sí | Sí | Sí |
| Adjuntar evidencia a cobro propio/autorizado | Sí | Sí | Sí |
| Eliminar/retirar evidencia de cobro | No | No | Sí |
| Asignar etiquetas a clientes | Según etiqueta | Según etiqueta | Sí |
| Consultar auditoría de cobradores de sus rutas | No | Sí | Sí |
| Crear y configurar rutas, cupos y asignaciones | No | No | Sí |
| Crear/editar/desactivar medios, plantillas y etiquetas | No | No | Sí |
| Ver mapa y recaudo global de la organización | No | En sus rutas | Sí |

La autorización es combinada: **rol + tenant + relación con ruta/cliente/acción + estado del recurso + políticas de etiqueta**. No basta verificar el nombre del rol.

## 6. Clientes

### 6.1 Datos y validaciones

- `firstNames` y `lastNames`: obligatorios.
- `businessOrWorkName`: opcional; nombre de comercio, empresa, trabajo o local.
- `phones`: **de 1 a 6** teléfonos válidos. Evitar duplicados internos; permitir marcar uno principal.
- `locations`: **de 1 a 4** ubicaciones. Captura manual en mapa o botón para obtener GPS actual con permiso del usuario. Cada ubicación tiene latitud, longitud, alias/dirección descriptiva opcional, indicador de principal y marca de origen/manual-GPS.
- `observations`: texto opcional de observaciones operativas sujeto a normas de tratamiento de datos.
- `tags`: cero o más etiquetas válidas de la organización.
- `routeId`: obligatorio para cliente activo dentro de una ruta; al crearlo el cobrador elige una ruta **de sus asignaciones**.
- Estado del cliente, fecha de alta y registro de cambios.

### 6.2 Reglas

1. No persistir un cliente activo sin al menos un teléfono y una ubicación.
2. Limitar de forma transaccional los cupos efectivos de clientes activos por ruta; validar antes de crear, activar o trasladar un cliente.
3. El administrador puede cambiar la ruta de un cliente conforme a permisos y límites; registrar origen, destino, autor y fecha. Definir el tratamiento de préstamos activos y asignaciones al trasladar en una decisión específica.
4. La captura de ubicación actual es voluntaria y requiere permiso del navegador; ofrecer selección manual si se deniega, falla o no existe GPS.
5. No mostrar clientes de rutas ajenas a cobradores/supervisores. Filtrar también búsquedas, autocompletados, mapas, adjuntos y exportaciones.
6. Los cambios de etiquetas requieren autorización específica por etiqueta.

## 7. Plantillas de cuantías y excepciones

### 7.1 Plantillas preconfiguradas por tenant

El administrador configura al menos dos categorías iniciales (`LOW` baja cuantía y `HIGH` alta cuantía) y puede crear **tantas plantillas personalizadas reutilizables** como desee: por ejemplo, «cuantía media», «20 días» o «30 días». Una plantilla puede incluir:

- ID, organización, nombre, descripción, clasificación `LOW`, `HIGH` o `CUSTOM_TEMPLATE`, estado y versión.
- `maxLoanDays`: duración máxima permitida (> 0).
- `allowedPercentages`: uno o varios porcentajes autorizados (ejemplo: 10 %, 15 % o 20 %), configurados de modo exacto y con reglas de selección.
- `maxPrincipalAmount`: monto máximo del préstamo (decimal monetario > 0).
- Parámetros de ruta heredados o sobreescritos solo cuando la organización permita ese tipo de ajuste.

**Punto esencial:** una plantilla `CUSTOM_TEMPLATE` aprobada y publicada por el administrador **no exige una nueva aprobación administrativa por cada préstamo** que respete íntegramente sus límites y porcentajes. La palabra «personalizada» en el nombre de la plantilla no implica revisión manual.

### 7.2 Condiciones excepcionales para un cliente

Cuando se soliciten monto, duración, tasa o condiciones **fuera de la plantilla aprobada aplicable**, se crea una `LoanExceptionRequest` vinculada a un **cliente concreto**, con términos propuestos, motivo, solicitante, estado, fechas y decisión. Solo el `ADMIN` del tenant puede aprobarla o rechazarla. No desembolsar/activar el préstamo excepcional antes de su aprobación. Una aprobación se aplica a esa solicitud/cliente/préstamo, **no amplía las plantillas generales**.

Estados recomendados: `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED`, con trazabilidad de actor, justificación y términos aprobados. Evitar aprobación por parte del mismo rol no autorizado aunque manipule la API.

### 7.3 Cambios de plantillas

Cambiar la configuración o desactivar una plantilla afecta nuevos préstamos, **nunca reescribe principal, porcentaje, total ni calendario ya pactado**. Conservar ID y versión/snapshot de la plantilla utilizada.

## 8. Préstamos y amortización

### 8.1 Datos obligatorios

- Tenant, ID, cliente obligatorio, ruta de operación, creador y fechas.
- Plantilla de cuantía seleccionada y snapshot de términos efectivos (incluida versión y fuente de herencia).
- Principal desembolsado, porcentaje elegido permitido, cargo/valor porcentual pactado, total a pagar, total pagado y saldo pendiente.
- Fecha de inicio/desembolso, máximo de días, fecha límite permitida y período de cobro.
- Cuotas generadas al inicio: número, importe, fecha de vencimiento, importe cubierto y estado.
- Estado del préstamo, referencia opcional de aprobación de excepción y auditoría.

### 8.2 Periodicidad

- `DAILY`: cobro diario.
- `WEEKLY`: cobro semanal indicando **día de la semana**.
- `EVERY_N_DAYS`: cobro cada `N` días enteros > 0; registrar además los días de semana habilitados **si la organización elige esa restricción**. Antes de implementar, cerrar la interpretación exacta de la regla combinada (§17).

El cálculo del cronograma utiliza calendario **local de la organización**; timestamps auditables se almacenan en UTC. Respetar límite máximo de días; no generar cuotas fuera de la duración permitida.

### 8.3 Cálculo financiero inicial propuesto

Como hipótesis a confirmar: cargo **simple fijo sobre el principal** para toda la duración, sin interés compuesto ni cargos moratorios implícitos:

```text
cargo = principal × porcentaje / 100
totalPactado = principal + cargo
saldo = totalPactado − sum(pagos válidos aplicados)
```

Por ejemplo, principal **100.000** con porcentaje **20 %** da cargo **20.000** y total pactado **120.000**. Si el calendario genera 20 cuotas, serían 20 cuotas de 6.000, ajustando la última por residuos de redondeo si corresponde. **La selección del porcentaje se hace entre los configurados para esa plantilla**, no se asume una tasa por defecto.

- Usar aritmética decimal; prohibido `float`/`double` para importes.
- Definir precisión monetaria y redondeo por moneda/tenant; suma de cuotas = total pactado exacto.
- Cuotas determinadas al originar el préstamo; los pagos parciales o anticipados **no reescriben su valor original**, pero actualizan pendiente/cobertura y las próximas obligaciones aplicables.
- No permitir registrar pagos que superen el saldo sin una política de excedentes expresamente aprobada.
- Reglas de imputación de pagos: por defecto cubrir primero las cuotas vencidas y luego las siguientes, documentando el algoritmo y sus pruebas.
- Los préstamos en estado pendiente de aprobación no aparecen como obligación cobrable.

### 8.4 Estados y finalización

Estados sugeridos: `PENDING_APPROVAL` (si aplica), `ACTIVE`, `PAID_OFF`, `CANCELLED` (solo flujo autorizado y documentado). **Mora es un estado derivado del cronograma, no una forma alternativa de cerrar un préstamo.**

- Cerrar como `PAID_OFF` únicamente si `saldo == 0`, ya sea por pagos programados o anticipados.
- Un cobrador no puede marcar arbitrariamente «finalizado», condonar, perdonar o modificar saldo para forzar un cierre.
- Un préstamo puede seguir `ACTIVE` y encontrarse en mora.
- Cancelaciones extraordinarias, condonaciones y anulaciones **no se implementan** hasta definir permisos y contabilidad.

## 9. Cobros, pagos y evidencias

### 9.1 Registro de cobro

Cada pago contiene ID, tenant, préstamo, cliente, ruta, cobrador/actor, importe decimal, medio de pago activo de su organización, fecha/hora de registro UTC, fecha de negocio local, cuotas cubiertas, estado y referencias de evidencia. El **medio de pago es informativo**: no se integra automáticamente con un banco ni certifica que el dinero haya sido efectivamente consignado.

Permitir pagos parciales y anticipados. Calcular el nuevo saldo en una transacción del servidor; validar actor, tenant, monto > 0, préstamo cobrable, medio permitido y restricciones de saldo. Habilitar clave de idempotencia, concurrencia segura y control de reintentos para evitar pagos duplicados por mala conectividad.

### 9.2 Fotografía opcional

- Al cobrar se puede **tomar foto, seleccionar de galería o cargar archivo**; el cobro no exige fotografía.
- El cobrador autorizado puede adjuntar evidencia **después del pago**, mientras el pago siga disponible y las reglas de alcance lo permitan.
- Después de subir una imagen, **cobradores y supervisores no pueden eliminarla**. Solo el administrador puede solicitar retiro, con motivo y registro auditable.
- Preferir eliminación lógica/restricción de acceso con preservación conforme a la política de retención y legislación aplicable; no borrar silenciosamente huellas de auditoría.
- Validar MIME real, tamaño, extensión y contenido; almacenar archivos privados, servir con URLs/autorizaciones temporales, evitar exponer identificadores entre tenants y proteger metadatos sensibles (por ejemplo, EXIF GPS cuando no se necesite).

### 9.3 Correcciones

No editar ni eliminar directamente pagos confirmados en una tabla para «corregir saldo». Diseñar posteriormente un flujo administrativo de ajuste/reversión trazable; conservar el movimiento original y efecto compensatorio. Registrar quién, cuándo, por qué y valores involucrados. Hasta definir ese flujo, no habilitar anulaciones en producción.

## 10. Medios de pago y control diario

- El administrador crea, edita y **desactiva** cuantos medios necesite por organización (ej.: efectivo, transferencia u otro); ID, nombre, orden, estado.
- Cada cobro captura el medio de pago utilizado. Los medios desactivados no se seleccionan en nuevos cobros; los históricos mantienen nombre/referencia inmutable o snapshot legible.
- El administrador consulta total monetario y cantidad de pagos por medio en fechas seleccionadas, con vista diaria, filtros por ruta, cobrador y zona horaria de negocio; desglosar reversos autorizados por separado.
- Totales calculados sobre pagos válidos, con trazabilidad del período, moneda y regla de cierre diario.
- Es un **control de recaudo registrado**, no una conciliación de cuentas bancarias ni prueba de efectivo realmente entregado.

## 11. Etiquetas

- Solo el administrador crea/edita/desactiva etiquetas, por tenant: ID, **nombre, color**, `assignableBySupervisor`, `assignableByCollector`, estado y auditoría.
- `ADMIN` siempre puede asignar o retirar etiquetas de sus clientes.
- `SUPERVISOR` solo si `assignableBySupervisor = true` y tiene alcance sobre el cliente.
- `COLLECTOR` solo si `assignableByCollector = true` y tiene alcance sobre el cliente.
- El color es atributo visual, no fuente de autoridad. Validar contraste mínimo y pertenencia a tenant.
- Cambiar permisos afecta futuras asignaciones; conservar historial de quién asignó, retiró o modificó.

## 12. Rutas y georreferenciación

### 12.1 Gestión de rutas

El administrador crea/modifica/desactiva rutas, con ID, tenant, nombre, descripción, estado, clientes activos vinculados, cobradores asignados, **supervisor responsable**, límites efectivos de clientes y préstamos activos, reglas heredadas/sobreescritas y fechas. Solo el administrador puede cambiar asignaciones; cada cambio se audita y toma efecto para los accesos nuevos.

- Permitir **configuración manual**: elegir clientes, ordenarlos y ajustar recorrido.
- Permitir **configuración asistida por georreferenciación**: proponer agrupación/orden con coordenadas existentes, capacidad y proximidad; requiere confirmación del administrador. No asignar clientes automáticamente sin revisión si hay ambigüedades.
- Mantener como mínimo una ubicación por cliente activo. Elegir ubicación principal o preferida para el mapa y la sugerencia de recorrido.
- Una ruta acepta múltiples cobradores asignados; definir un supervisor responsable por ruta, salvo decisión posterior distinta.
- El cobrador ve sus rutas; el supervisor ve rutas supervisadas; el administrador todas las de su tenant.
- Los cupos efectivos se validan en servidor al crear, trasladar o activar clientes y préstamos. Definición inicial: contar **clientes activos** y **préstamos `ACTIVE`**, evitando contar préstamos pagados. Registrar si se propone otra métrica.

### 12.2 Visualización

- Mapa administrativo: marcadores de clientes activos según filtro de ruta, ubicación principal, importe vencido, estado al día/en mora y cobros del día.
- Posibilidad de inspeccionar clientes, préstamos pendientes y avance de ruta.
- No confundir posición **del cliente** con posición **en tiempo real del cobrador**. La última solo podrá implementarse como funcionalidad nueva con consentimiento, finalidad, retención y permisos explícitos.
- Proveedor de mapas, límites de cuota, costos, cache y términos de licencia requieren decisión técnica antes de integrarlo.

## 13. Agenda diaria, vencimientos y mora

La PWA operativa presenta para cada ruta autorizada:

- **Cobrar hoy**: cuotas con vencimiento hoy y saldo por cubrir.
- **Vencidos/en mora**: cuotas con vencimiento anterior a hoy y saldo pendiente; mostrar días vencidos, cliente, ubicación y total pendiente.
- **Pagados hoy**: pagos aplicados en la fecha local de negocio, con monto y medio.
- **Próximos**: vencimientos posteriores, según filtros.

Regla sugerida para la etiqueta del cliente: «en mora» si tiene por lo menos una cuota vencida e impaga de algún préstamo activo; «al día» si no tiene mora. **«Ya pagó hoy»** no significa necesariamente «préstamo saldado» ni «sin mora»; representar esas condiciones de forma independiente.

Definir la fecha local a partir de zona horaria del tenant; el paso de día debe ser consistente para usuarios en diferentes husos. Los movimientos se registran con UTC para auditoría.

## 14. Aplicaciones y experiencia de usuario

### 14.1 `MF.cobroup-frontend` — Operativa

Áreas objetivo (implementación en fases posteriores): autenticación, selector de ruta autorizada cuando aplique, inicio con resumen del día, agenda de cobros hoy/vencidos, lista/búsqueda de clientes, detalle de cliente, teléfonos y ubicaciones, alta/edición, mapa/selección GPS, préstamo nuevo, detalle y calendario del préstamo, cobro rápido, captura/adjunto de evidencia, historial de pagos, etiquetas permitidas y vista de supervisión para supervisor. En pantallas pequeñas priorizar navegación, cobro rápido, accesibilidad y estados de conexión/errores.

### 14.2 `MF.cobroup-frontend-admin` — Administrativa

Áreas objetivo: autenticación del tenant, panel global por organización, mantenimiento de usuarios según política definida, plantillas bajas/altas/personalizadas, solicitudes de excepción, medios, etiquetas, rutas, cupos/herencia, asignación de cobradores/supervisores, mapa con filtros por ruta/cliente/estado, seguimiento de moras/pagos diarios, reportes de recaudo por medio y auditoría.

### 14.3 Criterios transversales

- Rutas protegidas en frontend **y autorización real en API**.
- Responsive y PWA instalable según el estándar, manifest, iconos y estrategia de actualización; HTTPS en despliegue.
- Modo claro inicialmente, tema oscuro configurable, tokens y componentes reutilizables dentro de los límites del estándar.
- Permisos GPS/cámara solicitados en el momento de uso; fallback a selección/carga manual si procede.
- Mostrar estado de sincronización y evitar afirmar «pago exitoso» hasta confirmación de API. Las operaciones financieras se asumen **online** hasta diseñar explícitamente una cola offline segura.
- Búsqueda y paginación sin cargar información de rutas ajenas, manejo de errores accesible, internacionalización según necesidad futura.

## 15. Dominio mínimo y fronteras de backend

### 15.1 Entidades/capacidades sugeridas

| Dominio | Entidades o agregados a diseñar |
|---|---|
| Organización | `Organization`, `OrganizationSettings`, `RouteSettingOverride` |
| Identidad | `User`, `UserRole`, `RouteCollectorAssignment`, `RouteSupervisorAssignment`, `Session` |
| Clientes | `Customer`, `CustomerPhone`, `CustomerLocation`, `CustomerTag` |
| Cuantías | `LoanTemplate`, `LoanTemplateVersion`, `LoanExceptionRequest` |
| Préstamos | `Loan`, `LoanTermsSnapshot`, `Installment`, `InstallmentPaymentAllocation` |
| Cobros | `Payment`, `PaymentMethod`, `PaymentEvidence`, `PaymentAdjustment` (futuro autorizado) |
| Rutas | `Route`, `RouteMembershipHistory`, `RouteAssignmentHistory` |
| Etiquetas | `Tag`, `TagAssignmentHistory` |
| Auditoría | `AuditEvent`, `OutboxEvent` (si se selecciona integración asíncrona) |

Estas son **propuestas de diseño**, no una orden de implementar todas las tablas en fase 1. Resolver cardinalidades, ownership y ciclo de vida en la fase de modelo de dominio.

### 15.2 Fronteras de propiedad

- `MS.Cobroup.Backend` es autoridad de escrituras de **Customer, Loan, Installment, Payment, PaymentEvidence** y de sus transacciones.
- `MS.Cobroup.BackendAdmin` es autoridad de **OrganizationSettings, Route, Assignment, LoanTemplate, PaymentMethod, Tag, LoanExceptionApproval** y vistas administrativas/auditoría según contratos.
- Para lectura de mapas, moras y reportes, el backend admin puede consumir contratos de consulta del backend operativo o proyecciones mantenidas coherentemente; **no escribir directamente en tablas propiedad del otro servicio**.
- La excepción creada por el backend operativo será aprobada por el backend admin mediante un contrato autenticado; la transición a préstamo cobrable debe ser transaccional/consistente y tolerar reintentos.
- La identidad, emisión de tokens y membresías deben tener un único responsable lógico entre ambos backend (definirlo antes de implementar autenticación real). No crear dos bases de contraseñas ni dos autoridades de sesión independientes.
- Si se comparte infraestructura física de datos inicialmente, mantener ownership lógico, esquemas o delimitación clara por servicio y prohibición de escrituras cruzadas. La estrategia de persistencia se decidirá con un ADR.

### 15.3 Contratos mínimos por versión (para diseñar, no implementar todos en fase 1)

- Operaciones: clientes, ubicaciones, préstamos, cuotas, pagos, evidencias, agenda y consulta del alcance del usuario.
- Administración: organizaciones/configuración, plantillas y porcentajes, excepciones, rutas, asignaciones, medios, etiquetas, mapa, recaudación y auditoría.
- Contrato de autenticación/claims con `sub`, rol(es), tenant y vigencia, aplicando validación y resolución interna de alcance.
- Contratos de errores, paginación, fecha-hora, moneda, autorización e idempotencia.
- Describir versiones y ejemplos OpenAPI; no asumir endpoints HTTP precisos hasta modelar casos de uso.

## 16. Seguridad, auditoría, cumplimiento y calidad

1. **Seguridad de acceso:** RBAC + alcance de ruta/cliente (ABAC), tenant enforced, tokens válidos, hashing seguro, protección de secretos y rate limit en endpoints sensibles.
2. **Integridad financiera:** almacenamiento decimal; validación y cierre transaccional; concurrencia e idempotencia; pruebas de saldos, repetición, redondeo, pago anticipado y vencimientos.
3. **Evidencia privada:** controles de subida, almacenamiento no público, autorización de lectura/eliminación administrativa y registro de cada acción.
4. **Auditoría:** para cada creación, edición, aprobación, rechazo, asignación/reasignación, pago, foto y retiro, registrar actor, tenant, entidad, acción, timestamp UTC y valores permitidos antes/después, protegiendo datos sensibles. Evitar registro de contraseñas, tokens y fotografías en logs.
5. **Aislamiento:** probar cross-tenant en cada API, búsqueda, consulta agregada, archivo y mapa; evitar IDs que permitan enumeración sin autorización.
6. **Privacidad y cobro responsable:** documentar autorización de tratamiento de datos, finalidad de georreferenciación y fotos, retención, acceso/retiro y políticas de contacto respetuosas. Verificar obligaciones legales del país de operación antes de producción (por ejemplo, protección de datos, límites de tasa, contratos y prácticas de cobranza). No implementar prácticas coercitivas.
7. **Disponibilidad:** health checks, logs estructurados, correlación, configuración por entorno, migraciones trazables, copias de seguridad y plan de recuperación.
8. **Pruebas:** unitarias de negocio, integración para transacciones, seguridad de permisos multi-tenant, E2E crítico de alta cliente → préstamo → cuota → pago → saldo y aceptación por rol.
9. **Observabilidad:** medir errores de cobro, latencia, duplicados evitados, pagos por método, préstamos en mora y acciones administrativas; no exponer información sensible en telemetría.
10. **No confiar en el frontend:** todas las reglas de límite, permisos, aprobaciones y consistencia se aplican en backend.

## 17. Decisiones pendientes que requieren definición antes de sus módulos

| ID | Decisión | Criterio de resolución |
|---|---|---|
| `ADR-001` | Identidad y aprovisionamiento del primer tenant/admin | Definir bootstrap sin crear silenciosamente un cuarto rol público |
| `ADR-002` | Motor de base de datos y separación de persistencia | Evaluar PostgreSQL y soporte geoespacial (p. ej. PostGIS) frente a requisitos operativos |
| `ADR-003` | Fórmula de porcentaje, impuestos, límites legales, redondeo y política de amortización | Validación financiera y legal antes de activar préstamos reales; ejemplo simple de §8 no reemplaza aprobación |
| `ADR-004` | Significado exacto de `EVERY_N_DAYS` más selección de días de semana | Definir algoritmo, casos límite y fechas base, conservando máximo de días |
| `ADR-005` | Semántica del cupo de préstamos por ruta y traslado de clientes con préstamos activos | Acordar conteos, reorganización, préstamos concurrentes y continuidad de agenda |
| `ADR-006` | Proveedor de mapas/geocodificación y estrategia de optimización | Precios, licencia, cuotas, privacidad, precisión, escenarios sin GPS |
| `ADR-007` | Pagos parciales, adelantos, excepciones por excedentes, reversos y cierre | Contrato formal, balance y pruebas de auditoría |
| `ADR-008` | Disponibilidad offline para operaciones financieras | En fase inicial solo online; diseñar sincronización/idempotencia antes de habilitar cola offline |
| `ADR-009` | Alcance de supervisión y múltiples supervisores por ruta | Inicialmente un supervisor responsable/ruta; definir delegaciones si se solicitan |
| `ADR-010` | Política de acceso, retención y retiro de imágenes y datos personales | Considerar protección de datos y evidencia financiera |
| `ADR-011` | Identificación única y membresías de usuario | **Resuelto parcialmente:** username global único; rol y tenant por membresía; login usa última organización accedida. Normalización telefónica continúa pendiente |
| `ADR-012` | Infraestructura, despliegue, ambientes y recursos compartidos | Documentar por separado sin suponer Kubernetes, Cloudflare o proveedor específicos |

**Prioridad:** estas decisiones no impiden iniciar **la fase 1 de scaffolding** siempre que se documente la incertidumbre y no se codifique comportamiento financiero o legal definitivo.

## 18. Planificación por fases

### 18.1 Fase 1 — ÚNICA fase autorizada para implementar al iniciar

**Objetivo:** inicializar los cuatro repositorios indicados, alineados con sus frameworks, compilables y preparados para evolución por módulos. Sin construir toda la aplicación funcional de una vez.

**Entregables obligatorios en cada repositorio:**

1. Repo/estructura de nombre exacto, en rama `feature/*` autorizada; conservar cualquier contenido previo.
2. Archivo `AGENTS.md` cronológico con origen del proyecto, marco efectivo, alcance fase 1 y evidencia de ejecución.
3. `README.md`, `.gitignore`, carpetas base y `docs/context/INDEX.md` (o equivalentes exigidos por su marco), decisiones/pendientes.
4. Frontend: Angular LTS compatible, TypeScript, SCSS, estructura feature-based, shell/rutas de navegación mínimas, themes Light/Dark, responsividad, configuración de API, base para autenticación y PWA según el marco. Sin datos reales ni flujos de cobro completos.
5. Backend: solución/proyectos `.csproj` separados en hexagonal, host API con health endpoint, DI mínima, configuración por entorno y proyectos de pruebas. Sin crear lógica ficticia de negocio, endpoints CRUD completos ni secretos.
6. Definición documental de fronteras entre los cuatro repositorios y primeros contratos (identidad, errores, tenant, versiones, auth), sin código de integración innecesario.
7. Validaciones reproducibles: instalar dependencias, compilar, ejecutar pruebas disponibles, levantar APIs y frontends y registrar resultados exactos; si falta SDK/Node u otra dependencia, registrar bloqueo sin afirmar éxito.
8. Primer inventario de decisiones pendientes y propuesta fundamentada de **siguientes fases**, a calcular con dependencias/riesgos/esfuerzo, sin asumir fechas ni aprobar desarrollo adicional.

**Criterios de aceptación fase 1:** existen exactamente los cuatro repositorios solicitados (no extras); siguen los estándares reales; los frontends arrancan con shells independientes; APIs arrancan y responden health; pruebas base ejecutadas; `AGENTS.md` en los cuatro registra resultados y pendientes; sin commits automáticos, secretos ni features operativas inventadas.

### 18.2 Fases 2+ — Calculadas durante la ejecución

Después de cerrar fase 1, el agente debe **proponer** el desglose y orden de fases basado en dependencias, seguridad, valor y complejidad. No numerarlas como plan definitivamente aprobado desde este documento. Valorar al menos estos bloques antes de ordenarlos: identidad + tenant; configuración; clientes/rutas; cuantías/excepciones; préstamos/cronograma; pagos/evidencias; agenda/mora; mapas; analítica/auditoría; endurecimiento y despliegue.

Para cada fase propuesta documentar: objetivo, dependencias, repositorios afectados, alcance, criterios de aceptación, casos de prueba, riesgos/decisiones previas, estimación relativa y plan de rollback. **La propuesta no autoriza la siguiente implementación**.

### 18.3 Control de cambios

Cualquier requisito nuevo se registra en `COBROUP.md` (o mediante referencia a decisión/versionado), se refleja en los `AGENTS.md` de repositorios afectados y obtiene alcance/fase antes de escribir código. No ampliar el backlog sin una decisión explícita.

### 18.4 Fase 3 — configuración y rutas completada

La fase 3 comprende configuración organizacional de zona horaria, moneda y cupos generales opcionales; alta, consulta, edición y desactivación de rutas; un supervisor responsable; uno o más cobradores activos; cupos de clientes/préstamos `INHERITED` u `OVERRIDDEN`; y auditoría atómica con detalle antes/después de cada reasignación o cambio. El catálogo de asignación incluye exclusivamente membresías activas `SUPERVISOR` y `COLLECTOR` de la organización derivada del token. Tanto API como Core rechazan escrituras de actores que no sean `ADMIN`; el cliente no envía ni elige un tenant para estas operaciones.

Contratos finales de fase: administración mediante `GET|POST /api/v1/routes`, `PUT /api/v1/routes/{id}`, `GET /api/v1/users/assignable` y `GET|PUT /api/v1/organization-settings`; lectura operativa mediante `GET /api/v1/route-access` en BackendAdmin y `GET /api/operations/v1/routes` en Backend operativo. Para operación se exponen solo rutas activas: todas las del tenant para `ADMIN`, las supervisadas para `SUPERVISOR` y las asignadas para `COLLECTOR`. Las dos PWA consumen estos contratos reales.

La selección y orden de clientes dentro de una ruta y la asistencia por georreferenciación quedan en los bloques de clientes/mapas: requieren primero `Customer`, ubicaciones y la decisión de proveedor, cuota y licencia. Esta dependencia explícita no deja incompleta la autoridad de configuración de rutas de fase 3 y evita implementar datos ficticios o propiedad duplicada.

### 18.5 Línea base consolidada de fases

La planificación vigente contiene **9 fases totales**. Las fases 1–5 están completadas funcionalmente y quedan **4 fases posteriores pendientes**. Esta numeración formaliza los ocho bloques posteriores a la fundación que ya figuraban en los roadmaps de los cuatro repositorios:

| Fase | Bloque | Estado |
|---:|---|---|
| 1 | Fundación y scaffolding | COMPLETA |
| 2 | Identidad y aislamiento tenant | REABIERTA — JWT |
| 3 | Configuración y rutas | CERRADA EN GIT |
| 4 | Clientes y alcance operativo | COMPLETA |
| 5 | Plantillas y excepciones | CERRADA EN GIT |
| 6 | Préstamos y cronograma | CERRADA EN GIT |
| 7 | Pagos, evidencias y conciliación | COMPLETA |
| 8 | Mapas, reportes y auditoría ampliada | PENDIENTE |
| 9 | Endurecimiento y despliegue | PENDIENTE |

El detalle de entregables, evidencia, faltantes reales, decisiones y método de actualización se mantiene en `informe/avance.md`; `informe/avance.html` es su presentación navegable. Las nueve fases tienen funcionalidad versionada, pero identidad permanece reabierta hasta demostrar conformidad JWT.

### 18.6 Fase 4 — clientes y alcance operativo cerrada

La fase implementa `Customer`, entre 1 y 6 teléfonos y entre 1 y 4 ubicaciones, con principal única, coordenadas manuales o GPS y pertenencia obligatoria a una ruta autorizada. `MS.Cobroup.Backend` es la autoridad de escritura y persiste en el esquema PostgreSQL `operations`; consume el cupo efectivo desde BackendAdmin y serializa altas, activaciones y traslados por ruta para impedir sobrecupo concurrente. Cada creación, edición, traslado o cambio de estado deja auditoría atómica con actor, fecha y resumen antes/después. BackendAdmin expone una fachada exclusiva para `Admin` y no escribe tablas operativas. Las dos PWA contienen sus vistas de clientes. Pruebas con PostgreSQL validaron aislamiento cross-tenant, reemplazo de contactos y concurrencia con cupo. La evidencia histórica de identidad no sirve como aceptación del nuevo contrato JWT y debe repetirse antes del cierre. Los criterios mínimos AC-01, AC-02, AC-04, AC-05 y AC-20 permanecen como contrato funcional.

### 18.7 Fase 5 — plantillas y excepciones cerrada en Git

La fase consolida el catálogo inmutable de plantillas de crédito en BackendAdmin (`LoanTemplate` y `LoanTemplateVersion` con versionado en cada actualización y clasificación `LOW`, `HIGH`, `CUSTOM_TEMPLATE`), la fachada operativa de consulta de plantillas publicadas en Backend operativo, la entidad `LoanExceptionRequest` en el esquema PostgreSQL `operations` con montos, plazos y porcentajes positivos, justificación obligatoria y estados `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED`. La autoridad exclusiva de decisión de excepciones corresponde al rol `Admin` (AC-08), probado contra manipulaciones de actores sin rol autorizado. La decisión administrativa registra auditoría atómica en BackendAdmin y es tolerante a reintentos (idempotente). Las plantillas prepublicadas no exigen aprobación previa (AC-07) y las versiones históricas quedan inalteradas para no modificar condiciones de préstamos pactados (AC-09). Ambas PWA integran sus pantallas administrativas y operativas de plantillas y excepciones con builds limpios y suites de pruebas completas (124 pruebas totales). Fase cerrada bajo autorización explícita del usuario mediante commits locales: `MS.Cobroup.Backend` (`8cc1e42`), `MS.Cobroup.BackendAdmin` (`b3b8e34`), `MF.cobroup-frontend-admin` (`b6594be`) y `MF.cobroup-frontend` (`674e62b`).

### 18.8 Fase 6 — préstamos y cronograma cerrada en Git

La fase implementa la originación de créditos vinculada a cliente activo y ruta autorizada (AC-06), capturando un snapshot inmutable de los términos financieros y versión de plantilla (AC-09). Implementa el motor de cronograma de cuotas decimales con redondeo legal para que la suma exacta de cuotas iguale el total pactado (AC-10), frecuencias `Daily`, `Weekly` y `EveryNDays`, estados del préstamo (`Active`, `PaidOff`, `Cancelled`), concurrencia transaccional y validación de cupo de préstamos activos por ruta. En `MS.Cobroup.Backend` se crearon las entidades de dominio `Loan`, `Installment`, `LoanTermsSnapshot`, el repositorio `EfLoanRepository` mapeado al esquema `operations`, la migración EF Core `20261021210019_AddLoansAndInstallments` aplicada a PostgreSQL, y el controlador `LoansController` con previsualización y originación. `MS.Cobroup.BackendAdmin` expone la fachada de consulta `/api/v1/loans` para auditoría y supervisión exclusiva de `Admin`. Ambas aplicaciones frontend (`MF.cobroup-frontend` y `MF.cobroup-frontend-admin`) cuentan con pantallas para originación y simulación de cronogramas, listados y modales de detalle de cuotas con diseño responsivo bajo los presupuestos CSS (< 4.00 kB). Fase cerrada bajo autorización explícita del usuario mediante commits locales: `MS.Cobroup.Backend` (`2537925`), `MS.Cobroup.BackendAdmin` (`57d45d7`), `MF.cobroup-frontend` (`762847f`) y `MF.cobroup-frontend-admin` (`b1f7f48`).

### 18.9 Fase 7 — pagos, evidencias y conciliación cerrada en Git

La fase implementa el registro de pagos ordinarios y cobros parciales/anticipados que descuentan saldo en cascada sobre las cuotas pendientes ordenadas cronológicamente (AC-11), garantía de idempotencia contra doble pago concurrente o por reintento de red (AC-12), catálogo y medios de pago autorizados por tenant (AC-13), soporte de fotografía opcional y retirable exclusivamente por `Admin` de manera auditable (AC-14), y conciliación diaria de recaudo por ruta y medio de pago (AC-17). En `MS.Cobroup.BackendAdmin` se implementó la entidad `PaymentMethod`, repositorio `EfPaymentMethodRepository`, migración EF Core `20261021215708_AddPaymentMethods` aplicada a PostgreSQL, auditoría atómica `AdministrativeAuditRecord.PaymentMethodModified`, y `PaymentMethodsController`. En `MS.Cobroup.Backend` se crearon las entidades `Payment`, `PaymentAllocation`, `PaymentEvidence`, motor de cascada `ApplyPayment` cerrando préstamo al alcanzar saldo cero, repositorio `EfPaymentRepository`, migración EF Core `20261021220458_AddPaymentsAndEvidences` aplicada a PostgreSQL, y `PaymentsController`. En `MF.cobroup-frontend` se creó `PaymentsService`, modal de cobro con cálculo en vivo de imputación en cuotas, captura de evidencia, y pantalla `/conciliacion` con desglose por medio y KPI cards. En `MF.cobroup-frontend-admin` se creó la pantalla `/medios-pago` para administración y activación reactiva de medios de pago. Ambas PWA cumplen estrictamente los presupuestos CSS (< 4.00 kB) sin advertencias y utilizan tokens semánticos del tema. Fase cerrada bajo autorización explícita del usuario mediante commits locales: `MS.Cobroup.Backend` (`03b2748`), `MS.Cobroup.BackendAdmin` (`6d563c9`), `MF.cobroup-frontend` (`f547dd0`) y `MF.cobroup-frontend-admin` (`e80f442`).

### 18.10 Fase 8 — mapas, georreferenciación y analítica cerrada en Git

La fase implementa el cálculo dinámico y estructurado de la agenda diaria de cobro con vencimientos y mora exactos (AC-16), georreferenciación de clientes en mapa con pins semánticos diferenciados por estado (verde = al día / cobrado hoy, amarillo = vence hoy, rojo = en mora, azul = sin cuota hoy), optimización secuencial de ruta por proximidad geográfica mediante la fórmula de Haversine en el dominio, supervisión administrativa unificada de ubicaciones geográficas de rutas activas, y motor analítico ejecutivo de cartera con cálculo en tiempo real de indicadores clave: saldo activo, cartera colocada, mora total, índice PAR% (Portfolio at Risk) y reporte de envejecimiento de deuda (Aging Report) segmentado en tramos estándar (al día 0d, mora 1-7d, mora 8-15d, mora 16-30d y mora >30d). En `MS.Cobroup.Backend` se creó `AgendaService`, el calculador puro de dominio `HaversineGeoCalculator`, `EfAgendaRepository`, y `AgendaController` (`GET /api/operations/v1/agenda/daily`, `POST /api/operations/v1/agenda/optimize-route`, `GET /api/operations/v1/agenda/map-locations`, `GET /api/operations/v1/agenda/portfolio-analytics`). En `MS.Cobroup.BackendAdmin` se crearon `MapSupervisionService`, `AnalyticsAdministrationService`, `OperationalAgendaDirectoryClient` y los controladores `MapSupervisionController` y `AnalyticsController`. En `MF.cobroup-frontend` se implementó el componente reutilizable `map-view`, la vista `/agenda` con reordenamiento GPS y acceso directo a cobro, y la vista `/mapa` de ruta con mapa interactivo y pins semánticos. En `MF.cobroup-frontend-admin` se implementó `/mapa` con supervisión multi-ruta y filtrado dinámico, y el dashboard ejecutivo `/analitica` con tarjetas KPI, tabla y barras de envejecimiento de cartera y ranking comparativo de rutas por riesgo. Fase cerrada bajo autorización explícita del usuario mediante commits locales: `MS.Cobroup.Backend` (`cf65b00`), `MS.Cobroup.BackendAdmin` (`ed609a5`), `MF.cobroup-frontend` (`ddfe552`) y `MF.cobroup-frontend-admin` (`db29c9a`).

### 18.11 Fase 9 — endurecimiento y despliegue K3s/Kubernetes cerrada en Git

La fase final implementa el endurecimiento integral para producción, empaquetado OCI y despliegue automatizado en K3s / Kubernetes para los ambientes `cobroup-dev`, `cobroup-qa` y `cobroup-prod`:
1. **Health Checks y observabilidad backend:** en `MS.Cobroup.Backend` y `MS.Cobroup.BackendAdmin` se implementaron verificaciones nativas de liveness (`/health/live`) y readiness (`/health/ready` con chequeo activo de conectividad a PostgreSQL mediante `OperationsDatabaseHealthCheck` e `IdentityDatabaseHealthCheck`).
2. **Reverse Proxy e Ingress:** configuración de `UseForwardedHeaders` (X-Forwarded-For, X-Forwarded-Proto) en ambos backend para soporte de Traefik e Ingress Controller.
3. **Seguridad HTTP:** middleware de cabeceras de seguridad (`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, `X-XSS-Protection: 0`) y rastreo de trazabilidad con `X-Correlation-ID`.
4. **Contenedorización multi-stage:** Dockerfiles y `.dockerignore` unificados para los 4 repositorios:
   - Backend: SDK .NET 10 -> ASP.NET 10 runtime, ejecución como usuario sin privilegios (`1000:1000`), puerto 8080.
   - Frontend: Node 22-alpine -> Nginx 1.27-alpine, usuario no privilegiado `nginx`, puerto 8080, compresión gzip, fallback SPA `try_files` y endpoint `/health`.
5. **Servicios PWA:** `PwaUpdateService` (notificación de nueva versión con `SwUpdate`) y `PwaInstallService` (manejo de evento `beforeinstallprompt`) en ambas PWA.
6. **Manifiestos K3s (`k8s/`):** Namespace dinámico por ambiente (`cobroup-dev`, `cobroup-qa`, `cobroup-prod`), Deployments y ClusterIP Services para los 4 servicios con límites de CPU/memoria y probes de liveness/readiness, ConfigMap y Secret templates, Ingress Traefik con TLS cert-manager, y script orquestador `k8s/deploy-k3s.sh`.
7. **Pipelines CI/CD (.github/workflows/ci-cd.yml):** integración continua en los 4 repositorios con triggers en `dev`, `qa`, `prod`, `main` y `workflow_dispatch`. Ejecución de pruebas automatizadas, compilación y publicación de imágenes en GitHub Container Registry (`ghcr.io`), y despliegue por SSH en el servidor K3s aplicando variables (`vars.*`) y secretos (`secrets.*`) de GitHub según el ambiente correspondiente. 225 pruebas automatizadas pasando al 100%. Fase cerrada bajo autorización explícita del usuario mediante commits locales: `MS.Cobroup.Backend` (`e02bd65`), `MS.Cobroup.BackendAdmin` (`91152b1`), `MF.cobroup-frontend` (`c3f007f`) y `MF.cobroup-frontend-admin` (`d218f03`).

## 19. Formato obligatorio de `AGENTS.md`

Cada repositorio mantiene al menos:

```markdown
# AGENTS — <nombre exacto del repositorio>
## Identidad y restricciones del proyecto
- Producto: COBROUP
- Rol del repositorio: ...
- Documento maestro: ../COBROUP.md (ajustar ruta real)
- Estándar utilizado: ... (versión y ruta efectivas)
- Rama activa requerida: feature/*
- Regla: prohibidos commits, push y cambios fuera del alcance autorizado

## Estado actual
- Fase: 1
- Resultado: PENDIENTE | EN_CURSO | COMPLETADA | BLOQUEADA
- Pruebas y compilación: ...
- Decisiones pendientes: ...

## Historial cronológico (append-only)
### 2026-10-21 — Inicio
- Prompt / solicitud: ...
- Alcance autorizado: ...
- Decisiones: ...
- Archivos creados/modificados: ...
- Comandos ejecutados y resultados: ...
- Problemas y bloqueos: ...
- Próxima acción sugerida: ...
```

Prohibido borrar entradas previas para ocultar fallas o decisiones. Mantener resumen sincronizado y trazabilidad entre repositorios cuando cambien contratos.

## 20. Reglas de aceptación funcional futura (contratos de comportamiento)

- **AC-01**: un cobrador no puede consultar un cliente de ruta no asignada ni mediante ID directo.
- **AC-02**: un supervisor puede consultar la actividad de cobradores de sus rutas, no de otra organización.
- **AC-03**: solo ADMIN crea rutas y asigna supervisores/cobradores.
- **AC-04**: cliente sin 1 teléfono o 1 ubicación es rechazado; más de 6 teléfonos o 4 ubicaciones es rechazado.
- **AC-05**: alta/traslado de cliente no supera cupo efectivo de clientes activos de ruta.
- **AC-06**: préstamos siempre referencian cliente del tenant y del alcance autorizado.
- **AC-07**: préstamo con plantilla personalizada prepublicada válida **no** exige aprobación extra.
- **AC-08**: excepción individual no puede activarse sin aprobación ADMIN.
- **AC-09**: cambio de plantilla o herencia no altera condiciones de préstamos existentes.
- **AC-10**: suma de cuotas originadas = total pactado exacto y no excede máximo de días.
- **AC-11**: cobro parcial/anticipado actualiza saldo; préstamo se cierra solo con saldo cero.
- **AC-12**: doble reintento con la misma clave de idempotencia no genera doble pago.
- **AC-13**: medio de pago seleccionado debe pertenecer al tenant y figurar permitido.
- **AC-14**: fotografía es opcional en el cobro y adjuntable después; solo ADMIN puede retirarla de manera auditable.
- **AC-15**: permisos de etiqueta se verifican por rol y bandera de la etiqueta.
- **AC-16**: mapa y agenda muestran vencimientos y mora correctos, según zona horaria del tenant.
- **AC-17**: totales diarios por medios de pago concilian con pagos válidos registrados para ese día local.
- **AC-18**: modo de herencia de ruta aplica cambios globales a valores heredados, pero no sobreescritos.
- **AC-19**: los cuatro repositorios cuentan con `AGENTS.md` y no hay commits no autorizados.
- **AC-20**: ningún usuario obtiene información de otro tenant en API, mapas, fotos, archivos o reportes.
- **AC-21**: el login no solicita organización; abre la última membresía activa utilizada, permite cambiar de organización reemitiendo sesión y no emite token tenant cuando no existe membresía activa.
- **AC-22**: una ruta heredada refleja el cupo organizacional vigente; una ruta sobrescrita conserva su valor positivo, y toda reasignación/cambio de estado o cupo deja auditoría con actor y antes/después.
- **AC-23**: autenticación cumple integralmente `DECISION_AUTENTICACION_JWT.md`: firma exclusiva `HS256`, secreto fuerte y distinto por ambiente bajo custodia GitHub/Kubernetes, validación estricta, access corto en memoria, refresh JWT en cookie segura, hash server-side, rotación/replay atómicos, cambio de tenant con revocación y pruebas por ingress. BackendAdmin es el único emisor lógico y Backend usa el secreto solo para validar.
- **AC-24**: almacenamiento de evidencias y archivos cumple integralmente `DECISION_ALMACENAMIENTO_ARCHIVOS.md`: persistencia actual en PersistentVolumeClaim (PVC) de Kubernetes con puerto hexagonal `IFileStorageService` desacoplado de la infraestructura de almacenamiento (diseñado para evolución futura a AWS S3 mediante DI/configuración sin alterar lógica ni contratos), claves lógicas relativas seguras en base de datos, soporte de modo `ReadWriteMany` en producción con dos réplicas, streaming seguro con magic bytes y escritura atómica.

## 21. Instrucción para cualquier agente de desarrollo

1. Leer **completo** este documento y los dos frameworks **reales** que correspondan al repositorio antes de proponer o modificar código.
2. Comprobar ruta, estado Git y rama `feature/*`; si se carece de autorización para crear/cambiar rama, no escribir.
3. Tratar estas reglas como fuente de requisitos de producto; marcar hipótesis o propuestas por separado.
4. En la primera ejecución implementar **únicamente fase 1**. No implementar CRUD, préstamo real, motor de pagos ni mapa integrado por entusiasmo.
5. Aplicar los estándares por tipo de repositorio y evitar dependencias circulares, duplicación de ownership y filtraciones multi-tenant.
6. Ejecutar pruebas/validaciones disponibles y registrar resultados verificables en `AGENTS.md` de cada repositorio.
7. Presentar el resumen por repositorio, problemas, decisiones abiertas y propuesta de próximas fases **para aprobación**.
8. No hacer commit/push/merge/tag/PR salvo petición expresa.

---

**Control de versión:** versión 1.10.0 reemplaza definitivamente RSA/JWKS por firma simétrica `HS256` con secreto independiente por ambiente y backlog específico `BACKLOG_SEGURIDAD_FIRMA_SIMETRICA.md`. Las versiones 1.9.0/1.9.1 y ADR-0006 se conservan solo como trazabilidad histórica. La versión 1.8.0 registra `DECISION_ALMACENAMIENTO_ARCHIVOS.md`; la versión 1.7.0 registró la adopción de JWT exclusivo.
