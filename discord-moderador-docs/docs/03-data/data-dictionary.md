---
doc_id: DATA-DICT-001
doc_type: data-dictionary
title: Diccionario de datos — persistencia del bot moderador
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: "equipo-discordmoderador (propuesto)"
last_review: 2026-07-15
review_cycle_days: 120
audience: [dev, qa, arquitectura, agentes-automaticos]
classification: uso-interno
traces: [RN-08, RC-02, ADR-02, RN-11, RN-13, RN-14, ADR-03, ADR-07, RC-11, RN-15, CU-06, CU-11]
---

# Diccionario de datos — persistencia del bot moderador

> Glosario canónico: [`../../GLOSSARY.md`](../../GLOSSARY.md). Modelo ER como código:
> [`er-diagrams/modelo.dbml`](er-diagrams/modelo.dbml).

## Alcance, origen y gap

Documenta las **13 tablas físicas** del contexto de persistencia embebida del servicio
`DiscordModeradorBot.Servicio`. La **fuente de verdad es el código EF Core**:
`Infraestructura/Persistencia/ContextoPersistencia.cs` (`OnModelCreating`) y
`Infraestructura/Persistencia/Entidades/*.cs`, más las migraciones `MIG0001..MIG0009`. El modelo
lógico `SDD2.2D/docs/05_arquitectura_tecnica/modelo-datos-logico_v1.0.md` se cita como fuente
secundaria.

**Gap declarado — documentado desde el esquema EF, sin acceso a datos.** No se provee cadena de
conexión ni se accedió a una base viva. Tipos, nulabilidad, defaults, índices y relaciones se leen
del mapeo EF; **no** se muestrearon volúmenes, distribuciones ni valores reales. El motor es SQLite
embebido en modo WAL (ADR-02).

### Convenciones de tipos físicos (SQLite vía EF)

- **Snowflakes** (servidor, usuario, canal, mensaje, rol) se almacenan como **TEXTO** (RN-08, RC-02).
- **`DateTimeOffset`** se persiste como **ticks UTC (entero `long`/`bigint`)** mediante
  `ValueConverter` (ADR-02); ordena y round-trip sin pérdida. Aplica a `CreadoEn`, `Instante`,
  `ReversionFecha`.
- **Claves primarias** `Id`: entero autoincremental por tabla.
- **Booleanos**: enteros 0/1 (mapeo EF de `bool`).

### Divergencias esquema-código vs. modelo lógico (fuente de verdad = código)

1. **No existe tabla `CanalDeSalida`.** El canal de salida está **desnormalizado en `Servidor`**
   (`SnowflakeCanalSalida`, `PropositoCanalSalida`). La 13.ª tabla física es `ParametrosServidor`,
   ausente del modelo lógico.
2. **`Incidente` no usa FK enteras a `Servidor`/`Evento`.** Guarda `ServidorId` y `UsuarioId` como
   **snowflake TEXTO** y la política por **`NombrePolitica` TEXTO** (walking skeleton R3). Son
   vínculos lógicos, no restricciones referenciales.
3. **No hay restricciones `CHECK` en la BD.** Los `ck_*` del modelo lógico (dominios de enum,
   coherencia modo-resultado, rango de ventana) **no se emiten**: los enums se guardan como texto
   libre y el dominio de valores lo valida el **Dominio**, no la BD. Ver §15 (test-data-matrix) para
   los huecos de cobertura que esto abre.
4. **FK forzadas por la BD (cascada):** solo `Incidente→MensajeAccionado/CanalAfectado`,
   `GrupoDeReglas→GrupoRegla`, `Evento→EventoGrupo/Accion`. Las demás (por snowflake,
   `ReversionAutorId`, `ReglaContenidoId`, `EventoGrupo.GrupoDeReglasId`) son **FK lógicas** sin
   `HasForeignKey` configurado.
5. Los índices únicos `ux_gruporegla_par` / `ux_eventogrupo_par` del modelo lógico **no existen** en
   el código; los índices reales sobre esas tablas son **no únicos** por FK.

### PII y datos sensibles (resumen)

