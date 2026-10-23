# FRAMEWORK_DESARROLLO_CSHARP

> **Versión del estándar:** 2.0  
> **Fecha de actualización:** 2026-10-21  
> **Ámbito:** proyectos **personales** de backend C#/.NET, exclusivamente **Microservices (`MS`)** y **Worker Services (`WS`)**.  
> **Aplicación:** creación, mantenimiento, revisión y auditoría de repositorios propios, tanto por desarrolladores como por agentes de IA.  
> **Naturaleza:** marco técnico personal, independiente de cualquier empresa, marca, cliente o identidad gráfica corporativa.

## 1. Propósito, alcance y prohibiciones

Este marco estandariza la creación y evolución de **microservicios, APIs REST, servicios HTTP, workers y procesos en segundo plano** en .NET. Incluye tanto servicios expuestos por HTTP como componentes que procesan mensajes, ejecutan tareas programadas, sincronizan datos o realizan procesamiento asíncrono sin ofrecer endpoints de negocio.

**Tipos de solución admitidos:**

- **`MS` — Microservice:** servicio backend autónomo. Puede usar un host `Api`, un host `Worker`, o ambos cuando la separación operacional esté justificada.
- **`WS` — Worker Service:** proceso autónomo en segundo plano; usa `BackgroundService`, `IHostedService`, consumidores de colas, trabajos periódicos o sus equivalentes, según los requerimientos.
- **Workers internos en un `MS`:** un microservicio puede contener tareas hospedadas en su API o un proyecto ejecutable `MS.<Nombre>.Worker` separado. La elección depende del ciclo de vida, escalado, despliegue y aislamiento requeridos.

**Fuera de alcance, sin excepciones dentro de este marco:**

- Frontend web o móvil; Angular, React, Vue, Blazor UI y tecnologías similares.
- WPF, Windows Forms, .NET MAUI, aplicaciones de escritorio, ventanas y paneles de configuración gráficos.
- Proyectos `Presentation`, `Config` visual, vistas, view models, recursos XAML, componentes CSS, estilos, tokens visuales o sistemas Light/Dark.
- Referencias, branding, URLs, paletas, plantillas o convenciones específicas de **AlphaTech** u otra empresa.

No se debe **crear ni sugerir un frontend como parte de la estructura**. Si en el futuro se solicita una interfaz gráfica, deberá gestionarse mediante **otro estándar**, no mediante este documento. La documentación de contratos HTTP mediante OpenAPI no constituye la incorporación de un frontend del producto; cualquier interfaz web de documentación se habilita solo si se solicita.

### 1.1. Reglas no negociables

1. **Rama `feature/*` obligatoria:** antes de editar cualquier archivo del repositorio, verificar que la rama activa comience por `feature/`. En caso contrario, solicitar el nombre de rama o autorización expresa para crear/cambiar a una. La inspección de solo lectura se permite.
2. **Arquitectura hexagonal obligatoria:** `Domain`, `Core`, `Shared`, `Infrastructure` y el/los hosts necesarios son **proyectos `.csproj` independientes**.
3. **Sin `src/`:** las bibliotecas y hosts están directamente en la raíz del repositorio, junto a la solución `.slnx` o `.sln`. Solo los proyectos de pruebas se agrupan bajo `tests/`.
4. **Plataforma:** utilizar la versión **LTS estable de .NET** y la versión estable compatible de C#, salvo impedimento técnico documentado.
5. **Límites por contratos:** las interfaces/puertos separan reglas de negocio de transportes, persistencia e integraciones.
6. **Responsabilidad de cada capa:** las reglas de negocio no se implementan en controllers, endpoints, workers, adaptadores, repositorios ni clases de arranque.
7. **Organización interna semántica:** las clases se ubican en subcarpetas según su responsabilidad; no dejar clases de aplicación sueltas en la raíz de los `.csproj`.
8. **Pruebas:** incluir `Core.Tests` e `Infrastructure.Tests`; ejecutar y reportar las verificaciones que correspondan.
9. **`AGENTS.md` obligatorio:** crear y mantener el registro cronológico de prompts, decisiones, cambios, resultados y limitaciones.
10. **No crear commits, pushes, merges, rebases, tags o PR** sin petición expresa. Implementar no implica autorización para confirmar cambios en Git.
11. **No ampliar el alcance:** no inventar módulos, endpoints, entidades, colas, jobs, bases de datos, proveedores, capas o infraestructura por iniciativa propia.
12. **`.gitignore` obligatorio** al crear un repositorio, protegiendo binarios, IDE, pruebas, secretos y artefactos locales sin ocultar fuentes necesarias.
13. **Creación desde cero admitida:** un repositorio vacío se puede inicializar con su solución, bibliotecas, host, pruebas y documentación, después de resolver la rama `feature/*`.
14. **Auditoría independiente, si se solicita:** el auditor debe ser diferente del implementador, trabajar en solo lectura y producir hallazgos comprobables sin corregir código.
15. **Diseño orientado a servicios:** cada microservicio/worker debe tener una responsabilidad clara, configuración propia y límites operativos explícitos; las integraciones entre servicios no deben saltarse los contratos.

**Precedencia:** este documento fija el estándar transversal. Los contratos funcionales y decisiones explícitas del proyecto precisan su implementación sin contradecir las reglas. Si hay conflicto, explicar la incompatibilidad y pedir resolución antes de realizar el cambio afectado. Actualizar el marco requiere cambio de versión y constancia de autorización.

## 2. Git, permisos y trabajo existente

### 2.1. Verificación previa

Antes de escribir archivos:

1. Identificar ruta de repositorio, solución y rama (`git branch --show-current`).
2. Consultar `git status --short`; preservar cualquier modificación previa, incluso cambios no confirmados.
3. Exigir una rama cuyo nombre empiece exactamente por `feature/`.
4. Si no existe una rama válida, solicitar una o permiso explícito para crear/cambiar a ella. **No modificar el repositorio** hasta resolverlo.
5. Si no existe repositorio, confirmar la ruta y la rama autorizada; inicializar directamente mediante `git init -b feature/<descripcion>` **antes** de generar archivos de aplicación.
6. No hacer reset destructivo, clean, descartes ni sobrescritura de trabajo existente sin autorización específica.

**Convención recomendada:** `feature/<descripcion-en-kebab-case>`, opcionalmente con un identificador previo; por ejemplo, `feature/procesar-eventos`.

### 2.2. Operaciones permitidas

| Operación | Regla |
| --- | --- |
| `status`, `diff`, inspección de archivos y rama | Permitida en solo lectura. |
| Crear o cambiar de rama | Solo con instrucción o autorización expresa. |
| Editar código, pruebas, configuración o documentación | Solo dentro de `feature/*`. |
| `git add` | Evitar por defecto; solicitar cuando implique preparar un commit. |
| `commit`, `push`, `merge`, `rebase`, `tag`, PR | Solo por solicitud expresa y alcance definido. |
| `reset --hard`, `clean`, borrados destructivos | Prohibidos salvo autorización específica, tras revisar el impacto. |

La ausencia de una rama válida también impide editar `AGENTS.md`: la anotación quedará pendiente y se añadirá cuando la rama sea autorizada.

## 3. Nomenclatura y tipos de hosts

### 3.1. Nombre de solución

Para proyectos nuevos, utilizar `{CodigoProyecto}.{NombreProyecto}.slnx` en SDK .NET 10+; `.sln` está permitido por compatibilidad o necesidad explícita documentada. No generar ambos formatos salvo petición.

| Tipo de solución | Código | Ejemplo | Host principal |
| --- | --- | --- | --- |
| Microservicio con API REST/HTTP | `MS` | `MS.Orders.slnx` | `MS.Orders.Api` |
| Microservicio de procesamiento sin HTTP | `MS` | `MS.Payments.slnx` | `MS.Payments.Worker` |
| Microservicio con HTTP y proceso autónomo | `MS` | `MS.Notifications.slnx` | `MS.Notifications.Api` y `MS.Notifications.Worker` |
| Worker Service autónomo | `WS` | `WS.SyncOrders.slnx` | `WS.SyncOrders.Worker` |

**Criterios de decisión:**

- Elegir **`MS`** cuando se construye una capacidad de negocio independiente dentro de una arquitectura distribuida.
- Elegir **`WS`** cuando el producto principal es un procesamiento de segundo plano autónomo.
- Incluir `Api` solo si realmente existe un contrato HTTP de negocio o de integración que lo requiera.
- Incluir `Worker` si debe consumir mensajes, ejecutar jobs, sincronizar o procesar de forma continua/periódica.
- Un `MS` puede usar un **`BackgroundService` dentro de `Api`** para procesos ligeros que comparten ciclo de vida y escalado. Cuando se exijan escalado, disponibilidad, consumo o despliegue independientes, crear un **host `MS.<Nombre>.Worker` separado**.
- No convertir automáticamente cada tarea periódica en un servicio independiente, ni agrupar workloads incompatibles por conveniencia.

### 3.2. Nombres de proyectos

```text
{MS|WS}.{NombreProyecto}.Domain
{MS|WS}.{NombreProyecto}.Core
{MS|WS}.{NombreProyecto}.Shared
{MS|WS}.{NombreProyecto}.Infrastructure
{MS|WS}.{NombreProyecto}.Api       # Solo MS con API HTTP
{MS|WS}.{NombreProyecto}.Worker    # MS con worker o WS

tests/{MS|WS}.{NombreProyecto}.Core.Tests
tests/{MS|WS}.{NombreProyecto}.Infrastructure.Tests
```

No existe el sufijo `.Presentation`, `.Config`, `.WPF`, `.WFM`, `.Maui` ni un host frontend de ningún tipo en este estándar.

## 4. Arquitectura hexagonal

### 4.1. Estructura de referencia de un microservicio API + Worker

```text
MS.Notifications/
├── MS.Notifications.slnx
├── FRAMEWORK_DESARROLLO_CSHARP.md
├── AGENTS.md
├── .gitignore
├── global.json                           # Opcional: SDK fijado
├── Directory.Build.props                 # Opcional: convenciones compartidas
├── MS.Notifications.Domain/
│   ├── MS.Notifications.Domain.csproj
│   ├── Entities/
│   │   └── Notification.cs
│   ├── ValueObjects/
│   └── Interfaces/
│       ├── Ports/
│       ├── Services/
│       ├── UseCases/Notifications/
│       │   └── ICreateNotificationUseCase.cs
│       ├── Repositories/
│       │   └── INotificationRepository.cs
│       └── Persistence/
├── MS.Notifications.Core/
│   ├── MS.Notifications.Core.csproj
│   ├── Services/
│   ├── UseCases/Notifications/
│   │   └── CreateNotificationUseCase.cs
│   ├── Dtos/
│   └── Validators/
├── MS.Notifications.Shared/
│   ├── MS.Notifications.Shared.csproj
│   ├── Helpers/
│   ├── Extensions/
│   └── Constants/
├── MS.Notifications.Infrastructure/
│   ├── MS.Notifications.Infrastructure.csproj
│   ├── Persistence/Notifications/
│   │   ├── NotificationsDbContext.cs
│   │   └── Repositories/NotificationRepository.cs
│   ├── Messaging/
│   ├── Integrations/
│   └── DependencyInjection/
├── MS.Notifications.Api/                # Solo cuando el microservicio expone HTTP
│   ├── MS.Notifications.Api.csproj
│   ├── Program.cs
│   ├── Endpoints/
│   ├── Controllers/
│   ├── Middleware/
│   └── DependencyInjection/
├── MS.Notifications.Worker/             # Solo si debe ejecutarse por separado
│   ├── MS.Notifications.Worker.csproj
│   ├── Program.cs
│   ├── HostedServices/
│   ├── Jobs/
│   └── Consumers/
├── tests/
│   ├── MS.Notifications.Core.Tests/
│   └── MS.Notifications.Infrastructure.Tests/
└── docs/
    ├── context/INDEX.md
    ├── contracts/
    └── decisions/
```

