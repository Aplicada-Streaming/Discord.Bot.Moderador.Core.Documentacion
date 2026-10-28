---
doc_id: GLOSSARY-000
doc_type: glossary
title: Glosario de la solución Administrador de Bots Moderador para Discord
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: equipo-discordmoderador (propuesto)
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, ops, qa, arquitectura, agentes-automaticos]
classification: uso-interno
traces: []
---

# Glosario

> Vocabulario controlado y normativo de la solución (Marco §12.3, `Rule-Dual-Audience`). Cada
> término se usa con este significado en toda la documentación; los sinónimos se registran como
> **alias** del término canónico. Fuente primaria: código en
> `../../Discord.Bot.Moderador.Core/src/DiscordModeradorBot.Servicio/`. Los términos verificados contra
> el código llevan su traza de origen.

## Cómo leer este glosario

Cada entrada indica el **término canónico**, su definición, sus **alias** (formas que NO deben usarse
como si fueran conceptos distintos) y la **traza** al elemento de código o documento que lo define.

## Términos del dominio de moderación

| Término canónico | Definición | Alias | Traza (origen) |
|---|---|---|---|
| **Ráfaga distribuida** | Patrón de spam en el que un mismo usuario publica en **varios canales distintos** dentro de una ventana de tiempo corta. El discriminador es la *cantidad de canales distintos*, no el volumen de mensajes. | fan-out, spam distribuido | `Dominio/Conducta/EvaluadorRafagaDistribuida.cs`; `RegistroDescriptores.UmbralCanalesDistintos` |
| **Incidente** | Registro persistente de una intervención de moderación: servidor, usuario, política que disparó, modo, acción representativa, resultado, evidencia (mensajes y canales) e instante. Es la unidad de trazabilidad y revisión. | — | `Dominio/Moderacion/Incidente.cs`; tabla `Incidente` |
| **Política** | Regla de negocio configurada que, al coincidir con un mensaje, dispara un conjunto ordenado de acciones. En el modelo persistido se materializa como **Evento**. Tiene prioridad, modo y bandera de continuación. | Evento (persistencia) | `Dominio/Moderacion/Politica.cs`; tabla `Evento` |
| **Evento** | Nombre de la **Política** en el modelo de datos normalizado (R7): fila de la tabla `Evento` con sus grupos y acciones. | Política (dominio) | tabla `Evento`; `Infraestructura/Persistencia/Entidades/EventoEntidad.cs` |
| **Grupo de reglas** | Conjunto de reglas (de contenido y/o de conducta) que se combinan según un **modo de coincidencia** (todas / alguna / mínimo N). Los grupos se combinan entre sí para formar la condición de disparo de una política. | — | `Dominio/Moderacion/Reglas/GrupoDeReglas.cs`; tabla `GrupoDeReglas` |
| **Regla de contenido** | Predicado **sin estado** sobre el texto de un mensaje (p. ej. expresión regular), evaluado con un tope de tiempo anti-ReDoS. | — | `Dominio/Contenido/EvaluadorReglaContenido.cs`; tabla `Regla` |
| **Regla de conducta** | Predicado **con estado** sobre la actividad acumulada del usuario (p. ej. la ráfaga distribuida), que observa la ventana deslizante de actividad. | — | `Dominio/Moderacion/Reglas/ReglaEvaluableConducta.cs`; `Dominio/Conducta/` |
| **Acción** | Efecto que una política ejecuta sobre el emisor o el servidor: banear, banear con borrado retroactivo, timeout, expulsar, asignar/quitar rol, reportar a canal privado. Tiene un **orden de ejecución** declarado. | medida, sanción | `Dominio/Moderacion/Accion.cs`; `TipoAccion`; tabla `Accion` |
| **Borrado retroactivo** | Purga de los mensajes recientes del emisor al ejecutar un baneo, acotada a una ventana en días con **tope de plataforma de 7 días**. | limpieza retroactiva | `RegistroDescriptores.VentanaBorradoRetroactivoDias`; `VentanaBorradoMaximaDias = 7` |
| **Exención** | Sujeto (usuario, rol o canal) marcado en un servidor para que sus mensajes se **descarten antes** de toda evaluación de moderación. | lista blanca, allowlist | `Dominio/Exenciones/Exencion.cs`; tabla `Exencion` |
| **Antirrebote** | Supresión de acciones repetidas sobre un mismo usuario ya accionado dentro de una ventana de tiempo, para no duplicar incidentes durante un único ataque. El estado vive **en memoria**, no se persiste. | debounce | `Dominio/Conducta/EstadoAntirreboteEnMemoria.cs`; `RegistroDescriptores.VentanaAntirreboteSegundos` |
| **Modo** | Determina si una política **ejecuta** acciones reales (`Ejecucion`) o solo registra lo que habría hecho sin tocar la plataforma (`Simulacion`). | modo simulación / modo ejecución | `Dominio/Moderacion/Modo.cs` |
| **Canal de salida** | Canal privado del servidor designado para publicar los reportes de incidentes. | canal de incidencias, canal de reportes | `Dominio/Servidores/CanalDeSalida.cs`; `Servidor.SnowflakeCanalSalida` |
| **Reporte de incidente** | Texto compuesto desde un incidente que se publica en el canal de salida; incluye la advertencia cuando la contención quedó *no accionable*. | — | `Dominio/Moderacion/ReporteIncidente.cs` |
| **Resultado de moderación** | Clasificación del desenlace de un incidente: `Ejecutada`, `Simulada`, `NoAccionable`, `Fallida`. El "peor" resultado de las acciones de contención manda. | — | `Dominio/Moderacion/ResultadoModeracion.cs` |
| **No accionable** | Estado de un incidente cuya acción de contención no pudo aplicarse por **jerarquía superior del objetivo o permisos faltantes** del bot; el incidente igual se registra y se reporta. | — | `MotorDeModeracion.cs` (RN-01, ADR-08) |
| **Descriptor de parámetro** | Definición única (etiqueta, leyenda, default, mínimo, máximo, ejemplos) de un parámetro configurable del dominio; es la **fuente única de verdad** de defaults y límites (config dirigida por esquema). | — | `Dominio/Configuracion/DescriptorParametro.cs`; `RegistroDescriptores.cs` |
| **Prueba de configuración** | Validación previa que debe superar un servidor antes de activarse para moderar (verifica que el bot puede operar con el token y permisos). | smoke test de servidor | `Dominio/Servidores/PruebaConfiguracion.cs`; `Aplicacion/ServicioPruebaConfiguracion.cs` |

