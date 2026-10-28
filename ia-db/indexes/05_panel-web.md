# 05 · Panel web (Blazor Server)

> **Propósito.** Mapa del panel de administración: páginas Blazor, diálogos, layout, rutas, gating de
> autorización y endpoints minimal API de autenticación. **Fuente primaria:**
> `src/DiscordModeradorBot.Servicio/Components/` y `Program.cs`. Los servicios que consumen las
> páginas están en [`03_aplicacion-servicios.md`](03_aplicacion-servicios.md); la postura de seguridad
> en [`09_configuracion-y-seguridad.md`](09_configuracion-y-seguridad.md).

## Composición Blazor

- **Render**: Blazor Server interactivo — `AddRazorComponents().AddInteractiveServerComponents()` +
  `AddMudServices()` (MudBlazor 9.5). Root `Components/App.razor`, ruteo en `Components/Routes.razor`,
  imports en `Components/_Imports.razor`.
- **Static assets**: `MapStaticAssets` + `UseStaticFiles` (este último garantiza `blazor.web.js` en el
  contenedor). En entorno `E2E` se fuerza `UseStaticWebAssets()`.

## Rutas y autorización

`Routes.razor` envuelve el router en un `AuthorizeRouteView` con `MainLayout` por defecto; el
contenido `NotAuthorized` monta `RedireccionAutenticacion`, que dirige a `/configuracion-inicial` si
no existe administrador o a `/ingresar` si existe pero no hay sesión (RN-12, CU-08/CU-09).

| Ruta | Página (`Components/Pages/`) | Layout | Autorización |
|---|---|---|---|
| `/` | `Home.razor` | MainLayout | `[Authorize]` |
| `/incidentes` | `Incidentes.razor` | MainLayout | `[Authorize]` |
| `/incidentes/{Id:int}` | `IncidenteDetalle.razor` | MainLayout | `[Authorize]` |
| `/servidores` | `Servidores.razor` | MainLayout | `[Authorize]` |
| `/configuracion` | `Configuracion.razor` | MainLayout | `[Authorize]` |
| `/exenciones` | `Exenciones.razor` | MainLayout | `[Authorize]` |
| `/cambiar-contrasena` | `CambiarContrasena.razor` | MainLayout | `[Authorize]` |
| `/configuracion-inicial` | `ConfiguracionInicial.razor` | **EmptyLayout** | pública (first-run, CU-08) |
| `/ingresar` | `Ingresar.razor` | **EmptyLayout** | pública (login, CU-09) |
| `/Error` | `Error.razor` | EmptyLayout | pública |
| `/not-found` | `NotFound.razor` | EmptyLayout | pública |

Navegación lateral (`Layout/NavMenu.razor`): **Incidentes**, **Servidores**, **Configuración**,
**Exenciones** (MudNavMenu/MudNavLink con íconos Material).

## Páginas por caso de uso

| Página | Función | CU |
|---|---|---|
| `Incidentes` | Listado paginado/filtrable de incidentes (`IRepositorioIncidentes.BuscarAsync`) | CU-06 |
| `IncidenteDetalle` | Detalle de un incidente + acción de desbaneo (`ServicioDesbaneo`) | CU-06, CU-07 |
| `Servidores` | Registro/edición de servidores, token, canal de salida, prueba y activación | CU-10, CU-12, CU-13 |
| `Configuracion` | Configuración de moderación: grupos/eventos/acciones, parámetros, reglas de contenido | CU-04, CU-11 |
| `Exenciones` | Alta/baja de exenciones por usuario/rol/canal | CU-15 |
| `ConfiguracionInicial` | Alta inicial de credenciales del administrador (first-run) | CU-08 |
| `Ingresar` | Login del administrador | CU-09 |
| `CambiarContrasena` | Cambio de contraseña del administrador autenticado | RN-13 |
| `Home` | Portada autenticada | — |

Diálogos (MudDialog): `DialogoConfirmacionDesbaneo`, `DialogoConfirmacionEliminarServidor`,
`DialogoConfirmacionLimpiarIncidentes`, `DialogoEditarRegla`, `DialogoEditarServidor`.

Layout: `Layout/MainLayout.razor` (chrome autenticado), `ChromePanel.razor`, `NavMenu.razor`,
`ReconnectModal.razor` (+`.css`), `EmptyLayout.razor` (páginas públicas). Tema en `Theme/AppTheme.cs`,
estilos en `wwwroot/app.css`.

## Endpoints de autenticación (minimal API en `Program.cs`)

Blazor Server interactivo no puede setear cookies desde un componente; el sign-in/out lo hacen
endpoints minimal API a los que postean las páginas públicas. Todos validan **antiforgery**
(`EsRequestValidaAsync`) y devuelven redirects neutros.

| Endpoint | Método | Función | Errores (query `?error=`) |
|---|---|---|---|
| `/api/auth/configuracion-inicial` | POST | First-run: crea el administrador inicial (CU-08) | `ANTIFORGERY`, `confirmacion`, `SETUP_YA_COMPLETADO`, `SETUP_CONTRASENA_DEBIL`, `SETUP_IDENTIFICADOR_INVALIDO` |
| `/api/auth/ingresar` | POST | Login: autentica y emite la cookie de sesión (rol `administrador`) | `ANTIFORGERY`, `AUTH_DEMASIADOS_INTENTOS`, `AUTH_CREDENCIALES_INVALIDAS` |
| `/api/auth/salir` | POST | Logout (`SignOutAsync`) | — |
| `/api/auth/cambiar-contrasena` | POST · `RequireAuthorization` | Cambio de contraseña del admin autenticado (RN-13) | `ANTIFORGERY`, `confirmacion`, `ACTUAL_INVALIDA`, `NUEVA_DEBIL`, `SIN_CUENTA` |

La clave del control de intentos combina identificador normalizado + IP (`ClaveIntentos`), sin filtrar
si el usuario existe (sin enumeración de cuentas). Login exitoso emite claims `Name` + `Role=administrador`
y redirige a `/incidentes`.

**Endpoints de sembrado E2E** (`/e2e/seed/administrador`, `/e2e/seed/incidente-baneo`) se montan
**solo** en entorno `E2E` (`MapearEndpointsSembradoE2E`); no existen en dev/producción.

## Gaps / no verificado

- El detalle interno de cada `.razor` (bindings, componentes MudBlazor concretos) no se indexa; ver el
  archivo de la página. Este índice cubre rutas, propósito y wiring de autenticación.
</content>