| Columna | Tabla | Clasificación | Tratamiento |
| --- | --- | --- | --- |
| `TokenCifrado` | Servidor | Secreto | Token de bot **cifrado AES** en reposo, clave maestra externa; nunca en claro (RN-14, ADR-07) |
| `ResguardoPassword` | Administrador | Secreto | **Hash PHC PBKDF2**, no reversible (RN-13, ADR-03) |
| `ContenidoCopiado` | MensajeAccionado | Dato personal | Copia del texto del mensaje del usuario, evidencia (RN-11) |
| `NombreUsuario` | MensajeAccionado | Dato personal | Nombre legible del autor al momento del incidente (CU-06) |
| `NombreCanal` | MensajeAccionado | Dato personal (bajo) | Nombre legible del canal (CU-06) |
| `UsuarioId` | Incidente | Identificador personal | Snowflake del usuario accionado |

### Retención

La evidencia de moderación (`Incidente` y sus hijas `MensajeAccionado` / `CanalAfectado`) es dato
personal conservado como registro de la intervención. **El código no implementa purga ni TTL
automáticos**: la conservación es indefinida hasta borrado manual del incidente, que **cascadea** a
la evidencia (RN-11). La política de minimización y retención se rige por ADR-06 (Ley 25.326); ver
[`access-policies.md`](access-policies.md). Gap: no hay mecanismo de expiración en el esquema.

---

## 1. Servidor

Servidor de Discord moderado, con su credencial cifrada, estado y canal de salida (modelo-datos-logico §2.2, §2.3). Tabla física `Servidor`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| SnowflakeServidor | TEXTO | No | Identificador | Snowflake del servidor; único (`ux_servidor_snowflake`) | RN-08, RC-02 |
| TokenCifrado | TEXTO | No | **Secreto** | Token de bot cifrado AES en reposo; nunca en claro | RN-14, ADR-07 |
| EstadoConexion | TEXTO | No | — | Enum `{desconectado, conectado}`; default `desconectado` | ServidorRegistrado.cs |
| EstadoActivacion | TEXTO | No | — | Enum `{inactivo, activo}`; default `inactivo`; activo solo tras prueba superada | RC-08, RN-16 |
| NombreDescriptivo | TEXTO | Sí | — | Etiqueta para el panel | §2.2 |
| CreadoEn | bigint (ticks UTC) | No | — | Instante de alta | ADR-02 |
| SnowflakeCanalSalida | TEXTO | Sí | Identificador | Canal de salida designado para reportes; null si no designó | CU-05 |
| PropositoCanalSalida | TEXTO | Sí | — | Propósito lógico del canal de salida | CU-05 |

**Índices:** `ux_servidor_snowflake` (único) sobre `SnowflakeServidor`.

## 2. Incidente

Registro persistente de una intervención de moderación; unidad de trazabilidad (modelo-datos-logico §2.11, RN-11).

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| ServidorId | TEXTO | No | Identificador | Snowflake del servidor (FK lógica) | RN-08 |
| UsuarioId | TEXTO | No | **Dato personal** | Snowflake del usuario accionado | RN-08 |
| NombrePolitica | TEXTO | No | — | Nombre lógico de la política/evento que disparó | §2.11 |
| Modo | TEXTO | No | — | Enum `{simulacion, ejecucion}` | RC-10, RN-09 |
| Accion | TEXTO | No | — | Acción representativa registrada | §2.11 |
| Resultado | TEXTO | No | — | Enum `{NoAccionable, Simulada, Ejecutada, Fallida}` | RN-01 |
| Instante | bigint (ticks UTC) | No | — | Momento del disparo | ADR-02 |
| ReversionAutorId | integer | Sí | — | FK lógica a `Administrador`; null si no hubo reversión | CU-07 |
| ReversionFecha | bigint (ticks UTC) | Sí | — | Fecha del desbaneo; null si no hubo reversión | CU-07 |

**Índices:** `ix_incidente_servidor_fecha` sobre `(ServidorId, Instante)`.
**Cascada:** al borrar el incidente se borran sus `MensajeAccionado` y `CanalAfectado` (RN-11).

## 3. MensajeAccionado

