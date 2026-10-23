# FRAMEWORK_DESARROLLO_FRONTEND

> **Versión:** 1.0  
> **Fecha:** 2026-10-21  
> **Clasificación:** estándar personal de desarrollo frontend  
> **Identificador obligatorio:** `MF.nombre-del-proyecto`  
> **Tecnología predeterminada:** Angular + TypeScript + SCSS  
> **Arquitectura:** feature-based, componentes standalone  
> **Producto final:** Progressive Web App (PWA)  
> **Diseño inicial:** tema claro, paleta sobria, tema oscuro seleccionable

## 1. Propósito, alcance y decisiones permanentes

Este documento define las reglas obligatorias para crear, ampliar, mantener y auditar aplicaciones frontend personales identificadas mediante el prefijo `MF`. Es un **marco distinto e independiente** de cualquier framework backend, microservicio o Worker Service.

### 1.1. Reglas no negociables

1. Toda nueva aplicación usa la identidad **`MF.<nombre-en-kebab-case>`** para el repositorio y directorio principal. Ejemplo: `MF.gestion-de-prestamos`.
2. Se utiliza **Angular y TypeScript** con arquitectura **feature-based**. No se estructura el producto completo por carpetas globales de componentes, servicios y modelos.
3. Cada funcionalidad reside en `src/app/features/<feature>/` y mantiene sus páginas, componentes, lógica, acceso a datos, modelos, estado y rutas **dentro de sus propios límites**.
4. `src/app/shared/` se reserva a piezas genéricas y reutilizables, sin lógica de negocio ni referencias a funcionalidades concretas.
5. `src/app/core/` se reserva a capacidades transversales de la aplicación: bootstrap, configuración, cliente HTTP, autenticación cuando exista, errores, telemetría y orquestación técnica de PWA.
6. `src/app/themes/` es la **única fuente autorizada** de tokens visuales globales, temas, escala tipográfica, espaciado, radios, sombras y variantes Light/Dark. Nada de colores arbitrarios repetidos en componentes.
7. La aplicación inicia **en modo claro la primera vez**, independientemente del tema del sistema operativo. Siempre permite cambiar manualmente entre `light` y `dark` y guarda la preferencia cuando el navegador lo permite.
8. Se utiliza una **paleta inicial sobria, neutra, discreta y accesible**. Los colores podrán modificarse durante el desarrollo editando los tokens de `themes`, sin reescribir componentes.
9. **Toda aplicación creada con este marco debe terminar siendo PWA instalable.** En el inicio se pregunta si habilitar el service worker inmediatamente o activarlo en una fase posterior; en ambos casos, PWA es obligatoria **antes de publicar la primera versión de producción**. No es opcional el resultado final.
10. Para proyectos nuevos se usa la **versión más reciente que figure oficialmente en fase LTS de Angular**, con el último parche estable de esa línea, junto con versiones compatibles de Node.js LTS, TypeScript y RxJS. La versión queda fijada; no se actualiza silenciosamente por cada ejecución.
11. No se crea backend dentro de la aplicación ni se incorporan servicios .NET: se consume su API a través de contratos y clientes HTTP.
12. La seguridad, accesibilidad, pruebas, responsive design, manejo de estados y rendimiento son requisitos de definición de terminado, no complementos posteriores.
13. No se añaden librerías de UI, gestores de estado globales, sistemas de traducción, herramientas de analítica, notificaciones push o estrategias offline de negocio por simple preferencia: requieren necesidad real.
14. Toda nueva funcionalidad o cambio de código se trabaja en una rama `feature/*`. Si la rama actual no cumple, no modificar el repositorio antes de obtener autorización para crear o cambiar de rama.
15. No ejecutar `commit`, `push`, `merge`, tags ni PR por iniciativa propia. Requieren petición expresa.
16. `AGENTS.md` y `docs/context/INDEX.md` son obligatorios y deben contener las restricciones del proyecto y el registro cronológico de intervenciones.
17. Se prohíbe incorporar identidades visuales, dominios, convenciones o referencias de organizaciones ajenas a estos proyectos personales.
18. Las instrucciones del proyecto particular pueden precisar este marco, pero no levantar restricciones obligatorias sin una decisión explícita y documentada.

### 1.2. Qué significa «siempre PWA»

- **Desde el día 1**: la arquitectura es compatible con web app instalable, diseño responsive, iconografía y metadatos previstos, URLs navegables, compatibilidad HTTPS, y límites claros de persistencia/caché.
- **Momento de activación consultado**: `al crear el proyecto` (recomendado) o `durante el desarrollo`.
- **Si se difiere**: registrar `PWA pendiente`, responsable y criterio de cierre; implementar el paso de §10 antes de producción.
- **Nunca prometer offline completo automáticamente**: la instalación de la PWA y el cacheo de recursos estáticos no significan que las operaciones de negocio funcionen sin red.

## 2. Nomenclatura obligatoria

### 2.1. Proyecto y repositorio

Patrón externo:

```text
MF.<nombre-del-proyecto>
```

Reglas para `<nombre-del-proyecto>`:

- Solo letras ASCII minúsculas (`a-z`), números (`0-9`) y guiones simples (`-`).
- Debe comenzar y terminar con letra o número; sin espacios, tildes, guiones repetidos o guiones bajos.
- Puede tener una o varias palabras: `MF.prestamos`, `MF.rutas-de-cobro`, `MF.control-de-gastos-personales`.
- `MF.` se escribe exactamente con mayúsculas en el **nombre visible del repositorio y directorio raíz**.
- Internamente, los identificadores sujetos a validaciones de herramientas (por ejemplo, `package.json` y el nombre del proyecto en Angular CLI) se mantienen en **minúsculas**: `rutas-de-cobro`. No forzar mayúsculas donde npm no las admite.
- El selector base de componentes puede usar `mf-` y los identificadores de rutas/archivos usar `kebab-case`.

