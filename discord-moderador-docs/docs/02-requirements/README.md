---
doc_id: REQ-INDEX-001
doc_type: requirements-index
title: Índice de requisitos y matriz de trazabilidad
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: equipo-discordmoderador (propuesto)
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, qa, arquitectura, negocio, agentes-automaticos]
classification: uso-interno
traces: [NB-01, NB-02, NB-03, NB-04, NB-05, NB-06, NB-07, CU-01, CU-02, CU-03, CU-04, CU-05, CU-06, CU-07, CU-08, CU-09, CU-10, CU-11, CU-12, CU-13, CU-14, CU-15, CU-16, RN-01, RN-02, RN-03, RN-04, RN-05, RN-06, RN-07, RN-08, RN-09, RN-10, RN-11, RN-12, RN-13, RN-14, RN-15, RN-16]
---

# Índice de requisitos y matriz de trazabilidad

> Índice navegable de las **7 necesidades de negocio (NB)**, **16 casos de uso (CU)** y
> **16 reglas de negocio (RN)** de la solución **Administrador de Bots Moderador para Discord**, con
> su **trazabilidad** hasta el módulo de código y los casos de prueba. Glosario:
> [`../../GLOSSARY.md`](../../GLOSSARY.md).

## Alcance de este documento

- **La especificación funcional autoritativa vive en la SDD 2.2** (fuente secundaria de este set):
  - NB: `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/01_necesidades_negocio/`
  - CU: `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/02_especificacion_funcional/casos-de-uso/`
  - RN: `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/02_especificacion_funcional/reglas-de-negocio/`
  - RC (reglas conceptuales de modelo): `.../02_especificacion_funcional/modelo-datos/reglas-conceptuales-de-modelo/`
- Este documento es el **índice y la trazabilidad**, no el SRS completo. Se **referencia** cada
  requisito por ID y ruta, no se reescribe (SSOT). Queda **declarada la omisión** de una carpeta
  `functional/` con especificaciones íntegras: la especificación de detalle no se duplica aquí.
- La **fuente de verdad última** es el código en
  `../../../../Discord.Bot.Moderador.Core/src/DiscordModeradorBot.Servicio/`; los módulos citados abajo
  son las rutas verificadas contra ese código.

## Necesidades de negocio (NB)

| NB | Título | Descripción (1 línea) | CU que la realizan | MoSCoW |
|---|---|---|---|---|
| NB-01 | Corte automático de la ráfaga de spam distribuido | Detectar y cortar ráfagas de spam distribuido que la moderación manual y el filtro nativo no frenan a tiempo. | CU-01, CU-02 | Must |
| NB-02 | Limpieza retroactiva de los mensajes del incidente | Borrar retroactivamente los mensajes generados durante el incidente detectado. | CU-03 | Must |
| NB-03 | Contención de contenido no deseado por patrón | Contener contenido no deseado identificándolo por patrón. | CU-04 | Must |
| NB-04 | Trazabilidad de incidentes y control de falsos positivos | Registrar/reportar incidentes y controlar falsos positivos (revisión y desbaneo). | CU-05, CU-06, CU-07 | Must |
| NB-05 | Configuración autónoma de la moderación por el administrador | Que el administrador configure la moderación de forma autónoma (panel, cuenta admin, ayuda contextual). | CU-08, CU-09, CU-10, CU-11 | Must |
| NB-06 | Operación confiable y validación previa | Garantizar operación confiable con validación previa (prueba de configuración, reconexión, estado). | CU-12, CU-13 | Should |
| NB-07 | Mitigación del riesgo de moderación errónea | Mitigar la moderación errónea (simulación, exenciones, antirrebote). | CU-14, CU-15, CU-16 | Should |

> Mapeo NB→CU explícito en `necesidades-negocio_v1.0.md §2`, sin solapamientos de CU entre NB.
> Raíces: NB-01 y NB-05; NB-02/03/04/07 dependen de NB-01; NB-04/06/07 dependen de NB-05.

## Casos de uso (CU)