Copia de un mensaje involucrado en un incidente, tomada antes de cualquier remoción (modelo-datos-logico §2.12, RN-11). Tabla hija de `Incidente`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| IncidenteId | integer (FK) | No | — | FK a `Incidente` (cascada) | RC-01, RN-11 |
| SnowflakeMensaje | TEXTO | No | Identificador | Snowflake del mensaje | RN-08 |
| SnowflakeCanal | TEXTO | No | Identificador | Snowflake del canal del mensaje | RN-08 |
| ContenidoCopiado | TEXTO | No | **Dato personal** | Copia del texto del mensaje del usuario (evidencia) | RN-11, RN-05 |
| NombreCanal | TEXTO | No | Dato personal (bajo) | Nombre legible del canal; default `""` para filas previas a su captura | CU-06 |
| NombreUsuario | TEXTO | No | **Dato personal** | Nombre legible del autor; default `""` para filas previas | CU-06 |

**Índices:** `ix_mensaje_accionado_incidente` sobre `IncidenteId`.

## 4. CanalAfectado

Canal en el que el incidente tuvo efecto (modelo-datos-logico §2.13, RN-11). Tabla hija de `Incidente`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| IncidenteId | integer (FK) | No | — | FK a `Incidente` (cascada) | RC-01, RN-11 |
| SnowflakeCanal | TEXTO | No | Identificador | Snowflake del canal afectado | RN-08 |

**Índices:** `ix_canal_afectado_incidente` sobre `IncidenteId`.

## 5. Regla

Regla de contenido (predicado sin estado sobre el texto del mensaje) (modelo-datos-logico §2.5, CU-04, R3). Tabla física `Regla`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| SnowflakeServidor | TEXTO | No | Identificador | Snowflake del servidor (FK lógica) | RN-08 |
| Nombre | TEXTO | No | — | Etiqueta de la regla | §2.5 |
| TipoCriterio | TEXTO | No | — | Enum `{ExpresionRegular, PalabrasClave}`; R3 solo `ExpresionRegular` | RC-09 |
| Criterio | TEXTO | No | — | Criterio serializado ya validado (p. ej. la regex) | RN-03 |
| SensibleAMayusculas | boolean | No | — | Si la coincidencia distingue mayúsculas/minúsculas | CU-04 |
| NombrePolitica | TEXTO | No | — | Vínculo a la política por nombre lógico (walking skeleton R3) | §2.5 |

**Índices:** `ix_regla_servidor` sobre `SnowflakeServidor`.

## 6. Administrador

Cuenta única que opera el sistema (modelo-datos-logico §2.1, RC-06, RN-13).

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| IdentificadorCuenta | TEXTO | No | Identificador | Identificador de cuenta; único (`ux_administrador_cuenta`) | RC-06 |
| ResguardoPassword | TEXTO | No | **Secreto** | Hash PHC PBKDF2 de la contraseña; nunca en claro | RN-13, ADR-03 |
| CreadoEn | bigint (ticks UTC) | No | — | Instante de alta (primer ingreso) | ADR-02, CU-08 |

**Índices:** `ux_administrador_cuenta` (único). La unicidad de fila (a lo sumo un administrador, RC-06) la resguarda el alta del servicio, no un CHECK.

## 7. Exencion

Sujeto de confianza (rol, usuario o canal) excluido de la moderación (modelo-datos-logico §2.4, CU-15, RN-07).

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| SnowflakeServidor | TEXTO | No | Identificador | Snowflake del servidor (FK lógica) | RN-08 |
| TipoSujeto | TEXTO | No | — | Enum `{rol, usuario, canal}` | ck_exencion_tipo |
| SnowflakeSujeto | TEXTO | No | Identificador | Snowflake del rol/usuario/canal exento | RN-08, RC-02 |

**Índices:** `ix_exencion_servidor` sobre `SnowflakeServidor`; `ux_exencion_sujeto` (único) sobre `(SnowflakeServidor, TipoSujeto, SnowflakeSujeto)`.

## 8. GrupoDeReglas