## Términos de plataforma y técnicos

| Término canónico | Definición | Alias | Traza (origen) |
|---|---|---|---|
| **Snowflake** | Identificador de 64 bits de Discord (servidor, usuario, canal, mensaje, rol). Se almacena como **TEXTO** para preservar identidad y ordenar de forma estable en SQLite. | ID de Discord | `Dominio/Snowflake.cs` (RN-08, RC-02) |
| **Gateway** | Puerto de salida hacia Discord. Tiene dos adaptadores seleccionables por configuración: **Simulado** (sin red ni token) y **Discord** (real, con Discord.Net). | adaptador de gateway | `Aplicacion/Puertos/IAdaptadorGateway.cs`; `Infraestructura/Gateway/` |
| **Modo Simulado** | Adaptador de gateway por defecto: no abre red ni requiere token; un *walking skeleton* inyecta escenarios de demostración. Es el modo de desarrollo y de pruebas. | gateway simulado | `Infraestructura/Gateway/AdaptadorGatewaySimulado.cs`; `ModoGateway.Simulado` |
| **Modo Discord** | Adaptador de gateway real: abre una conexión por servidor con Discord.Net y enruta los mensajes al motor. | gateway real | `Infraestructura/Gateway/AdaptadorGatewayDiscord.cs`; `ModoGateway.Discord` |
| **Walking skeleton** | Servicio en segundo plano que, en modo Simulado, inyecta mensajes de demostración para ejercitar el pipeline de moderación sin Discord. | esqueleto ambulante | `Infraestructura/WalkingSkeletonHostedService.cs` |
| **Gestor de conexiones** | Servicio en segundo plano que, en modo Discord, abre y mantiene una conexión de gateway por cada servidor registrado y activo. | — | `Aplicacion/GestorConexionesGateway.cs`; `Infraestructura/GestorConexionesGatewayHostedService.cs` |
| **Administrador único** | Único rol de la solución: la persona que opera el panel. Se autentica por cookie; sus credenciales se resguardan con PBKDF2 en formato PHC. | admin | `Dominio/Administracion/Administrador.cs`; `Aplicacion/ServicioAdministrador.cs` (RN-12, ADR-03) |
| **Clave maestra** | Secreto (variable de entorno `DISCORDMODERADOR_CLAVE_MAESTRA`) con el que se cifran/descifran los tokens de bot en reposo (AES). Nunca se guarda en el repositorio. | master key | `Infraestructura/Seguridad/ServicioCifradoAes.cs` (RN-14, ADR-07) |
| **First-run** | Alta inicial de credenciales del administrador la primera vez que se abre el panel (página `/configuracion-inicial`). | primer ingreso, setup inicial | `Components/Pages/ConfiguracionInicial.razor` (CU-08) |
| **Rebanada vertical** | Unidad de entrega del proyecto (R1..R7): un corte funcional de punta a punta a través de todas las capas. | vertical slice | `SDD2.2D/docs/07_plan-sprint/` |

## Prefijos de identificadores estables (Marco §12.3)

| Prefijo | Significado | Dónde viven |
|---|---|---|
| `NB-` | Necesidad de negocio | `SDD2.2D/docs/01_necesidades_negocio/` (fuente) |
| `CU-` | Caso de uso | `SDD2.2D/docs/02_especificacion_funcional/casos-de-uso/` (fuente) |
| `RN-` | Regla de negocio | `SDD2.2D/docs/02_especificacion_funcional/reglas-de-negocio/` (fuente) |
| `RC-` | Regla conceptual de modelo | `SDD2.2D/docs/02_especificacion_funcional/modelo-datos/` (fuente) |
| `ADR-` | Decisión de arquitectura | `04-decisions/` (este set) y `SDD2.2D/docs/05_arquitectura_tecnica/adrs/` (fuente) |
| `TC-` | Caso de prueba derivado del modelo | `03-data/test-data-matrix.md` (este set) |
| `RB-` | Runbook operativo | `07-operations/` (este set) |

> **Nota de trazabilidad:** los prefijos `NB-`, `CU-`, `RN-`, `RC-` fueron acuñados por la
> documentación SDD 2.2 existente del proyecto (`../../Discord.Bot.Moderador.Core/SDD2.2D/`), que este
> conjunto documental toma como fuente secundaria y referencia por ID. Este set genera los `TC-` (§15)
> y consolida los `ADR-` en su propio índice.