| CU | Nombre | Descripción (1 línea) |
|---|---|---|
| CU-01 | Detectar ráfaga distribuida | Reconoce el patrón de ráfaga (un usuario publicando en más canales distintos que el umbral en ventana corta) para disparar la contención. |
| CU-02 | Banear al emisor de la ráfaga | Ejecuta el baneo automático del emisor cuando la política se dispara por ráfaga distribuida, sin intervención humana. |
| CU-03 | Banear con borrado retroactivo | Elimina, en el mismo acto del baneo, los mensajes del usuario dentro de una ventana hacia atrás configurable. |
| CU-04 | Detectar contenido no deseado | Evalúa el contenido de un mensaje contra criterios (regex o palabras/frases) y, si cumple, dispara la contención. |
| CU-05 | Reportar a un canal privado | Envía a un canal privado un reporte con los mensajes accionados y los canales afectados al dispararse una política. |
| CU-06 | Revisar incidentes desde el panel | Permite consultar los incidentes con copia de mensajes y canales para evaluar falsos positivos. |
| CU-07 | Revertir contención (desbaneo) | Permite revertir un baneo desde el panel cuando la revisión concluye que fue un falso positivo. |
| CU-08 | Alta de credenciales del administrador (first-run) | Crea, en el first-run, la cuenta única de administrador con sus credenciales. |
| CU-09 | Autenticar al administrador | Verifica las credenciales del administrador para abrir una sesión autorizada. |
| CU-10 | Registrar un servidor con su token | Registra un servidor a moderar junto con su token de bot como contexto independiente del firewall multi-contexto. |
| CU-11 | Administrar reglas, grupos, eventos y acciones | Crear/ajustar reglas, agruparlas, definir eventos con acciones y configurar parámetros con ayuda contextual. |
| CU-12 | Probar la configuración de un servidor | Verifica token, permisos, jerarquía, recepción de eventos y canales antes de activar. |
| CU-13 | Reconectar y mostrar estado de conexión | Recupera automáticamente la conexión ante caídas transitorias y mantiene visible el estado de cada servidor. |
| CU-14 | Ejecutar en modo simulación | Evalúa la política y registra la acción que habría hecho sin ejecutarla, para calibrar antes de promover. |
| CU-15 | Definir exenciones | Declara sujetos exentos (rol, usuario o canal) que se descartan antes de evaluar cualquier regla. |
| CU-16 | Antirrebote por usuario | Impide que un usuario reciba la misma acción varias veces durante una única ráfaga. |

## Reglas de negocio (RN)

| RN | Nombre | Enunciado (1 línea) | CU afectados |
|---|---|---|---|
| RN-01 | Jerarquía de roles del bot | El bot no acciona sobre un usuario de rol superior/igual ni sin el permiso requerido; se registra y reporta como no accionable. | 02, 03, 04, 07, 12 |
| RN-02 | Tope del borrado retroactivo | La ventana de borrado nunca excede 7 días; todo valor mayor se acota a ese tope. | 03, 11 |
| RN-03 | Validez del patrón de contenido | Una regla de contenido solo se aplica si su criterio es válido (regex que compila o conjunto no vacío). | 04, 11 |
| RN-04 | Evaluación por prioridad, primera coincidencia | Las políticas se evalúan por prioridad y se detiene en la primera coincidencia, salvo bandera "continuar". | 01, 02, 04, 11, 14 |
| RN-05 | Orden de ejecución de acciones | Las acciones se ejecutan en el orden configurado; la copia de mensajes se toma antes de removerlos. | 02, 03, 04, 11 |
| RN-06 | Antirrebote por usuario | Un usuario no recibe la misma acción más de una vez dentro de la ventana de antirrebote. | 02, 16 |
| RN-07 | Descarte previo de exentos | Un sujeto exento se descarta antes de evaluar cualquier regla; nunca es alcanzado por una acción. | 01, 02, 04, 15 |
| RN-08 | Identidad de los snowflakes | Todo identificador de plataforma es un snowflake tratado y almacenado como texto; el servidor se identifica por su snowflake. | 10, 15 |
| RN-09 | El modo simulación no ejecuta | Una política en simulación registra la acción pero no la ejecuta; ante modo indefinido se asume simulación (modo seguro). | 02, 04, 05, 11, 14 |
| RN-10 | Configuración por descriptor | Cada parámetro tiene un descriptor único (default, límites, leyenda, ejemplos); solo acepta valores dentro de límites. | 01, 11, 16 |
| RN-11 | Integridad de la evidencia del incidente | Todo incidente conserva la copia de mensajes disparadores y canales afectados, tomada antes de la remoción. | 05, 06 |
| RN-12 | Autorización por administrador único | Solo el administrador autenticado (único rol) puede administrar; toda acción administrativa requiere sesión válida. | 06, 07, 08, 09, 10, 11, 12, 15 |
| RN-13 | Resguardo de credenciales del admin | A lo sumo una cuenta de administrador; contraseña con hash robusto, nunca en texto claro, con política mínima de robustez. | 08, 09 |
| RN-14 | Cifrado del token en reposo | El token de bot se almacena cifrado con clave maestra fuera de la base; nunca en claro, solo se descifra en memoria. | 10, 12, 13 |
| RN-15 | Composición de un grupo de reglas | Un grupo tiene al menos una regla y un modo de coincidencia (todas / alguna / al menos N); anidamiento booleano de dos niveles. | 11 |
| RN-16 | Activación condicionada a la prueba | Un servidor no se activa con un faltante bloqueante; solo se habilita sin bloqueantes; las advertencias se registran. | 12, 13 |

## Matriz de trazabilidad condensada (NB → CU → RN → módulo → TC)