Conjunto de reglas combinadas por un modo de coincidencia (RN-15, RC-04). Tabla física `GrupoDeReglas`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| SnowflakeServidor | TEXTO | No | Identificador | Snowflake del servidor (FK lógica) | RN-08 |
| Nombre | TEXTO | No | — | Etiqueta del grupo | CU-11 |
| ModoCoincidencia | TEXTO | No | — | Enum `{todas, alguna, almenosn}` | RN-15 |
| MinimoCoincidencias | integer | Sí | — | N mínimo; solo en modo `almenosn` (1..cantidad de reglas) | RN-15 |

**Índices:** `ix_grupo_servidor` sobre `SnowflakeServidor`.
**Cascada:** al borrar el grupo se borran sus `GrupoRegla`. La composición mínima (≥1 regla, RC-04) la garantiza el Dominio, no la BD.

## 9. GrupoRegla

Relación M:M entre grupo y regla (RC-03). Tabla física `GrupoRegla`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| GrupoDeReglasId | integer (FK) | No | — | FK a `GrupoDeReglas` (cascada) | RC-03 |
| ClaseRegla | TEXTO | No | — | Enum `{contenido, conducta}` | GrupoReglaEntidad.cs |
| ReglaContenidoId | integer | Sí | — | FK lógica a `Regla` si `ClaseRegla=contenido`; null si conducta | §2.5 |
| ClaveReglaConducta | TEXTO | Sí | — | Clave lógica del catálogo si `ClaseRegla=conducta` (p. ej. `rafaga-distribuida`) | CU-01 |

**Índices:** `ix_gruporegla_grupo` sobre `GrupoDeReglasId`.

## 10. Evento

Política de moderación con prioridad, bandera continuar y modo (R7, modelo-datos-logico §2.8; CU-11, RN-04, RN-09). Tabla física `Evento`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| SnowflakeServidor | TEXTO | No | Identificador | Snowflake del servidor (FK lógica) | RN-08 |
| Nombre | TEXTO | No | — | Etiqueta del evento | CU-11 |
| Prioridad | integer | No | — | Orden de evaluación; menor valor = mayor prioridad | RN-04, RC-05 |
| Continuar | boolean | No | — | Permite seguir evaluando eventos de menor prioridad tras coincidir | RN-04 |
| Modo | TEXTO | No | — | Enum `{simulacion, ejecucion}`; default `simulacion` | RN-09 |
| ModoCombinacionGrupos | TEXTO | No | — | Enum `{todos, alguno}`; default `todos` | RN-15 |

**Índices:** `ix_evento_servidor_prioridad` sobre `(SnowflakeServidor, Prioridad)`.
**Cascada:** al borrar el evento se borran sus `EventoGrupo` y `Accion`.

## 11. EventoGrupo

Relación M:M entre evento y grupo (RC-03); segundo nivel del anidamiento booleano (RN-15). Tabla física `EventoGrupo`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| EventoId | integer (FK) | No | — | FK a `Evento` (cascada) | RC-03 |
| GrupoDeReglasId | integer | No | — | FK lógica a `GrupoDeReglas` | RC-03 |

**Índices:** `ix_eventogrupo_evento` sobre `EventoId`.

## 12. Accion

Acción de moderación de un evento, con orden de ejecución y parámetros por tipo (modelo-datos-logico §2.10, RN-05). Tabla física `Accion`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| EventoId | integer (FK) | No | — | FK a `Evento` (cascada) | — |
| Tipo | TEXTO | No | — | Enum de `TipoAccion` (ver dominios) | §2.10 |
| OrdenEjecucion | integer | No | — | Orden dentro del evento | RN-05, RC-05 |
| VentanaBorradoDias | integer | Sí | — | Ventana de borrado del baneo, 0..7 días; null si no aplica | RC-11, RN-02 |
| DuracionTimeoutMinutos | integer | Sí | — | Duración del timeout; null si no aplica (R6) | — |
| RolObjetivo | TEXTO | Sí | Identificador | Snowflake del rol objetivo para asignar/quitar rol; null si no aplica | RN-08 |

**Índices:** `ix_accion_evento_orden` sobre `(EventoId, OrdenEjecucion)`.