**El árbol es ilustrativo:** solo se crean carpetas y clases cuando existe una funcionalidad real que lo requiera. Si el proyecto necesita únicamente API, no se crea `Worker`; si necesita solo Worker, no se crea `Api`. No se generan entidades, colas ni un DbContext ficticio para completar el ejemplo.

### 4.2. Estructura mínima de un Worker Service autónomo

```text
WS.SyncOrders/
├── WS.SyncOrders.slnx
├── FRAMEWORK_DESARROLLO_CSHARP.md
├── AGENTS.md
├── .gitignore
├── WS.SyncOrders.Domain/
├── WS.SyncOrders.Core/
├── WS.SyncOrders.Shared/
├── WS.SyncOrders.Infrastructure/
├── WS.SyncOrders.Worker/
│   ├── Program.cs
│   ├── HostedServices/
│   ├── Jobs/
│   └── Consumers/
├── tests/
│   ├── WS.SyncOrders.Core.Tests/
│   └── WS.SyncOrders.Infrastructure.Tests/
└── docs/context/INDEX.md
```

Todas las carpetas de las capas y el host están al mismo nivel que `.slnx`; **`src/` está prohibida**. Cada carpeta de proyecto contiene su `.csproj` con el mismo nombre.

### 4.3. Responsabilidades por capa

| Capa | Incluye | Prohibido |
| --- | --- | --- |
| **Domain** | Entidades, value objects, invariantes, modelos de dominio, contratos de servicios/casos de uso/repositorios/puertos/persistencia. | HTTP, clientes concretos, SQL, EF Core, infraestructura, hosting, UI. |
| **Core** | Casos de uso, servicios de aplicación, validadores, orquestación, DTOs de aplicación cuando correspondan. | Controllers, endpoints, consumers de infraestructura, `DbContext`, acceso SQL, implementaciones de proveedores. |
| **Shared** | Utilidades puras, helpers, constantes y extensiones verdaderamente transversales. | Lógica de negocio del servicio, entidades específicas, adaptadores o dependencias globales con estado. |
| **Infrastructure** | Persistencia, repositorios concretos, proveedores, mensajería, clientes externos y adaptadores que implementan puertos. | Reglas de negocio, decisiones de aplicación e implementación de casos de uso. |
| **Api** | HTTP, minimal APIs, controllers, validación del transporte, autenticación del borde, mapeos, middlewares, arranque y DI. | Negocio, SQL directo desde endpoints, acceso directo a repositorios saltando Core. |
| **Worker** | Arranque, `BackgroundService`, `IHostedService`, triggers, consumidores y jobs que delegan en Core. | Negocio, lógica de integración acoplada al host, repositorios llamados para saltarse casos de uso. |
| **Tests** | Pruebas, fixtures, dobles e integraciones de prueba aisladas. | Funcionalidad de producción necesaria para que el host funcione. |

### 4.4. Dependencias permitidas

```text
Domain          -> bibliotecas base de .NET
Shared          -> bibliotecas base / utilidades genéricas
Core            -> Domain, Shared
Infrastructure  -> Domain, Core, Shared (solo lo necesario)
Api             -> Core, Shared; Infrastructure solo para composición DI
Worker          -> Core, Shared; Infrastructure solo para composición DI
Core.Tests      -> Core, Domain, Shared, dobles de prueba
Infrastructure.Tests -> Infrastructure, Core, Domain, Shared, fixtures
```

La dirección de las dependencias de negocio apunta **hacia el núcleo**. `Domain` define las interfaces de los puertos; `Core` implementa los casos de uso y servicios; `Infrastructure` implementa los adaptadores técnicos. Los hosts se limitan a recibir solicitudes/eventos y ejecutar casos de uso.

Nunca permitir `Domain -> Core`, `Domain -> Infrastructure`, `Core -> Infrastructure`, `Core -> Api`, `Core -> Worker`, ni referencias circulares. El host puede referenciar `Infrastructure` para registrar DI, **no** para evadir el núcleo.

### 4.5. Organización interna obligatoria

Usar **PascalCase en carpetas** y espacios de nombres alineados con rutas. Ejemplos:

```text
Domain/Entities/<Entidad>.cs
Domain/ValueObjects/<ValueObject>.cs
Domain/Models/<Modelo>.cs
Domain/Interfaces/Ports/I<Integracion>.cs
Domain/Interfaces/Services/I<Servicio>Service.cs
Domain/Interfaces/UseCases/<Modulo>/I<Caso>UseCase.cs
Domain/Interfaces/Repositories/I<Entidad>Repository.cs
Domain/Interfaces/Persistence/I<Dao>.cs
Core/Services/<Servicio>Service.cs
Core/UseCases/<Modulo>/<Caso>UseCase.cs
Core/Dtos/<Modulo>/<Dto>.cs
Core/Validators/<Modulo>/<Validador>.cs
Infrastructure/Persistence/<Contexto>/<DbContext>.cs
Infrastructure/Persistence/<Contexto>/Repositories/<Repositorio>.cs
Infrastructure/Persistence/<Contexto>/Dao/<Dao>.cs
Infrastructure/Ports/<Adaptador>.cs
Infrastructure/Messaging/<Proveedor>/<Adaptador>.cs
Infrastructure/Integrations/<Proveedor>/<Cliente>.cs
Infrastructure/DependencyInjection/<Modulo>Registration.cs
Api/Endpoints/<Modulo>/<Endpoint>.cs
Api/Controllers/<Modulo>Controller.cs
Api/Middleware/<Middleware>.cs
Worker/HostedServices/<Nombre>HostedService.cs
Worker/Jobs/<Nombre>Job.cs
Worker/Consumers/<Nombre>Consumer.cs
Shared/Helpers/<Helper>.cs
Shared/Extensions/<Extension>.cs
tests/<Proyecto>.Core.Tests/UseCases/<Modulo>/<Caso>Tests.cs
tests/<Proyecto>.Infrastructure.Tests/Persistence/<Contexto>/<Adapter>Tests.cs
```

Son **rutas lógicas dentro de cada proyecto**, no carpetas literales `Domain/` dentro de otra carpeta `Domain`, y no obligan a crear rutas vacías. Regla general: **un tipo principal y una responsabilidad por archivo**. `Program.cs`, `.csproj` y archivos de arranque/configuración requeridos por plantillas pueden permanecer en la raíz del proyecto; las clases de aplicación no.

### 4.6. Reglas de diseño

- Un caso de uso consume puertos y tipos independientes; no esconde entidades ni interfaces en clases anidadas de servicio.
- Los value objects protegen sus invariantes; las entidades no invocan adaptadores externos.
- Los adaptadores implementan contratos sin decidir reglas del negocio.
- API y Worker **delegan en Core**; nunca comparten negocio copiando código entre hosts.
- `Shared` se crea como biblioteca del servicio, **no como excusa para acoplar múltiples microservicios** mediante entidades y reglas compartidas.
- Para comunicar servicios, preferir contratos públicos explícitos y versionados (HTTP/mensajería), evitando consultas directas a la base de datos de otro servicio.
- Los cambios entre servicios requieren contratos compatiblemente evolucionables; ninguna dependencia de red debe suponerse infalible.

## 5. Directrices de microservicios y Worker Services

### 5.1. Límites, datos e integraciones

1. Definir qué responsabilidad de negocio tiene el servicio y qué queda fuera.
2. Cada microservicio controla su modelo de datos y sus operaciones de persistencia. Evitar que otros servicios lean o escriban directamente sus tablas; si se necesita integración, usar contratos definidos.
3. Seleccionar HTTP/REST, eventos o colas **solo según necesidad del proyecto**. No instalar un broker por defecto.
4. Diseñar entradas para tolerar reintentos cuando corresponda. Los consumidores deben considerar **idempotencia**, duplicados y procesamiento al menos una vez.
5. Definir respuestas a errores transitorios, timeouts, cancelaciones, fallos de dependencias y mensajes malformados sin ocultar errores.
6. Usar `CancellationToken` y apagado ordenado para tareas que puedan interrumpirse.
7. Si hay transacciones, mensajes o eventos con efectos durables, registrar expresamente cómo se preserva la consistencia (por ejemplo, outbox si aplica); no introducir esos patrones sin necesidad.
8. Declarar la zona horaria/uso de UTC y el contrato de fechas para cada integración que dependa de tiempo.

### 5.2. Reglas para workers

- Implementar procesos de larga duración como `BackgroundService`/`IHostedService` si corresponde, conservando el procesamiento real en casos de uso de `Core`.
- Respetar el scope de DI: no inyectar directamente dependencias `Scoped` en singletons sin administrar scopes correctamente.
- En colas, decidir **ack/nack**, reintentos, backoff y destino de errores (`dead-letter` cuando exista soporte), documentando el comportamiento real.
- En jobs periódicos, evitar solapamientos no deseados; considerar concurrencia, intervalos, cron, reinicios y reejecuciones.
- Registrar inicio, ejecución, errores, finalización y apagado con logs estructurados; no registrar secretos o payloads sensibles sin justificación.
- Si API y Worker deben escalarse de forma independiente, usar **hosts separados**, aun dentro del mismo microservicio lógico.
- No asumir que un Worker necesita exposición HTTP, ni introducir una UI de diagnóstico.

### 5.3. Operación, seguridad y observabilidad

- **Configuración externa:** `appsettings` para opciones no sensibles; variables de entorno o un gestor de secretos para credenciales.
- **Seguridad:** autenticación/autorización en endpoints cuando aplique, validación de entradas, mínimos privilegios, gestión segura de certificados y ausencia de tokens en logs.
- **Observabilidad:** logging estructurado y correlación de solicitudes/eventos; métricas, trazas y health checks cuando el despliegue lo requiera.
- **Resiliencia:** políticas de reintento acotadas, timeouts y circuit breakers solo donde tengan sentido; evitar tormentas de reintentos y operaciones duplicadas.
- **Despliegue:** Docker, contenedores, Kubernetes, CI/CD, readiness y liveness se incorporan cuando el contrato o el entorno los requieran. No son proyectos/capas obligatorios del esqueleto.
- **Compatibilidad:** contratos HTTP y eventos versionados de forma adecuada; migraciones con estrategia para cambios no compatibles cuando existan consumidores.
- **No se añade infraestructura de forma preventiva**. Todo proveedor, base de datos, cola o servicio nuevo debe tener una necesidad funcional u operativa comprobable.

