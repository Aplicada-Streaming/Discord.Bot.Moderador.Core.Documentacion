# 10 · Glosario (condensado)

> **Propósito.** Referencia rápida del vocabulario del dominio para arrancar contexto sin abrir el
> código. **Este índice es una versión CONDENSADA.** El glosario canónico completo —con alias, trazas
> y clasificación— vive en
> [`../../discord-moderador-docs/GLOSSARY.md`](../../discord-moderador-docs/GLOSSARY.md); ante cualquier
> duda o para el detalle normativo, esa es la fuente. Aquí solo se listan los términos con una
> definición de una línea y el índice de esta ia-db donde se profundizan.

## Dominio de moderación

| Término | Definición breve | Profundizar en |
|---|---|---|
| **Ráfaga distribuida** | Un usuario publica en varios **canales distintos** en una ventana corta (discrimina canales, no volumen) | [`02`](02_dominio.md#conducta-dominioconducta-adr-09) |
| **Incidente** | Registro persistente de un disparo de política (servidor, usuario, modo, acción, resultado, evidencia, instante) | [`02`](02_dominio.md), [`06`](06_modelo-de-datos.md) |
| **Política / Evento** | Regla configurada que, al coincidir, dispara acciones ordenadas; "Evento" es su nombre persistido | [`02`](02_dominio.md#política) |
| **Grupo de reglas** | Conjunto de reglas combinadas por modo de coincidencia (todas/alguna/mínimo N) | [`02`](02_dominio.md#composición-y-grupos-de-reglas-r7-rn-15) |
| **Composición** | Combinación de grupos (AND/OR) que forma la condición de disparo de una política (nivel 2) | [`02`](02_dominio.md#composición-y-grupos-de-reglas-r7-rn-15) |
| **Regla de contenido** | Predicado **sin estado** sobre el texto (regex) con tope de tiempo anti-ReDoS | [`02`](02_dominio.md#contenido-dominiocontenido) |
| **Regla de conducta** | Predicado **con estado** sobre la actividad acumulada del usuario (ráfaga) | [`02`](02_dominio.md#conducta-dominioconducta-adr-09) |
| **Acción** | Efecto sobre el emisor/servidor con orden de ejecución (banear, borrado retroactivo, timeout, expulsar, rol, reportar) | [`02`](02_dominio.md#acción-y-tipos) |
| **Borrado retroactivo** | Purga de mensajes recientes al banear, acotada a **7 días** (tope de plataforma) | [`02`](02_dominio.md#configuración-por-descriptores-dominioconfiguracion-adr-12-rn-10) |
| **Exención** | Sujeto (usuario/rol/canal) cuyos mensajes se descartan **antes** de toda evaluación | [`02`](02_dominio.md#exenciones-dominioexenciones) |
| **Antirrebote** | Supresión de acciones repetidas sobre un usuario ya accionado dentro de una ventana; estado **en memoria** | [`02`](02_dominio.md#conducta-dominioconducta-adr-09) |
| **Modo** | `Simulacion` (registra sin ejecutar) vs `Ejecucion` (acción real). Default seguro: simulación | [`02`](02_dominio.md#incidente-modo-y-resultado) |
| **Canal de salida** | Canal privado donde se publican los reportes de incidentes | [`02`](02_dominio.md#servidores-y-administración) |
| **Resultado de moderación** | `Ejecutada` / `Simulada` / `NoAccionable` / `Fallida`; el "peor" manda | [`02`](02_dominio.md#incidente-modo-y-resultado) |
| **No accionable** | La contención no se aplicó por jerarquía superior o permisos; el incidente igual se registra y reporta | [`01`](01_arquitectura-y-pipeline.md#manejo-de-errores-del-pipeline-adr-08-rn-01) |
| **Descriptor de parámetro** | Fuente única de default/límites/leyenda de un parámetro configurable | [`02`](02_dominio.md#configuración-por-descriptores-dominioconfiguracion-adr-12-rn-10) |
| **Prueba de configuración** | Smoke test que un servidor debe superar antes de activarse para moderar | [`03`](03_aplicacion-servicios.md) |

## Plataforma y técnicos

| Término | Definición breve | Profundizar en |
|---|---|---|
| **Snowflake** | Id de 64 bits de Discord, almacenado como **texto** | [`02`](02_dominio.md#snowflake-y-mensaje-entrante), [`06`](06_modelo-de-datos.md) |
| **Gateway** | Puerto de salida a Discord con dos adaptadores: Simulado y Discord | [`04`](04_infraestructura.md#gateway-infraestructuragateway) |
| **Modo Simulado** | Adaptador por defecto sin red/token; walking skeleton inyecta escenarios demo | [`04`](04_infraestructura.md) |
| **Modo Discord** | Adaptador real (Discord.Net): una conexión por servidor | [`04`](04_infraestructura.md) |
| **Walking skeleton** | Hosted service que en modo Simulado ejercita el pipeline con mensajes demo | [`04`](04_infraestructura.md#hosted-services-y-wiring) |
| **Gestor de conexiones** | Hosted service que en modo Discord abre/mantiene una conexión por servidor activo | [`03`](03_aplicacion-servicios.md#cargador-de-políticas-y-gestor-de-conexiones), [`04`](04_infraestructura.md) |
| **Administrador único** | Único rol/cuenta; cookie auth + PBKDF2/PHC | [`09`](09_configuracion-y-seguridad.md#autenticación-adr-03-rn1213) |
| **Clave maestra** | `DISCORDMODERADOR_CLAVE_MAESTRA`, cifra tokens en reposo (AES-GCM) | [`09`](09_configuracion-y-seguridad.md#secretos-y-clave-maestra) |
| **First-run** | Alta inicial de credenciales del admin (`/configuracion-inicial`) | [`05`](05_panel-web.md) |
| **Rebanada vertical (R1..R7)** | Unidad de entrega: corte funcional de punta a punta | [`00`](00_MASTER-INDEX.md#rebanadas-verticales-r1r7-y-casos-de-uso) |

## Prefijos de identificadores estables

Acuñados por la documentación SDD 2.2 (`SDD2.2D/docs/`); se referencian por id, no se copian.

| Prefijo | Significado | Dónde |
|---|---|---|
| `NB-` | Necesidad de negocio | `SDD2.2D/docs/01_necesidades_negocio/` |
| `CU-` | Caso de uso | `SDD2.2D/docs/02_especificacion_funcional/casos-de-uso/` |
| `RN-` | Regla de negocio | `SDD2.2D/docs/02_especificacion_funcional/reglas-de-negocio/` |
| `RC-` | Regla conceptual de modelo | `SDD2.2D/docs/02_especificacion_funcional/modelo-datos/` |
| `ADR-` | Decisión de arquitectura | `SDD2.2D/docs/05_arquitectura_tecnica/adrs/` |
| `TC-` | Caso de prueba derivado del modelo | documentación de pruebas |

> Para el detalle completo de cada término (alias, traza al archivo de código, clasificación), abrir el
> glosario canónico [`../../discord-moderador-docs/GLOSSARY.md`](../../discord-moderador-docs/GLOSSARY.md).
</content>
