# 03 · Aplicación: motor, servicios y puertos

> **Propósito.** Catálogo de la capa de aplicación (`Aplicacion/`): el `MotorDeModeracion`, los
> servicios de caso de uso, el cargador de políticas y el gestor de conexiones, más las interfaces de
> `Puertos/`. **Fuente primaria:** `src/DiscordModeradorBot.Servicio/Aplicacion/`. El flujo del motor
> se detalla en [`01_arquitectura-y-pipeline.md`](01_arquitectura-y-pipeline.md); los adaptadores en
> [`04_infraestructura.md`](04_infraestructura.md).

## MotorDeModeracion

`Aplicacion/MotorDeModeracion.cs` — orquesta el pipeline de 9 etapas (ver [`01`](01_arquitectura-y-pipeline.md#pipeline-de-moderación-de-9-etapas)).
Entrada `ProcesarAsync(MensajeEntrante, ct)` → `Incidente?`. Depende (por constructor) de los estados
en memoria, los evaluadores de dominio, `ICargadorPoliticas`, `IAdaptadorGateway`, los repositorios de
incidentes/servidores/exenciones, `IReloj` y `ILogger`. Dos constructores: uno con `ICargadorPoliticas`
(producción, políticas por servidor) y uno de compatibilidad con lista fija de políticas
(pruebas/walking skeleton, que envuelve la lista en `CargadorPoliticasFijas`).

Responsabilidades clave del motor (no repetir en otros índices):

- Ordena políticas por prioridad y aplica **primera coincidencia** (RN-04), con `Continuar` para
  seguir evaluando.
- Resuelve el eje de disparo (composición / contenido / conducta).
- Toma la ventana de antirrebote **efectiva** del servidor si la config aporta parámetros; si no, la
  del motor.
- Reclasifica el resultado del incidente tras ejecutar las acciones (`PeorResultado`, ADR-08) y nunca
  se cae por jerarquía/permisos.

## Servicios de caso de uso

Todos scoped (registrados en `RegistroDependencias`). Cada uno encapsula un CU y devuelve un record
de resultado con código de error del CU cuando falla.

| Servicio | CU / rebanada | Responsabilidad | Códigos de error notables |
|---|---|---|---|
| `ServicioAdministrador` | CU-08, CU-09, RN-12/13 | Alta inicial (first-run), autenticación, cambio de contraseña; usa `IServicioHashContrasena` y `ControlIntentosAutenticacion` | `SETUP_YA_COMPLETADO`, `SETUP_CONTRASENA_DEBIL`, `ACTUAL_INVALIDA`, `NUEVA_DEBIL`; `ResultadoAutenticacion.{DemasiadosIntentos,CredencialesInvalidas}` |
| `ServicioDesbaneo` | CU-07 | Revierte un baneo ejecutado desde el panel (registra quién/cuándo); no restaura mensajes borrados | `INCIDENTE_NO_ENCONTRADO`, no-baneo, `YaRevertido` |
| `ServicioExenciones` | CU-15, RN-07/08 | Alta/listado/baja de exenciones; valida snowflake al guardar | `EXENCION_IDENTIFICADOR_INVALIDO`, `EXENCION_DUPLICADA` |
| `ServicioConfiguracionModeracion` | CU-11, RN-10 | Configuración normalizada (grupos/eventos/acciones) y parámetros por servidor; valida contra descriptores | `ResultadoValidacionDescriptor`, `ResultadoConfiguracion` |
| `ServicioPruebaConfiguracion` | CU-12, RN-16 | Smoke test del servidor (token/permisos/intents) y activación; envío de mensaje de prueba (CU-05) | `ResultadoProbarYActivar` |
| `ServicioRegistroServidor` | CU-10 | Registro de servidor (cifra el token, designa canal de salida) | `ServidorYaRegistrado`, `ServidorIdentificadorInvalido`, `ServidorTokenVacio`, `CanalIdentificadorInvalido` |
| `ServicioRegistroReglaContenido` | CU-04, RN-03 | Alta de reglas de contenido; valida el patrón al guardar; matchTimeout desde descriptor | `CONTENIDO_PATRON_INVALIDO` |

## Cargador de políticas y gestor de conexiones

- `ICargadorPoliticas` (`Aplicacion/ICargadorPoliticas.cs`) — puerto que entrega las políticas de un
  servidor y sus parámetros. Implementaciones: `CargadorPoliticasDesdeConfiguracion` (producción) y
  `CargadorPoliticasFijas` (lista fija, dentro del motor).
- `CargadorPoliticasDesdeConfiguracion.cs` — puente config→dominio: toma eventos → grupos → reglas
  (contenido por id, conducta por clave) → acciones y los materializa a `Politica` + `ComposicionPolitica`
  + `GrupoDeReglas`. **Tolerante a config incompleta** (RN-15, RC-04, ADR-08): omite grupos sin reglas
  válidas y eventos sin grupos válidos, sin tumbar el resto. Es lo que hace que el panel DIRIJA la
  moderación (ya no hay política hardcodeada).
- `GestorConexionesGateway.cs` — lógica de aplicación (independiente del SDK) del ciclo de vida de
  conexiones reales por servidor (ADR-13, CU-13): abre la conexión con el token descifrado, enruta
  cada mensaje al motor, refleja el estado de conexión, cierra al desactivar. El token nunca se
  loguea (RN-14). Testeable con dobles (`IFabricaClienteGateway`/`IClienteGatewayServidor`).

## Puertos (`Aplicacion/Puertos/`)

Interfaces que la infraestructura implementa. Ver la tabla puerto→adaptador en
[`01`](01_arquitectura-y-pipeline.md#puertos-y-adaptadores).

| Puerto | Miembros principales |
|---|---|
| `IAdaptadorGateway` | `event MensajeRecibido`, `bool EsSimulado`, `ProbarConfiguracionAsync`, `EnviarMensajePruebaAsync`, `ReportarAsync`, `BanearConBorradoAsync`, `AplicarTimeoutAsync`, `ExpulsarAsync`, `AsignarRolAsync`, `QuitarRolAsync` |
| `IClienteGatewayServidor` (`: IAsyncDisposable`) | `event MensajeRecibido`, `ConectarAsync(token)`, `DetenerAsync` |
| `IFabricaClienteGateway` | crea un cliente por servidor |
| `IEstadoConexionGateway` | `Actualizar(...)`, `Obtener(...)`, `ObtenerTodos()`; enum `MotivoEstadoConexion` |
| `IRepositorioServidores` | `ExisteAsync`, `AgregarAsync`, `ObtenerAsync`, `ListarAsync`, `ActualizarEstadoAsync` |
| `IRepositorioIncidentes` | `AgregarAsync`, `ListarAsync`, `BuscarAsync(FiltroIncidentes)→PaginaIncidentes`, `ObtenerAsync(id)`, `MarcarRevertidoAsync` |
| `IRepositorioReglasContenido` | `AgregarAsync`, `ListarPorServidorAsync`, `ActualizarAsync`, ... |
| `IRepositorioConfiguracion` | grupos/eventos/acciones: `AgregarGrupoAsync`, `ActualizarGrupoAsync`, `ListarGruposAsync`, ... (records `GrupoPersistido`, etc.) |
| `IRepositorioExenciones` | `AgregarAsync`, `ListarPorServidorAsync`, `QuitarAsync` |
| `IRepositorioAdministrador` | `ExisteAsync`, `ObtenerAsync`, `AgregarAsync`, `ActualizarAsync` |
| `IReloj` | `Ahora` (inyectable para tests) |
| `IServicioCifrado` | `Cifrar`, `Descifrar` |
| `IServicioHashContrasena` | `Hash(contrasena)`, `Verificar(contrasena, resguardoPhc)` |

`FiltroIncidentes.cs` define el filtro/paginación de la búsqueda de incidentes (usado por el panel,
[`05`](05_panel-web.md)).

## Gaps / no verificado

- Los constructores exactos de cada `Servicio*` no se listan aquí; ver la clase para la firma completa.
- `IRepositorioConfiguracion` expone varios records persistidos (`GrupoPersistido`, `EventoPersistido`,
  `ReglaContenidoPersistida`) cuyo detalle de campos vive en el propio puerto.
</content>