Ejemplo:

```text
Repositorio/directorio:  MF.rutas-de-cobro
Angular CLI project:   rutas-de-cobro
package.json name:     rutas-de-cobro
selector:              mf-customer-list
feature:               customers
ruta:                  /customers
archivo:               customer-list.page.ts
clase:                 CustomerListPage
```

El uso del prefijo `MF.` es una convención de **identificación externa**, no una invitación a incumplir las reglas de nombres de npm.

### 2.2. Nombres de carpetas, tipos y archivos

| Elemento | Convención | Ejemplo |
| --- | --- | --- |
| Directorio de feature | `kebab-case` | `payment-history/` |
| Página de ruta | `*.page.ts` | `payment-history.page.ts` |
| Componente reutilizable | `*.component.ts` | `amount-input.component.ts` |
| Servicio | `*.service.ts` | `customer-api.service.ts` |
| Guard | `*.guard.ts` | `auth.guard.ts` |
| Interceptor | `*.interceptor.ts` | `auth.interceptor.ts` |
| Pipe | `*.pipe.ts` | `currency-format.pipe.ts` |
| Directiva | `*.directive.ts` | `autofocus.directive.ts` |
| Modelo | `*.model.ts` | `customer.model.ts` |
| Contrato de API | `*.dto.ts` | `customer-response.dto.ts` |
| Prueba unitaria | `*.spec.ts` | `customer-list.page.spec.ts` |
| Archivo de rutas | `*.routes.ts` | `customers.routes.ts` |
| Prueba E2E | `*.spec.ts`, en `e2e/` | `customers-flow.spec.ts` |
| Variables visuales | `*.tokens.scss` / `*.theme.scss` | `_colors.tokens.scss` |
| Eventos/estado | nombres concretos | `customer.store.ts` |

Preferir un concepto principal por archivo, rutas y nombres explícitos, y colocación de pruebas cerca del código que verifican.

## 3. Política de versiones LTS