> El módulo es la ruta verificada bajo `src/DiscordModeradorBot.Servicio/`. Los `TC-` (casos de
> prueba) se catalogan en [`../03-data/test-data-matrix.md`](../03-data/test-data-matrix.md)
> (referencia; su generación es responsabilidad del set de datos §15).

| NB | CU | RN principales | Módulo(s) de código | TC |
|---|---|---|---|---|
| NB-01 | CU-01 | RN-04, 07, 08, 10 | `Dominio/Conducta` (EvaluadorRafagaDistribuida), `Aplicacion` (MotorDeModeracion) | `../03-data/test-data-matrix.md` |
| NB-01 | CU-02 | RN-01, 04, 05, 06, 07, 09 | `Dominio/Moderacion`, `Aplicacion` (MotorDeModeracion), `Infraestructura/Gateway` | `../03-data/test-data-matrix.md` |
| NB-02 | CU-03 | RN-01, 02, 05 | `Dominio/Moderacion` (Accion/TipoAccion), `Infraestructura/Gateway` | `../03-data/test-data-matrix.md` |
| NB-03 | CU-04 | RN-01, 03, 04, 05, 07, 09 | `Dominio/Contenido` (EvaluadorReglaContenido), `Aplicacion` | `../03-data/test-data-matrix.md` |
| NB-04 | CU-05 | RN-01, 09, 11 | `Dominio/Moderacion` (ReporteIncidente), `Infraestructura/Gateway`, `Infraestructura/Persistencia` | `../03-data/test-data-matrix.md` |
| NB-04 | CU-06 | RN-09, 11, 12 | `Components/Pages`, `Infraestructura/Persistencia` (RepositorioIncidentes) | `../03-data/test-data-matrix.md` |
| NB-04 | CU-07 | RN-01, 12 | `Components/Pages`, `Aplicacion`, `Infraestructura/Gateway` | `../03-data/test-data-matrix.md` |
| NB-05 | CU-08 | RN-12, 13 | `Components/Pages` (ConfiguracionInicial), `Aplicacion` (ServicioAdministrador), `Infraestructura/Seguridad` (Pbkdf2) | `../03-data/test-data-matrix.md` |
| NB-05 | CU-09 | RN-12, 13 | `Program.cs` (endpoints auth), `Aplicacion` (ServicioAdministrador), `Dominio/Administracion` (ControlIntentosAutenticacion) | `../03-data/test-data-matrix.md` |
| NB-05 | CU-10 | RN-08, 12, 14 | `Components/Pages`, `Dominio/Servidores`, `Infraestructura/Seguridad` (CifradoAes), `Infraestructura/Persistencia` | `../03-data/test-data-matrix.md` |
| NB-05 | CU-11 | RN-04, 05, 09, 10, 12, 15 | `Components/Pages`, `Dominio/Moderacion/Reglas`, `Dominio/Configuracion` (Descriptores) | `../03-data/test-data-matrix.md` |
| NB-06 | CU-12 | RN-01, 12, 14, 16 | `Dominio/Servidores` (PruebaConfiguracion), `Aplicacion` (ServicioPruebaConfiguracion), `Infraestructura/Gateway` | `../03-data/test-data-matrix.md` |
| NB-06 | CU-13 | RN-14, 16 | `Aplicacion` (GestorConexionesGateway), `Infraestructura` (GestorConexionesGatewayHostedService) | `../03-data/test-data-matrix.md` |
| NB-07 | CU-14 | RN-04, 09 | `Dominio/Moderacion` (Modo), `Aplicacion` (MotorDeModeracion) | `../03-data/test-data-matrix.md` |
| NB-07 | CU-15 | RN-07, 08, 12 | `Dominio/Exenciones` (Exencion), `Components/Pages`, `Infraestructura/Persistencia` | `../03-data/test-data-matrix.md` |
| NB-07 | CU-16 | RN-06, 10 | `Dominio/Conducta` (EstadoAntirreboteEnMemoria), `Aplicacion` (MotorDeModeracion) | `../03-data/test-data-matrix.md` |

## Trazabilidad a decisiones y controles

- Decisiones de arquitectura que soportan estos requisitos:
  [`../04-decisions/README.md`](../04-decisions/README.md).
- Controles de seguridad derivados de RN-01, RN-12, RN-13, RN-14:
  [`../06-security-compliance/threat-model.md`](../06-security-compliance/threat-model.md) y
  [`../06-security-compliance/data-protection.md`](../06-security-compliance/data-protection.md).
- Operación de CU-08..CU-13:
  [`../07-operations/README.md`](../07-operations/README.md).

## Referencias

- SDD 2.2 (fuente autoritativa de NB/CU/RN/RC): `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/`
- Código (fuente de verdad): `../../../../Discord.Bot.Moderador.Core/src/DiscordModeradorBot.Servicio/`
- Catálogo de decisiones: [`../04-decisions/README.md`](../04-decisions/README.md)