## 13. ParametrosServidor

Parámetros de moderación dirigidos por descriptor, por servidor; una fila por servidor (CU-11, RN-10). Tabla física `ParametrosServidor`.

| Columna | Tipo | Nulo | PII/Sensible | Descripción | Traza |
| --- | --- | --- | --- | --- | --- |
| Id | integer (PK, auto) | No | — | Clave primaria | ContextoPersistencia |
| SnowflakeServidor | TEXTO | No | Identificador | Snowflake del servidor; único (`ux_parametros_servidor`) | RN-08 |
| UmbralCanalesDistintos | integer | Sí | — | Umbral de canales distintos de la ráfaga; null = default del descriptor | CU-01, RN-10 |
| VentanaDeteccionSegundos | real | Sí | — | Ventana de detección de la ráfaga en segundos; null = default | CU-01, RN-10 |
| VentanaAntirreboteSegundos | real | Sí | — | Ventana de antirrebote por usuario en segundos; null = default | CU-16, RN-06 |

**Índices:** `ux_parametros_servidor` (único) sobre `SnowflakeServidor`. Semántica de null: "usar el valor por defecto del descriptor" (un servidor sin fila usa todos los defaults).

---

## Dominios de valores de enums (semántica que el esquema no captura)

Los enums se persisten como **texto** sin `CHECK`. La forma canónica es la del código de Dominio.

| Enum (columna) | Valores | Semántica | Fuente |
| --- | --- | --- | --- |
| `TipoAccion` (`Accion.Tipo`) | Reportar, Banear, BaneoConBorradoRetroactivo, Desbanear, Timeout, Expulsar, AsignarRol, QuitarRol, ReportarACanalPrivado | Tipo de operación de moderación; parámetros propios según el tipo | Dominio/Moderacion/Accion.cs |
| `Modo` (`Evento.Modo`, `Incidente.Modo`) | simulacion, ejecucion | Simulación registra sin ejecutar (RN-09); default seguro simulación | Dominio/Moderacion/Modo.cs |
| `ResultadoModeracion` (`Incidente.Resultado`) | NoAccionable, Simulada, Ejecutada, Fallida | Resultado registrable del incidente | Dominio/Moderacion/ResultadoModeracion.cs |
| `TipoCriterioContenido` (`Regla.TipoCriterio`) | ExpresionRegular, PalabrasClave | Clase del criterio de contenido; R3 solo ExpresionRegular | Dominio/Contenido/TipoCriterioContenido.cs |
| `TipoSujetoExento` (`Exencion.TipoSujeto`) | rol, usuario, canal | Tipo de sujeto de confianza exento | Dominio/Exenciones/Exencion.cs |
| `ModoCoincidencia` (`GrupoDeReglas.ModoCoincidencia`) | todas, alguna, almenosn | Combinación booleana de reglas del grupo (AND / OR / umbral N) | Dominio/Moderacion/Reglas/GrupoDeReglas.cs |
| `ModoCombinacionGrupos` (`Evento.ModoCombinacionGrupos`) | todos, alguno | Combinación booleana de grupos del evento (AND / OR) | Dominio/Moderacion/Reglas/ComposicionPolitica.cs |
| `ClaseRegla` (`GrupoRegla.ClaseRegla`) | contenido, conducta | Clase de la regla asociada al grupo | Dominio/GrupoReglaEntidad.cs |
| `EstadoConexion` (`Servidor.EstadoConexion`) | desconectado, conectado | Estado de la conexión al gateway | Dominio/Servidores/ServidorRegistrado.cs |
| `EstadoActivacion` (`Servidor.EstadoActivacion`) | inactivo, activo | Activo solo tras superar la prueba de configuración (RC-08, RN-16) | Dominio/Servidores/ServidorRegistrado.cs |

> Advertencia de integridad: como no hay `CHECK`, la BD aceptaría un valor de enum fuera de dominio,
> una `VentanaBorradoDias` fuera de 0..7, o un incidente `Modo=simulacion` con `Resultado=Ejecutada`.
> Estos casos se cubren como pruebas negativas en [`test-data-matrix.md`](test-data-matrix.md).