1. Antes de iniciar cada proyecto, consultar las páginas **oficiales** de [Angular releases](https://angular.dev/reference/releases) y [Angular compatibility](https://angular.dev/reference/versions).
2. Elegir la **mayor versión cuyo estado sea expresamente `LTS`**. No confundir `Active` con `LTS` ni asumir que la mayor versión numérica es la requerida.
3. Instalar el parche estable más reciente disponible **dentro de esa mayor LTS**; registrar el número exacto en `AGENTS.md`.
4. Elegir una versión Node.js **LTS y soportada por esa versión de Angular**. Fijar la versión de Node mediante `.nvmrc`, `.node-version` o el mecanismo acordado.
5. Respetar la matriz oficial de compatibilidad de TypeScript y RxJS; dejar versiones resueltas en `package-lock.json`.
6. Establecer `package-lock.json` versionado y usar `npm ci` en CI. No cambiar gestores de paquetes sin solicitud expresa.
7. No utilizar versiones pre-release (`alpha`, `beta`, `rc`, `next`) ni `latest` flotante en automatizaciones reproducibles.
8. Para cada migración de Angular, verificar compatibilidad y ejecutar `ng update` en una rama feature específica, con pruebas y validaciones PWA/temas.

**Referencia verificada el 21/10/2026:** Angular **21** está en fase LTS y Angular **22** figura en soporte activo. Este dato es ilustrativo: **no fijar permanentemente Angular 21 en el marco**. La regla dinámica anterior prevalece.

## 4. Arquitectura feature-based

### 4.1. Responsabilidades de las áreas

| Área | Responsabilidad | Prohibiciones |
| --- | --- | --- |
| `features/` | Flujos de usuario, páginas, componentes específicos, estado y acceso a datos de una funcionalidad | Dependencias directas de detalles internos de otra feature |
| `shared/` | UI genérica, directivas, pipes, validadores genéricos y utilidades sin negocio | Clientes de API de negocio, stores globales, permisos de una feature |
| `core/` | Servicios raíz y capacidades técnicas transversales, HTTP, auth, errores, navegación técnica, PWA | Páginas de negocio, DTOs/estado propios de una feature |
| `themes/` | Tokens, reglas Light/Dark, tipografía, espaciado, estados y mixins visuales | Lógica comercial, peticiones HTTP, almacenamiento de datos de negocio |
| `layouts/` | Shells visuales de aplicación, navegación y estructura de pantalla | Reglas de dominio o acoplamientos a implementaciones de features |
| `environments/` | Valores públicos de configuración por compilación, si son necesarios | Claves privadas, tokens, secretos |

**Dirección de dependencias**:

```text
app bootstrap / routes / layouts
             |
             +--> features/<feature> --> core (contratos transversales)
             |                     \--> shared (componentes/utilidades)
             +--> core
             +--> shared
             +--> themes (estilos/tokens)

shared   -X-> features
core     -X-> features
featureA -X-> featureB/internals
business -X-> themes en TypeScript (la interacción visual usa tokens CSS)
```

Una feature puede consumir una interfaz estable expuesta intencionalmente por un contrato común, pero **no importar servicios, stores, rutas internas o componentes privados** de otra feature. Para comunicación entre features, usar navegación, contratos públicos bien definidos o coordinación en `core` cuando sea genuinamente transversal. Evitar crear un `shared` gigantesco para saltar los límites.

### 4.2. Estructura de referencia

```text
MF.rutas-de-cobro/
├── .git/
├── .gitignore
├── .nvmrc                              # LTS Node compatible
├── AGENTS.md
├── FRAMEWORK_DESARROLLO_FRONTEND.md
├── README.md
├── angular.json
├── package.json
├── package-lock.json
├── tsconfig.json
├── ngsw-config.json                    # Cuando se active el Service Worker PWA
├── public/
│   ├── favicon.ico
│   ├── manifest.webmanifest             # Cuando se active PWA
│   └── icons/                          # Íconos finales aprobados
├── src/
│   ├── main.ts
│   ├── index.html
│   ├── styles.scss                      # Solo imports y estilos globales necesarios
│   ├── app/
│   │   ├── app.ts                       # Componente raíz (nombre según CLI LTS)
│   │   ├── app.config.ts
│   │   ├── app.routes.ts
│   │   ├── core/
│   │   │   ├── config/
│   │   │   ├── http/
│   │   │   │   ├── api-client.service.ts
│   │   │   │   └── interceptors/
│   │   │   ├── auth/                   # Solo si hay autenticación
│   │   │   ├── error-handling/
│   │   │   └── pwa/                    # Servicio de instalación/actualización si aplica
│   │   ├── layouts/
│   │   │   └── main-layout/
│   │   │       ├── main-layout.component.ts
│   │   │       └── main-layout.component.html
│   │   ├── features/
│   │   │   ├── customers/
│   │   │   │   ├── customers.routes.ts
│   │   │   │   ├── pages/
│   │   │   │   │   └── customer-list/
│   │   │   │   │       ├── customer-list.page.ts
│   │   │   │   │       ├── customer-list.page.html
│   │   │   │   │       ├── customer-list.page.scss
│   │   │   │   │       └── customer-list.page.spec.ts
│   │   │   │   ├── components/
│   │   │   │   ├── data-access/
│   │   │   │   │   ├── customer-api.service.ts
│   │   │   │   │   └── customer-response.dto.ts
│   │   │   │   ├── models/
│   │   │   │   └── state/              # Solo si necesita estado coordinado
│   │   │   └── payments/              # Segunda feature, si existe
│   │   │       ├── payments.routes.ts
│   │   │       └── ...
│   │   ├── shared/
│   │   │   ├── ui/
│   │   │   │   ├── theme-switch/
│   │   │   │   ├── button/
│   │   │   │   └── empty-state/
│   │   │   ├── directives/
│   │   │   ├── pipes/
│   │   │   ├── validators/
│   │   │   └── utils/
│   │   └── themes/
│   │       ├── _colors.tokens.scss
│   │       ├── _typography.tokens.scss
│   │       ├── _spacing.tokens.scss
│   │       ├── _components.tokens.scss
│   │       ├── _light.theme.scss
│   │       ├── _dark.theme.scss
│   │       ├── _mixins.scss
│   │       ├── themes.scss
│   │       └── theme.service.ts        # Estado del tema; solo adaptador de la UI
│   └── environments/                  # Solo si se requieren
│       ├── environment.ts
│       └── environment.development.ts
├── e2e/                                # Cuando se configure prueba E2E
└── docs/
    ├── context/INDEX.md
    ├── decisions/
    └── contracts/
```

**Este árbol muestra ubicaciones y responsabilidades; no obliga a generar funcionalidades ficticias ni carpetas vacías.** La única estructura que debe quedar prevista desde el inicio es la raíz Angular y las áreas arquitectónicas indispensables. Con la primera necesidad real se crea cada subcarpeta.

`src/` **sí está permitido**: Angular usa su estructura convencional. Esta regla deliberadamente difiere del marco backend de proyectos `.csproj`.

### 4.3. Reglas internas por feature

- Una feature es una capacidad del usuario: `customers`, `payments`, `reports`, `settings`, no simplemente una categoría técnica.
- Páginas y rutas propias viven dentro de su feature; si una ruta contiene varios pasos, dividirla por flujos.
- `data-access/` contiene clientes HTTP y DTOs del contrato; `models/` contiene modelos internos de presentación/dominio de la feature; **no acoplar toda la UI a las respuestas crudas de la API**.
- `state/` aloja estado local a la feature; no hacer singleton global por defecto.
- `components/` contiene piezas específicas de ese flujo; solo promover a `shared/ui/` si existen **usos reales** en varias features y el componente no transporta lógica de negocio.
- No usar archivos índice para reexportar indiscriminadamente carpetas internas. Las importaciones públicas deben ser reducidas y explícitas.
- Lazy loading para features accesibles por ruta siempre que corresponda; no convertir pequeñas vistas de un mismo flujo en cientos de bundles sin evaluación.
- Mantener jerarquías cortas y consistentes: se permiten subcarpetas técnicas (`pages`, `components`, `data-access`) **dentro de cada feature**, no como arquitectura global.

Ejemplo de enrutamiento lazy (ajustar al API de la LTS elegida):

```ts
// src/app/app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'customers',
    loadChildren: () =>
      import('./features/customers/customers.routes').then(m => m.CUSTOMERS_ROUTES),
  },
  { path: '', pathMatch: 'full', redirectTo: 'customers' },
];
```

### 4.4. `core` frente a `shared`

**`core`**: infraestructuras con ciclo de vida de toda la aplicación (cliente HTTP, sesión, interceptores, errores globales, PWA, configuración pública). No colocar todo servicio en `core` por conveniencia.

**`shared`**: componentes y utilidades realmente reutilizables (botón, campo, modal genérico, tabla visual, tooltip, estado vacío, directiva de foco, validadores no comerciales), independientes de sesión y API de negocio. Deben recibir datos con inputs y comunicar acciones con outputs/eventos.

**`layouts`**: shell, header, navegación y zonas visuales. Puede integrar el selector Light/Dark mediante la API de `ThemeService`; no toma decisiones comerciales.

## 5. Angular moderno y convenciones técnicas

1. Preferir **standalone components**, bootstrap mediante `app.config.ts` y routing moderno, siempre que estén disponibles en la LTS elegida.
2. Aplicar TypeScript `strict` y plantillas con comprobación estricta. Evitar `any`, aserciones no justificadas y `!` para ocultar estados nulos.
3. Usar señales (`signal`, `computed`) para estado de UI cuando corresponda; RxJS para flujos asíncronos donde aporte valor. No usar ambos sin una necesidad específica.
4. Usar DI de Angular: servicios globales solo cuando su alcance deba ser global; servicios de feature con alcance de ruta/componente cuando corresponda.
5. No colocar HTTP directo en páginas o componentes UI. Los servicios de `data-access` encapsulan endpoints; los componentes consumen modelos/estado.
6. Centralizar base URL, timeouts, errores de transporte, headers pertinentes y serialización. No duplicar rutas de API dispersas por la UI.
7. Componentes con templates accesibles, inputs/outputs tipados, `track` en listas y control explícito de estados `loading`, `empty`, `error`, `success` cuando aplique.
8. Los guards mejoran la navegación pero **no sustituyen la autorización en el servidor**.
9. No crear abstracciones, repositorios frontend, interfaces vacías ni generic services artificiales: cada pieza debe resolver una necesidad concreta.
10. Estilos de presentación en archivos `.scss` propios; globales mínimos en `styles.scss`, tokens globales siempre en `themes`.
11. Mantener contratos públicos estables para las features y no filtrar implementaciones privadas hacia otras áreas.
12. Evitar dependencias innecesarias: evaluar bundle, mantenimiento, licencia, accesibilidad y compatibilidad antes de añadir paquetes.

## 6. Sistema de temas obligatorio

### 6.1. Política de temas

- Modos soportados: **`light`** y **`dark`** exclusivamente por defecto.
- **Primera visita sin preferencia guardada: `light`**, incluso si `prefers-color-scheme` indica oscuro.
- Si existe una preferencia explícita previa válida, restaurarla en las siguientes visitas. Si está corrupta, volver a `light`.
- Debe existir una forma visible y accesible de cambiar a oscuro/claro (por ejemplo, un botón o switch en header/ajustes) y su implementación real desde la fase base de UI.
- El selector cambia las variables semánticas del documento, no asigna estilos línea por línea.
- La preferencia se guarda en `localStorage` si está disponible. Si falla el almacenamiento, el cambio funciona al menos durante la sesión en memoria.
- Los temas siempre deben conservar legibilidad, estados de interacción y controles accesibles; no basta con invertir el fondo.
- No usar automáticamente colores de una empresa, producto existente o marca externa.

### 6.2. Única fuente de diseño en `themes/`

Mantener tokens semánticos, no colores de pantalla acoplados a un caso de uso.

| Categoría | Tokens esperados |
| --- | --- |
| Superficies | `--color-background`, `--color-surface`, `--color-surface-raised` |
| Texto | `--color-text`, `--color-text-muted` |
| Bordes | `--color-border`, `--color-border-strong` |
| Acciones | `--color-primary`, `--color-primary-hover`, `--color-on-primary` |
| Feedback | `--color-success`, `--color-warning`, `--color-danger`, `--color-info` |
| Accesibilidad | `--color-focus-ring`, `--color-disabled` |
| Medidas | `--space-*`, `--radius-*`, `--shadow-*`, `--font-*`, `--motion-*` |

### 6.3. Paleta de referencia inicial (sobria, editable)

Estas cifras son **puntos de partida** para UI neutra, no branding definitivo. Validar contraste de texto y controles en contexto (incluidas combinaciones sobre superficies).

| Token | Light (inicial) | Dark (inicial) |
| --- | --- | --- |
| Background | `#F6F7F8` | `#15191D` |
| Surface | `#FFFFFF` | `#1F252B` |
| Surface raised | `#F0F2F4` | `#293139` |
| Text principal | `#242B32` | `#F0F2F4` |
| Text muted | `#52606B` | `#B8C0C8` |
| Border | `#D8DEE3` | `#424B54` |
| Primary | `#405C72` | `#91A9BB` |
| Primary hover | `#344A5B` | `#ACC0CE` |
| On-primary | `#FFFFFF` | `#11191F` |
| Focus ring | `#3B637F` | `#ADC3D3` |

Para los colores de error, advertencia, éxito e información, definir tonos discretos **pero distinguibles** y sus correspondientes tokens de texto/superficie. Nunca depender solo del color para comunicar un estado.

### 6.4. Implementación mínima de variables SCSS/CSS

```scss
/* src/app/themes/_light.theme.scss */
:root,
:root[data-theme='light'] {
  color-scheme: light;
  --color-background: #f6f7f8;
  --color-surface: #ffffff;
  --color-surface-raised: #f0f2f4;
  --color-text: #242b32;
  --color-text-muted: #52606b;
  --color-border: #d8dee3;
  --color-primary: #405c72;
  --color-primary-hover: #344a5b;
  --color-on-primary: #ffffff;
  --color-focus-ring: #3b637f;
}
```

```scss
/* src/app/themes/_dark.theme.scss */
:root[data-theme='dark'] {
  color-scheme: dark;
  --color-background: #15191d;
  --color-surface: #1f252b;
  --color-surface-raised: #293139;
  --color-text: #f0f2f4;
  --color-text-muted: #b8c0c8;
  --color-border: #424b54;
  --color-primary: #91a9bb;
  --color-primary-hover: #acc0ce;
  --color-on-primary: #11191f;
  --color-focus-ring: #adc3d3;
}
```

```scss
/* src/app/themes/themes.scss */
@use './light.theme';
@use './dark.theme';
@use './colors.tokens';
@use './typography.tokens';
@use './spacing.tokens';
@use './components.tokens';

/* src/styles.scss debe cargar temas.scss UNA sola vez. */
```

**Regla de diseño**: colores específicos de un producto solo se cambian centralmente; los componentes utilizan `var(--color-...)`. Se permiten tokens de componente referenciando tokens semánticos, pero no hexadecimales arbitrarios dentro de `.page.scss` o `.component.scss`.

### 6.5. Contrato mínimo de `ThemeService`

- API requerida: `theme` (estado reactivo `light | dark`), `setTheme(mode)` y `toggleTheme()`.
- Inicialización: recuperar preferencia válida o `light`; aplicar `document.documentElement.dataset['theme']` y sincronizar el estado.
- Persistencia: escritura protegida contra fallas/denegación de storage.
- Acceso a DOM: si se habilita SSR/prerender, encapsular acceso con las protecciones de plataforma apropiadas. No asumir `window`/`document` en servidor.
- El tema no modifica endpoints, roles ni datos de negocio.
- El control en `shared/ui/theme-switch/` ofrece texto/etiqueta accesible, estado actual y operación de teclado.

Ejemplo conceptual de aplicación del estado:

```ts
// Código orientativo; integrarlo con ThemeService y comprobación de plataforma.
type ThemeMode = 'light' | 'dark';

function applyTheme(doc: Document, mode: ThemeMode): void {
  doc.documentElement.dataset['theme'] = mode;
}
```

### 6.6. Accesibilidad y diseño responsive

- Contraste WCAG AA según el rol del elemento; comprobar textos, estados, iconos y bordes interactivos.
- `:focus-visible` perceptible, navegación por teclado, labels de formularios, mensajes de error asociados a los campos.
- Layouts mobile-first, diseño funcional en móvil, tableta y escritorio; sin scroll horizontal inesperado.
- Usar unidades flexibles (`rem`, `%`, `clamp`, `minmax`) y breakpoints derivados del contenido cuando sea viable.
- Estados de carga con feedback apropiado y skeletons solo cuando aporten valor.
- Respetar `prefers-reduced-motion`; no bloquear acciones con animaciones largas.
- Soportar zoom a 200% y comprobar los componentes clave en dispositivos táctiles.
- El tema Dark/Light debe cubrir también dialogs, tooltips, toasts, gráficos, tablas, formularios y estados de error.

## 7. Estados, servicios HTTP y seguridad

### 7.1. Consumo de microservicios

1. Usar `HttpClient` y contratos tipados por feature; no llamar APIs directamente desde plantillas ni componentes presentacionales.
2. Inyectar una configuración pública de base URL (por ejemplo, `apiBaseUrl`); distinguir claramente build-time y runtime configuration cuando aplique.
3. Centralizar autenticación, correlación, manejo de `401/403`, errores y encabezados compartidos **solo cuando sean necesidades reales**.
4. Mantener DTOs del backend separados de modelos de pantalla cuando haya diferencias de semántica, transformación o formato.
5. Las decisiones de permisos sensibles, límites, validaciones autorizativas, cálculos críticos y protección multi-tenant residen en el servidor; el frontend solo mejora la experiencia.
6. Configuración pública en el cliente no es un mecanismo para guardar secretos. No incluir contraseñas, tokens privados ni claves secretas en `environment.*`.
7. Las credenciales en navegador deben manejarse mediante un diseño seguro acordado con el backend (cookies seguras, mitigación CSRF, expiración/sesión, etc.). Evitar recomendar `localStorage` para tokens sensibles como política por defecto.
8. Normalizar UTC/datos de fecha según el contrato de backend y convertir para visualización solo cuando corresponda.
9. `HttpInterceptor` debe ser transversal; no introducir dependencias de features en interceptores globales.

### 7.2. Estados de pantalla

- Separar `loading`, `empty`, `error`, `success` y `offline` cuando sean relevantes; la ausencia de datos no equivale siempre a error.
- Toda acción que cambie datos debe prevenir duplicaciones accidentales, mostrar resultado confiable y mantener coherencia de interfaz.
- Cancelar suscripciones/operaciones al destruir vistas o cambiar de navegación usando APIs de Angular apropiadas.
- Formularios: validaciones tipadas, mensajes claros, protección de doble envío cuando corresponda y respeto de errores provenientes del servidor.
- Si una operación involucra pagos, movimientos o acciones irreversibles, el servidor necesita idempotencia/validación; no atribuir esa garantía únicamente a bloquear un botón.

## 8. Rendimiento, routing y estado

- Lazy-load por feature/ruta para funcionalidades no iniciales. El shell inicial debe ser ligero.
- Medir tamaños de bundle y evitar grandes dependencias añadidas solo por un control simple.
- Compresión, cache correcto de recursos con hash y política de `index.html`/`ngsw.json` conforme a Angular PWA.
- Imágenes adaptativas, carga diferida y formatos eficientes; preservar dimensiones para evitar desplazamiento visual.
- Usar estado local por defecto; compartir estado solo si lo exige un flujo concreto.
- Los servicios de navegación, autenticación y tema pueden ser globales por su responsabilidad, sin convertir toda la lógica del producto en singletons.
- Medir Core Web Vitals mediante herramientas disponibles y registrar regresiones relevantes.
- No ejecutar trabajo pesado ni polling intenso permanentemente en segundo plano sin justificación y mecanismo de cancelación.

## 9. Git, documentación y control de cambios

### 9.1. Flujo de ramas

1. Inspeccionar repositorio, `git status --short` y `git branch --show-current` antes de cualquier edición.
2. Toda creación/modificación requiere rama `feature/*` válida y autorizada.
3. Si no existe Git, solicitar el nombre de la rama o autorización para crearla y ejecutar `git init -b feature/<descripcion>` **antes de generar archivos de aplicación**.
4. Preservar siempre modificaciones ajenas y no ejecutar comandos destructivos (`reset --hard`, `clean`) sin autorización específica.
5. No hacer `commit`, `push`, `merge`, `rebase`, tags ni PR sin solicitud expresa.

### 9.2. Archivos obligatorios

- `.gitignore`: Node/Angular (`node_modules/`, `dist/`, caches, IDE, coverage, secretos, logs), sin excluir archivos fuente, tests ni lockfiles.
- `README.md`: finalidad, requisitos, instalación, ejecución, pruebas y despliegue.
- `AGENTS.md`: bitácora cronológica de cada solicitud, decisiones, cambios, rama, verificaciones, resultados y limitaciones.
- `docs/context/INDEX.md`: propósito, límites, flujos, contratos API, decisiones, restricciones, pruebas y alcance PWA.
- `package-lock.json`: obligatorio en repositorios npm.
- `.nvmrc` o equivalente: Node LTS verificado y compatible.

Registrar cambios de versión Angular, activación PWA, decisiones de almacenamiento offline y personalizaciones centrales del tema como decisiones explícitas en el contexto del proyecto.

## 10. PWA obligatoria: inicialización y activación diferida

### 10.1. Pregunta obligatoria al iniciar un proyecto

> **¿Activo PWA ahora o durante el desarrollo?**  
> **A. Ahora (recomendado):** generar manifest, service worker, assets e instalación desde el inicio.  
> **B. Más adelante:** generar estructura frontend compatible; registrar el pendiente y activarlo antes de publicar en producción.

Esta pregunta **no permite elegir «proyecto no PWA»**: define solo la fase de implementación.

Si el usuario no responde y la tarea de creación ya está autorizada, elegir **A: activar ahora** por ser el valor seguro que cumple el requisito permanente. Si el usuario expresa B, respetar la fase elegida y mantener la puerta de despliegue de producción cerrada hasta terminarlo.

### 10.2. Procedimiento en proyecto nuevo o existente

1. Verificar rama `feature/*`, cambios locales, Angular LTS exacto y compatibilidad de Node/paquetes.
2. Para un proyecto nuevo, generar primero Angular feature-based; para uno existente, preservar sus rutas y archivos.
3. En la raíz Angular ejecutar la herramienta oficial de la misma **versión principal** instalada:

```bash
# Ejemplo para un proyecto Angular 21 LTS; reemplazar 21 cuando cambie la LTS oficial.
ng add @angular/pwa@21
```

4. Inspeccionar modificaciones automáticas: dependencias de service worker, registro, `manifest.webmanifest`, iconos, `index.html`, `angular.json` y `ngsw-config.json`. Las rutas exactas dependen de la versión de CLI; conservar las generadas correctamente.
5. Personalizar `name`, `short_name`, `start_url`, `scope`, `display`, `theme_color` y `background_color` del manifest con el nombre, ruta de despliegue e identidad del proyecto.
6. Generar iconos **reales** en tamaños/formatos adecuados, con recursos de marca definidos para ese producto; no reutilizar un logo ajeno.
7. Confirmar HTTPS en producción, manifest accesible, service worker registrado y ejecución de la SPA bajo su ruta real de despliegue.
8. Documentar capacidad offline **real** del producto. Como mínimo, verificar que la shell estática previamente visitada puede abrirse sin conexión, cuando el navegador lo permita.
9. Validar actualizaciones, reinicios, renovación de sesión, estado offline, cambios de tema y redirecciones de rutas.
10. Registrar versión, verificación y fecha de activación en `AGENTS.md`.

**Importante:** el service worker de Angular da cacheo básico. Si el producto requiere sincronización compleja, colas de escritura o caché avanzada, diseñarlo expresamente; no asumir que `ng add @angular/pwa` lo proporciona.

### 10.3. Componentes técnicos de PWA

- `core/pwa/`: `PwaUpdateService` para detectar disponibilidad de versiones y proponer actualización segura, `PwaInstallService` para gestionar prompts de instalación cuando el navegador los soporte y un adaptador de estado de conectividad si se necesita.
- `shared/ui/`: controles genéricos de instalación, aviso de actualización y estado offline si forman parte del alcance de la UI.
- `themes/`: estilos de banners/toasts/controles PWA que usan tokens; nunca estilos paralelos.
- La experiencia de instalación debe contemplar **diferencias de plataforma**: el evento `beforeinstallprompt` no está disponible universalmente; en iOS puede requerir guía de «Agregar a pantalla de inicio».
- No afirmar que un botón puede instalar automáticamente en todos los navegadores. Si el prompt no está disponible, explicar la ruta manual adecuada en la interfaz.

### 10.4. Caché, sesión y almacenamiento offline

**Política predeterminada prudente:**

- Cachear **recursos estáticos versionados** necesarios para la aplicación mediante configuración Angular.
- No cachear indiscriminadamente respuestas de endpoints autenticados, datos personales, registros de otros usuarios o tenants, ni respuestas de API con información sensible.
- Para datos de negocio, preferir acceso `network-only` por defecto. Cualquier `dataGroups`/IndexedDB de negocio requiere una estrategia documentada de autorización, caducidad, aislamiento, invalidación, resolución de conflictos y borrado de sesión.
- El evento `navigator.onLine` es una pista, **no prueba suficiente de que la API es alcanzable**; representar errores de red reales.
- No introducir operaciones offline de escritura ni «sincronización automática» ficticias sin contrato: requieren cola persistente, idempotencia de backend, reconciliación, seguridad y manejo de errores.
- Si se activa caché de datos, evaluar espacio, privacidad, logout, múltiples perfiles y dispositivos compartidos antes de lanzar.

### 10.5. Gestión de actualizaciones

- Detectar nuevas versiones con APIs vigentes de Angular y presentar una acción comprensible de actualizar/reiniciar cuando sea seguro.
- Evitar forzar actualización de modo que una operación del usuario se pierda.
- No activar indiscriminadamente workers nuevos a mitad de sesión sin comprobar compatibilidad de recursos y estado de formularios.
- Probar despliegue A→B y la recuperación cuando el manifest o la red fallen.
- El servidor/CDN debe exponer `ngsw.json`, `index.html`, scripts con hash y manifest sin reglas de caché contradictorias.

### 10.6. Puerta de publicación

**Se prohíbe presentar la primera versión de producción como terminada** si falta el manifest válido, el service worker operativo bajo HTTPS, la comprobación de instalación en navegadores objetivo, los iconos, y las verificaciones de modo claro/oscuro y experiencia offline acordada.

## 11. Crear un proyecto desde cero

### 11.1. Datos mínimos para iniciar

- Nombre deseado: se normaliza como `MF.nombre-del-proyecto`.
- Ruta de destino, confirmando que no sobrescribe un proyecto previo.
- Rama feature autorizada (`feature/inicializacion`, por ejemplo).
- Primeras funcionalidades realmente solicitadas; no inventar `customers` o `payments` cuando no correspondan.
- Política de activación PWA: **ahora** o **durante el desarrollo**, nunca «sin PWA».
- Detalles de autenticación/API y despliegue solo si ya se conocen y afectan la inicialización; no inventar endpoints ni credenciales.
- Colores de marca solo si los aporta el usuario; en caso contrario aplicar paleta neutra de §6.3.

Si solo se solicita estructura vacía, crear la base y los temas, selector de tema funcional y soporte PWA según fase elegida, sin páginas de negocio imaginarias.

### 11.2. Ejemplo operativo con Angular LTS

Los comandos son **orientativos para Angular 21 (21/10/2026)**, no una versión inmutable del estándar. Desde el directorio padre, y **solo tras obtener autorización para crear la rama**:

```bash
mkdir MF.rutas-de-cobro
cd MF.rutas-de-cobro

git init -b feature/inicializacion

# Usar una versión LTS de Node compatible previamente seleccionada.
# La CLI se ancla a la major LTS de Angular; no usar @latest sin comprobar su estado.
npx --yes --package @angular/cli@21 -- ng new rutas-de-cobro \
  --directory . \
  --routing \
  --style scss \
  --standalone \
  --skip-git

# Tras revisar package.json y fijar el runtime en .nvmrc:
npm install

# Si el usuario escogió 'PWA ahora':
npx ng add @angular/pwa@21

# Verificar con herramientas de la versión del proyecto:
npm run build
npm test -- --watch=false

git status --short
```

**Atención:** la CLI instalada y la major de `@angular/pwa` deben corresponder a la LTS seleccionada. Revisar las opciones reales de Angular CLI antes de ejecutar el ejemplo en otra versión. Si una plantilla o script no existe en la versión escogida, utilizar el equivalente documentado; nunca declarar un comando ejecutado exitosamente sin comprobarlo.

Después de ejecutar las plantillas:

1. Confirmar que el nombre de carpeta/repositorio respeta `MF.` y los identificadores npm/Angular siguen válidos.
2. Eliminar demos/plantillas sin valor contractual.
3. Crear `src/app/themes/` con tokens, modo Light/Dark, `ThemeService` y control funcional accesible.
4. Crear `core`, `shared`, `layouts` y `features` **en la medida en que su contenido exista**.
5. Configurar `app.routes.ts`, lazy loading y el shell mínimo.
6. Agregar contrato de `AGENTS.md` y `docs/context/INDEX.md`.
7. Revisar `.gitignore`, lockfile, Node LTS compatible y TypeScript strict.
8. Si PWA se difiere, registrar la decisión y el criterio obligatorio de activación preproducción.
9. Ejecutar pruebas, compilación y validaciones de comportamiento observables.
10. Entregar cambios y resultados; **no ejecutar commit/push por defecto**.

## 12. Pruebas y control de calidad

### 12.1. Pruebas requeridas por comportamiento

- **Unitarias:** services, validators, pipes, guards, transformaciones y estado de feature.
- **Componentes:** render, validaciones, interacciones, estados vacío/carga/error, eventos y accesibilidad básica.
- **Integración:** cliente HTTP ante respuestas correctas, `400`, `401`, `403`, `404`, `409`, `500`, timeout y red caída, cuando correspondan.
- **Navegación:** lazy loading, redirecciones, deep links y rutas protegidas según contrato.
- **Tema:** primera carga `light`, switch a `dark`, guardado/restauración, preferencia inválida, ausencia de storage y contraste.
- **PWA:** manifest, registro HTTPS, instalación donde se soporte, uso de shell sin red, cambios de versión, ausencia de servicio y recuperación.
- **End-to-end:** al menos flujos funcionales críticos acordados en contrato.

Una prueba no ejecutada **no se reporta como aprobada**. No inventar porcentajes de cobertura si no están establecidos, pero identificar rutas críticas sin verificar.

### 12.2. Lista de cierre

- [ ] Identidad externa `MF.nombre-del-proyecto` e identidad npm válida.
- [ ] Versión Angular LTS oficial verificada y parche fijado.
- [ ] Node LTS compatible, TypeScript/RxJS compatibles, lockfile actualizado.
- [ ] Arquitectura por features, sin acceso a implementaciones privadas entre features.
- [ ] `core` técnico y `shared` genérico, sin mezcla de lógica de negocio.
- [ ] `themes/` como única fuente de tokens y variables visuales globales.
- [ ] Primera carga sin preferencia en tema claro.
- [ ] Cambio claro/oscuro real, accesible y persistente.
- [ ] Paleta sobria, responsive y contraste suficiente en ambos modos.
- [ ] Consumo HTTP fuera de componentes visuales, contratos tipados.
- [ ] Sin secretos ni permisos confiados exclusivamente al cliente.
- [ ] Estados de pantalla, errores y accesibilidad comprobados.
- [ ] PWA instalada y validada antes de producción (o marcada pendiente si aún es desarrollo).
- [ ] No se almacenan datos sensibles indebidamente en caché offline.
- [ ] Compilación, tests y navegación ejecutados con resultados reales.
- [ ] Rama `feature/*`, `.gitignore`, `AGENTS.md` y contexto verificados.
- [ ] No hubo commit/push/merge no autorizado.

## 13. Guía de intervención para agentes de desarrollo

Por cada solicitud:

1. Identificar si se crea un proyecto nuevo o se modifica uno existente.
2. Leer este marco, `AGENTS.md`, `docs/context/INDEX.md`, contratos y decisiones realmente presentes.
3. Verificar rama `feature/*` y estado local sin modificar nada si falta autorización.
4. Confirmar alcance de la tarea y límites de la feature; en proyecto nuevo preguntar momento de activar PWA.
5. Comprobar Angular LTS y compatibilidad antes de escoger herramientas o comandos.
6. Crear/modificar solo los archivos dentro de la feature correspondiente, promoviendo reutilización a `shared` cuando esté justificada.
7. Toda nueva necesidad de estilo se materializa primero como token semántico en `themes/`, manteniendo Light/Dark.
8. Revisar efectos sobre routing, auth, estado, caché/actualizaciones PWA, accesibilidad y rendimiento.
9. Crear o actualizar pruebas pertinentes y ejecutar las comprobaciones posibles.
10. Registrar resultado, archivos cambiados, riesgos y pendientes reales en `AGENTS.md`.
11. Reportar lo validado y lo no validado sin simular una auditoría independiente.
12. No ampliar alcance, reescribir contratos ni ejecutar Git destructivo o publicar cambios sin autorización.

### 13.1. Auditoría independiente (si se solicita)

Un auditor distinto del implementador debe revisar en **solo lectura**: estructura feature-based, imports cruzados, tokens, tema inicial y persistencia, dependencia de Angular LTS, configuración PWA, caché, seguridad, pruebas, accesibilidad, contratos y evidencia. Reportar hallazgos con ruta, severidad, regla, evidencia y corrección propuesta. Si no existe un segundo agente, declarar que la auditoría independiente no se realizó.

### 13.2. Plantilla mínima de bitácora `AGENTS.md`

```markdown
## Solicitud <ID> — <fecha>
- Tipo: creación | modificación | análisis | auditoría
- Resumen:
- Rama:
- Alcance y feature:
- Decisiones confirmadas:
- PWA: activada | pendiente de activación | no afectada
- Tema: light/dark verificado | no afectado | pendiente
- Archivos modificados:
- Pruebas y compilación (resultado real):
- Riesgos y pendientes:
- Commit autorizado: NO (salvo petición expresa)
```

## 14. Instrucción maestra para agentes de IA

> Usa `FRAMEWORK_DESARROLLO_FRONTEND.md` como estándar obligatorio para proyectos personales frontend. Identifica el repositorio/directorio como `MF.nombre-del-proyecto`, con nombre compuesto en `kebab-case`; no fuerces ese prefijo con mayúsculas dentro de `package.json` si npm no lo permite. Usa Angular **en la mayor versión que oficialmente figure LTS**, con parche estable y Node.js LTS compatible; registra las versiones concretas. Trabaja exclusivamente en una rama `feature/*` autorizada y conserva cambios previos. Aplica arquitectura **feature-based**: las páginas, componentes, servicios HTTP, contratos y estado de cada flujo viven en `src/app/features/<feature>`; solo UI y utilidades genéricas en `shared`, capacidades transversales en `core`, shells en `layouts`. Define **toda la identidad visual** mediante tokens centralizados en `src/app/themes` con paleta sobria y editable. La primera visita debe usar **modo claro**, respetar una preferencia guardada en visitas posteriores y ofrecer un selector funcional y accesible **Light/Dark**. Toda app debe ser **PWA al momento de producción**: al iniciar pregunta si habilitar el service worker ahora (por defecto) o durante el desarrollo; si se difiere, registra la obligación y ejecútala antes del primer lanzamiento. No cachees datos de negocio privados indiscriminadamente ni prometas offline completo sin diseño probado. Utiliza Angular standalone y rutas lazy cuando corresponda; valida responsive, accesibilidad, seguridad, contratos, tests y build. Mantén `AGENTS.md` como registro verificable e índice de contexto. No inventes funcionalidades, no introduzcas dependencias sin necesidad y no hagas commit, push, merge ni PR salvo solicitud expresa.

## 15. Ejemplo de petición para iniciar un proyecto

> Crear un nuevo frontend personal llamado **`MF.rutas-de-cobro`** en una carpeta nueva. Utilizar Angular en la última versión formalmente **LTS**, con Node LTS compatible, TypeScript estricto y SCSS. Inicializar Git directamente en la rama autorizada `feature/inicializacion`. Definir arquitectura por features; crear solo las features que especifique el alcance. Preparar `shared`, `core`, `layouts` y `themes` con un estilo sobrio, colores en tokens y dos temas Light/Dark; primera visita en modo claro y selector de tema usable. **Activar PWA desde el inicio** con manifest, iconos y service worker oficial, verificando instalación y carga de shell offline. Incorporar `.gitignore`, pruebas mínimas reales, `AGENTS.md`, `docs/context/INDEX.md`, build y evidencias. No agregar funcionalidades no pedidas ni realizar commits.

## 16. Fuentes y revisión del estándar

Referencias técnicas oficiales (consultar al crear/migrar, porque cambian con el tiempo):

- Angular — versiones y ciclo LTS: https://angular.dev/reference/releases
- Angular — matriz de compatibilidad: https://angular.dev/reference/versions
- Angular — guía de estilo y organización por funcionalidades: https://angular.dev/style-guide
- Angular — service worker/PWA: https://angular.dev/ecosystem/service-workers/getting-started
- Angular — límites del service worker integrado: https://angular.dev/ecosystem/service-workers
- Angular — configuración de caché: https://angular.dev/ecosystem/service-workers/config
- Node.js — versiones y soporte: https://nodejs.org/en/about/previous-releases

**Vigencia:** el documento permanece aplicable hasta su reemplazo expreso por una versión posterior. Las tecnologías cambian, pero permanecen como invariantes la nomenclatura `MF.`, los límites feature-based, `shared`, temas centralizados, Light inicial, switch Dark/Light y el requisito PWA de producción.