## 6. Plataforma .NET y convenciones técnicas

- Para soluciones nuevas usar la **última LTS de .NET soportada** que sea compatible con el entorno y dependencias.
- **Referencia a octubre de 2026:** .NET 10 LTS y C# 14 estable. Verificar vigencia cuando se inicie cada proyecto; no usar `preview` por defecto.
- Usar `net10.0` como TFM de referencia para ejemplos actuales, sujeto a compatibilidad real.
- Documentar/fijar SDK con `global.json` cuando sea relevante para reproducibilidad.
- Mantener versiones de paquetes y dependencias controladas; no introducir nuevas librerías sin necesidad explícita.
- Favorecer inyección de dependencias del host .NET, opciones tipadas y logging estándar cuando satisfagan la necesidad; no imponer frameworks extra.
- No actualizar SDK, paquetes mayores ni arquitectura por iniciativa propia cuando el prompt trate otro cambio.

Referencias técnicas: [política de soporte .NET](https://learn.microsoft.com/dotnet/core/releases-and-support) y [versiones de C#](https://learn.microsoft.com/dotnet/csharp/language-reference/language-versioning).

## 7. Creación desde cero

### 7.1. Información mínima necesaria

Antes de crear archivos, identificar:

1. **Nombre del proyecto** y tipo `MS` o `WS`.
2. **Destino** y estado del directorio/repo existente.
3. **Rama `feature/*` autorizada**; sin ella no escribir archivos del repo.
4. **Host o hosts reales:** `Api`, `Worker` o ambos si es `MS`; `Worker` si es `WS`.
5. **Objetivo inicial** y restricciones reales. Si todavía no se ha definido funcionalidad, solo crear estructura arquitectónica y pruebas sin código ficticio de negocio.
6. **Necesidades técnicas comprobadas:** datos, mensajes, integración, seguridad o despliegue, cuando hayan sido definidas.

**No preguntar ni ofrecer opciones de frontend**, UI de configuración, identidad gráfica o tecnologías de presentación.

### 7.2. Secuencia obligatoria

1. Comprobar SDK (`dotnet --info`), TFM, plantilla de host y compatibilidad.
2. Verificar directorio, repositorio, rama y cambios previos. Si se autorizó inicializar Git, hacerlo directamente en `feature/*`.
3. Crear o revisar `.gitignore` raíz (sin sobrescribir uno existente).
4. Generar `{MS|WS}.{NombreProyecto}.slnx` en la raíz (o `.sln` justificado).
5. Crear `Domain`, `Core`, `Shared` e `Infrastructure` como cuatro bibliotecas independientes, directamente en la raíz.
6. Crear exclusivamente los hosts necesarios: `Api`, `Worker` o ambos; para `WS` crear `Worker`.
7. Crear y agregar `Core.Tests` e `Infrastructure.Tests` bajo `tests/`.
8. Añadir todos los `.csproj` a la solución y configurar referencias respetando §4.4.
9. Distribuir código real por carpetas semánticas según §4.5; quitar clases vacías generadas sin valor contractual.
10. Crear `AGENTS.md`, `docs/context/INDEX.md` y copia/referencia vigente del framework; registrar decisiones, pendientes y alcance inicial.
11. Implementar solo la funcionalidad autorizada, con pruebas relevantes.
12. Ejecutar `dotnet restore`, `dotnet build`, `dotnet test`, listar solución y verificar `.gitignore`/estado Git. Registrar resultados **reales** sin inventarlos.
13. Entregar resumen y limitaciones; **sin hacer commit ni push** por defecto.

### 7.3. Ejemplo operativo — microservicio HTTP

**Ejemplo únicamente; no ejecutar la rama sin autorización expresa.** Desde la raíz de un directorio vacío verificado, con SDK .NET 10:

```bash
# Requiere autorización previa del usuario
# git init -b feature/inicializacion-orders

dotnet new gitignore
dotnet new sln -n MS.Orders

dotnet new classlib -n MS.Orders.Domain -o MS.Orders.Domain -f net10.0
dotnet new classlib -n MS.Orders.Core -o MS.Orders.Core -f net10.0
dotnet new classlib -n MS.Orders.Shared -o MS.Orders.Shared -f net10.0
dotnet new classlib -n MS.Orders.Infrastructure -o MS.Orders.Infrastructure -f net10.0
dotnet new webapi -n MS.Orders.Api -o MS.Orders.Api -f net10.0

dotnet new xunit -n MS.Orders.Core.Tests -o tests/MS.Orders.Core.Tests -f net10.0
dotnet new xunit -n MS.Orders.Infrastructure.Tests -o tests/MS.Orders.Infrastructure.Tests -f net10.0

dotnet sln MS.Orders.slnx add \
  MS.Orders.Domain/MS.Orders.Domain.csproj \
  MS.Orders.Core/MS.Orders.Core.csproj \
  MS.Orders.Shared/MS.Orders.Shared.csproj \
  MS.Orders.Infrastructure/MS.Orders.Infrastructure.csproj \
  MS.Orders.Api/MS.Orders.Api.csproj \
  tests/MS.Orders.Core.Tests/MS.Orders.Core.Tests.csproj \
  tests/MS.Orders.Infrastructure.Tests/MS.Orders.Infrastructure.Tests.csproj

# Referencias mínimas orientativas; ampliar solo si las usa el proyecto.
dotnet add MS.Orders.Core/MS.Orders.Core.csproj reference MS.Orders.Domain/MS.Orders.Domain.csproj MS.Orders.Shared/MS.Orders.Shared.csproj
dotnet add MS.Orders.Infrastructure/MS.Orders.Infrastructure.csproj reference MS.Orders.Core/MS.Orders.Core.csproj MS.Orders.Domain/MS.Orders.Domain.csproj
dotnet add MS.Orders.Api/MS.Orders.Api.csproj reference MS.Orders.Core/MS.Orders.Core.csproj MS.Orders.Infrastructure/MS.Orders.Infrastructure.csproj
dotnet add tests/MS.Orders.Core.Tests/MS.Orders.Core.Tests.csproj reference MS.Orders.Core/MS.Orders.Core.csproj
dotnet add tests/MS.Orders.Infrastructure.Tests/MS.Orders.Infrastructure.Tests.csproj reference MS.Orders.Infrastructure/MS.Orders.Infrastructure.csproj

# Tras incorporar AGENTS.md, docs y el código real autorizado:
dotnet sln MS.Orders.slnx list
dotnet restore MS.Orders.slnx
dotnet build MS.Orders.slnx --no-restore
dotnet test MS.Orders.slnx --no-build
git status --short --ignored
```

### 7.4. Ejemplo operativo — Worker Service

Para `WS.SyncOrders`, usar `WS.SyncOrders.slnx`, las cuatro bibliotecas `WS.SyncOrders.{Domain,Core,Shared,Infrastructure}` y el host:

```bash
dotnet new worker -n WS.SyncOrders.Worker -o WS.SyncOrders.Worker -f net10.0
```

Las pruebas se denominan `WS.SyncOrders.Core.Tests` y `WS.SyncOrders.Infrastructure.Tests`. Las referencias siguen exactamente la misma dirección del §4.4. Si se requiere un Worker **dentro de un MS**, usar `MS.<Nombre>.Worker`; si es ejecutable separado, agregarlo a la solución, su propio `Program.cs` y su composición DI.

**Precaución:** los comandos anteriores solo demuestran estructura; no sustituyen la verificación previa de Git, la documentación y la creación/registro de pruebas. En repositorios existentes, no reinicializar Git ni recrear proyectos sin orden explícita.

### 7.5. `.gitignore` obligatorio

Un único `.gitignore` raíz protege, según proceda:

| Grupo | Patrones o archivos representativos |
| --- | --- |
| Compilación .NET | `**/bin/`, `**/obj/`, salidas temporales de publicación |
| IDE y usuario | `.vs/`, `.idea/`, `*.user`, `*.suo`, estados locales de editor |
| Pruebas | `TestResults/`, `*.trx`, `coverage/`, `*.coverage*` |
| Archivos temporales | `*.log`, dumps, archivos intermedios locales |
| Credenciales | `.env`, `.env.*` (salvo ejemplos públicos), secretos, claves privadas, certificados de uso privado |
| Operación local | volúmenes de desarrollo, estados locales de workers y colas si están definidos |

No ignorar archivos `.cs`, `.csproj`, `.slnx`, `.sln`, pruebas, migraciones, `AGENTS.md`, documentación o configuraciones **seguras** necesarias. Validar patrones cuando proceda con `git check-ignore -v` y `git status --short --ignored`; no eliminar archivos versionados por conveniencia.

## 8. Contratos e implementación

1. Leer `AGENTS.md`, `docs/context/INDEX.md`, contratos, decisiones cerradas y código relacionado antes de implementar.
2. Delimitar entradas, salidas, restricciones, fallos, necesidades operativas y criterios de aceptación.
3. Definir/reutilizar puertos e interfaces en `Domain/Interfaces` antes de crear implementaciones.
4. Implementar el cambio mínimo con ubicación semántica, namespaces coherentes y sin agregar capas ni hosts innecesarios.
5. Validar entradas del transporte y preservar invariantes en el dominio.
6. Propagar cancelaciones y errores correctamente; evitar retornos de éxito falsos.
7. Usar configuración externa para secretos, telemetría segura y manejo explícito de fallos en integraciones.
8. Para workers, documentar reintentos, duplicados, concurrencia y apagado si afectan el comportamiento.
9. Actualizar pruebas y realizar verificaciones factibles antes de declarar implementado.
10. Registrar resultados, archivos, pendientes, limitaciones y autorización Git en `AGENTS.md`.

## 9. Pruebas y validación

Cada solución incluye al menos:

- **`tests/{MS|WS}.{Nombre}.Core.Tests`:** casos de uso, servicios de aplicación, validadores, éxito/error y condiciones de borde, con dobles para puertos.
- **`tests/{MS|WS}.{Nombre}.Infrastructure.Tests`:** adaptadores, persistencia, serialización, proveedores, mensajería y errores técnicos; cuando se conecte a sistemas reales, clasificar las pruebas como integración.

**Reglas:**

1. Las pruebas deben verificar comportamientos útiles, no solo instancias ni aserciones triviales.
2. Las pruebas unitarias han de ser deterministas y aisladas de servicios reales.
3. Cubrir los casos positivos y los principales casos de error/reintento cuando estén dentro del contrato.
4. Cuando corresponda, comprobar cancelación, duplicados, idempotencia, carreras, fallos transitorios y recuperación en workers.
5. No declarar compilación/pruebas exitosas sin ejecutarlas; reportar comandos no ejecutados y motivo.
6. xUnit es el framework recomendado para soluciones nuevas; respetar una elección existente válida.
7. Pruebas adicionales de hosts, contratos HTTP o integraciones distribuidas se incorporan cuando su riesgo o contrato lo justifiquen; no son un requisito de proyecto vacío.

```bash
dotnet restore <Solucion>.slnx
dotnet build <Solucion>.slnx --no-restore
dotnet test <Solucion>.slnx --no-build
```

Sustituir `.slnx` por `.sln` si el proyecto real así lo requiere.

## 10. Implementador y auditoría independiente

| Rol | Puede | No puede |
| --- | --- | --- |
| **Implementador** | Inspeccionar, implementar en `feature/*`, crear pruebas, registrar cambios y ejecutar verificaciones. | Confirmar/pushear cambios sin permiso; presentarse como auditor externo. |
| **Auditor independiente** | Inspeccionar repositorio, contratos, diff y pruebas; ejecutar verificaciones de solo lectura y emitir informe. | Modificar código, documentación, pruebas o configuración; arreglar hallazgos. |
| **Responsable de decisión** | Aprobar cambios contractuales, excepciones e instrucciones Git. | Convertir comprobaciones pendientes en evidencia validada. |

Solo se declara «auditoría independiente» si la realizó un **agente distinto** del implementador. Si no existe segundo agente, reportar **«Auditoría independiente no realizada»**; la autoverificación técnica continúa siendo obligatoria.

### 10.1. Escala de gravedad

- **Bloqueante:** incumplimiento directo del contrato que impide operar o permite lo prohibido; no integrar hasta corregir.
- **Importante:** defecto funcional, contractual, de seguridad o de evidencia que requiere tratamiento explícito.
- **Menor:** convención, nombres o forma sin impacto demostrado superior.

No elevar preferencias de estilo a bloqueantes sin evidencia.

### 10.2. Formato del informe

```markdown
# Informe de auditoría
- Solución / rama / alcance / contrato:
- Implementador:
- Auditor (distinto):
- Contexto revisado:
- Verificaciones ejecutadas y resultados:

## Hallazgos
### H-001 · [Bloqueante | Importante | Menor]
- Archivo y línea:
- Regla incumplida:
- Evidencia observada:
- Corrección acotada propuesta:
- Procedimiento de verificación:

## Cierre
- Resultado: [Sin hallazgos | Con hallazgos]
- Limitaciones:
- El auditor no modificó archivos.
```

«Sin hallazgos» es válido cuando está sustentado. La corrección, si se aprueba, vuelve al implementador y se somete a nuevas verificaciones.

## 11. `AGENTS.md`: historial obligatorio de prompts

Cada repositorio incluye `AGENTS.md` en su raíz como guía operativa y bitácora. Por cada solicitud, incluso cuando no haya cambios, registrar:

- Fecha real e identificador de solicitud.
- Resumen fiel del requerimiento y alcance autorizado.
- Rama activa y estado (implementado, pendiente, bloqueado, análisis, cancelado).
- Archivos creados o modificados y decisiones explícitas.
- Pruebas/comandos ejecutados y resultados reales.
- Riesgos, limitaciones, trabajo pendiente y auditoría independiente si realmente ocurrió.
- **`Commit autorizado: NO`**, salvo autorización expresa.

Conservar orden cronológico y las entradas anteriores; no registrar secretos, credenciales, tokens ni prompts extensos con información sensible. Si la rama no es válida, no escribir `AGENTS.md`; documentar la anotación pendiente hasta estar autorizado.

## 12. Contexto y decisiones cerradas

`docs/context/INDEX.md` debe enlazar documentos **existentes**, como:

- Objetivo, alcance y fronteras del microservicio/worker.
- Contratos funcionales y criterios de aceptación.
- Arquitectura y puertos principales.
- Eventos, APIs e integraciones vigentes si los hay.
- Decisiones cerradas y restricciones técnicas.
- Estrategia de pruebas, build y despliegue relevante.

Si un documento aún no existe, marcarlo «pendiente». No inventar decisiones cerradas, contratos ni evidencia de lectura.

## 13. Secuencia obligatoria por solicitud

```text
1. RECIBIR PROMPT
   ↓
2. IDENTIFICAR SI SE CREA UN REPOSITORIO O SE MODIFICA UNO EXISTENTE
   ↓
3. LEER CONTEXTO REAL; IDENTIFICAR MS O WS Y HOSTS NECESARIOS
   ↓
4. VALIDAR RAMA feature/*, DESTINO Y TRABAJO PREEXISTENTE
   ├── Sin Git y autorización explícita → inicializar en feature/*
   ├── Sin rama/autorización → detener escritura, pedir decisión
   └── Con feature/* → continuar
   ↓
5. DELIMITAR ALCANCE Y CONTRATOS, SIN INVENTAR COMPONENTES
   ↓
6. SI ES NUEVO: .gitignore + SOLUCIÓN + CAPAS + HOST(S) + TESTS + AGENTS.md
   SI EXISTE: PRESERVAR ESTRUCTURA Y CAMBIAR SOLO LO AUTORIZADO
   ↓
7. CREAR INTERFACES/PUERTOS Y UBICAR CLASES EN CARPETAS SEMÁNTICAS
   ↓
8. IMPLEMENTAR CORE, ADAPTADORES Y HOST API/WORKER SEGÚN CONTRATO
   ↓
9. ACTUALIZAR PRUEBAS, COMPILAR Y VERIFICAR DIFF
   ↓
10. [SI SE SOLICITA] AUDITORÍA POR AGENTE DISTINTO, SOLO LECTURA
   ↓
11. REGISTRAR RESULTADOS REALES EN AGENTS.md
   ↓
12. ENTREGAR RESUMEN, EVIDENCIA Y LIMITACIONES
   ↓
13. NO COMMIT / PUSH / MERGE SIN ORDEN EXPRESA
```

## 14. Lista de control de cierre

Marcar cada punto **Sí / No / No aplica / No verificado**, con evidencia relevante:

- [ ] Se determinó `MS` o `WS` y los hosts **reales** (`Api`, `Worker` o ambos) del backend.
- [ ] No existen proyectos frontend, `Presentation`, `Config` gráfico ni archivos de estilo/UI.
- [ ] No se incorporaron marcas, convenciones o referencias corporativas.
- [ ] Se verificó la rama `feature/*` antes de editar.
- [ ] Se preservaron cambios previos del usuario.
- [ ] El archivo `.slnx`/`.sln` y las capas están directamente en raíz, sin `src/`.
- [ ] Domain, Core, Shared, Infrastructure y hosts son `.csproj` independientes.
- [ ] Interfaces de Domain y sus implementaciones respetan responsabilidades y dirección de referencias.
- [ ] No hay lógica de negocio en controllers, endpoints, consumers, hosted services ni adaptadores.
- [ ] Código organizado en carpetas semánticas, sin clases de aplicación sueltas.
- [ ] `.gitignore` protege artefactos/secretos sin ocultar archivos necesarios.
- [ ] Contratos entre microservicios, colas, bases de datos y jobs reflejan necesidades reales.
- [ ] Reintentos, duplicados, idempotencia, cancelación y concurrencia fueron considerados cuando aplica.
- [ ] Existen proyectos `Core.Tests` e `Infrastructure.Tests` con pruebas pertinentes.
- [ ] Se ejecutaron build/pruebas aplicables y se reportaron sus resultados efectivos.
- [ ] Se actualizó `AGENTS.md` o se indicó el bloqueo por rama.
- [ ] Si hubo auditoría, fue realizada por un agente distinto sin editar archivos.
- [ ] No se ejecutaron operaciones Git restringidas sin autorización.

## 15. Instrucción maestra para agentes de IA

> Trabaja exclusivamente con `FRAMEWORK_DESARROLLO_CSHARP.md` y `AGENTS.md` para **proyectos personales backend en .NET**. Este marco **solo admite microservicios (`MS`) y Worker Services (`WS`)**. Nunca crees ni propongas frontend, WPF, WinForms, MAUI, presentación gráfica o archivos de estilo, y no utilices branding ni patrones de AlphaTech. Identifica si el requerimiento crea un proyecto desde cero o modifica uno existente. Verifica la rama `feature/*` y preserva cambios previos; si la rama no cumple, solicita autorización antes de escribir. Para proyectos nuevos crea `.slnx` (o `.sln` justificado), `.gitignore`, Domain, Core, Shared, Infrastructure, los hosts `Api` y/o `Worker` realmente necesarios, Core.Tests, Infrastructure.Tests, `AGENTS.md` y `docs/context/INDEX.md`. Cada capa es un `.csproj` separado y se ubica directamente en la raíz sin carpeta `src/`. Clasifica cada clase en su subcarpeta específica; Domain declara contratos, Core implementa casos de uso e Infrastructure adaptadores; los hosts solo reciben solicitudes, eventos o disparadores y delegan en Core. Permite `BackgroundService` dentro del host o `MS.<Nombre>.Worker` autónomo cuando el contrato lo amerite. No inventes endpoints, colas, entidades o infraestructura. Aplica principios de resiliencia, cancelación, seguridad, idempotencia y observabilidad según los riesgos reales. Ejecuta y registra build/pruebas, actualiza `AGENTS.md` y entrega resultados verificables. Auditoría independiente solo si otro agente la realiza. Nunca hagas commit, push, merge, tag o PR sin solicitud expresa.

### 15.1. Ejemplo de prompt para microservicio

> **Modo:** crear desde cero. **Nombre:** `Orders`. **Tipo:** `MS`. **Hosts:** `Api` y `Worker` independiente, porque los trabajos se ejecutarán y escalarán por separado. **Rama autorizada:** `feature/inicializacion-orders`. Crea la solución `MS.Orders.slnx`, `Domain`, `Core`, `Shared`, `Infrastructure`, `MS.Orders.Api`, `MS.Orders.Worker`, proyectos `Core.Tests` e `Infrastructure.Tests`, `.gitignore`, `AGENTS.md` e índice de contexto. Todo al nivel de la solución excepto las pruebas en `tests/`; no crear `src/`. Diseña puertos en Domain y lógica en Core; deja los hosts solo como adaptadores de entrada y composición. No inventes funcionalidades no definidas, compila, prueba y no hagas commits.

### 15.2. Ejemplo de prompt para Worker Service

> **Modo:** crear desde cero. **Nombre:** `SyncOrders`. **Tipo:** `WS`. **Host:** `Worker` sin API. **Rama autorizada:** `feature/inicializacion-sync-orders`. Crea `WS.SyncOrders.slnx` con `Domain`, `Core`, `Shared`, `Infrastructure`, `WS.SyncOrders.Worker`, pruebas de Core e Infrastructure, `.gitignore`, `AGENTS.md` e índice de contexto. No agregues frontend, UI de configuración, API REST ni infraestructura ajena al contrato. Implementa el Worker con `BackgroundService` únicamente cuando se definan sus tareas; los flujos de negocio pertenecen a Core. Verifica la solución, registra resultados y no realices commits.

---

**Vigencia:** estándar personal v2.0. Sustituye las versiones orientadas a aplicaciones generales y cualquier convención de branding o presentación visual. Los proyectos individuales pueden añadir reglas técnicas más estrictas, pero no introducir frontend dentro de este marco ni relajar las obligaciones sin modificación explícita del estándar.
