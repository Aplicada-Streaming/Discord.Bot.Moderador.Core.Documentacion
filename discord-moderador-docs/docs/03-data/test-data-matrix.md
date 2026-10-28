---
doc_id: DATA-TDM-001
doc_type: test-data-matrix
title: Matriz de datos de prueba — derivación desde el modelo de datos (§15)
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: "equipo-discordmoderador (propuesto)"
last_review: 2026-07-15
review_cycle_days: 120
audience: [dev, qa, arquitectura, agentes-automaticos]
classification: uso-interno
traces: [RN-08, RC-02, RN-09, RC-10, RN-11, RC-01, RN-13, RC-06, CU-15, RN-15, RC-04, RN-04, RN-05, RN-02, RC-11, RN-10, RN-14]
---

# Matriz de datos de prueba — derivación desde el modelo (§15)

> Glosario: [`../../GLOSSARY.md`](../../GLOSSARY.md). Modelo: [`data-dictionary.md`](data-dictionary.md),
> [`er-diagrams/modelo.dbml`](er-diagrams/modelo.dbml).

## Método de derivación

Cada `TC-` nace de una propiedad del modelo de datos (fuente de verdad = código EF):
por **columna con dominio/constraint** → clases de equivalencia + valores borde; por **índice
único** → caso de colisión; por **FK/cascada** → integridad referencial; por **enum** → valor de
dominio (válido e inválido); por **regla de negocio (RN/RC)** → caso negativo. Cada TC traza a un
RN/RC/CU.

**Nota transversal de gaps:** el código **no emite `CHECK`**, por lo que la BD **aceptaría** valores
de enum fuera de dominio, `VentanaBorradoDias` fuera de 0..7 y combinaciones modo/resultado
incoherentes. Los TC marcados **[gap-BD]** verifican que la **validación vive en el Dominio/servicio**
(no en la BD): el esperado es rechazo/normalización por la capa de aplicación, y se anota el hueco de
la base. Los TC marcados **[FK-lógica]** dependen de FK no forzadas por la BD (por snowflake o Id sin
`HasForeignKey`): su integridad la sostiene la aplicación.

## 1. Servidor

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
| --- | --- | --- | --- | --- | --- |
| TC-01 | `ux_servidor_snowflake` | Colisión (unicidad) | Dos servidores con el mismo `SnowflakeServidor` | Segundo insert rechazado por índice único | RN-08, RC-02 |
| TC-02 | `EstadoConexion` enum | Valor de dominio | `conectado`; `desconectado` | Ambos aceptados | ServidorRegistrado.cs |
| TC-03 | `EstadoActivacion` enum | Valor de dominio inválido **[gap-BD]** | `activo-total` (fuera de `{inactivo,activo}`) | Rechazo del servicio; la BD lo aceptaría (gap) | RC-08, RN-16 |
| TC-04 | `TokenCifrado` sensible | Confidencialidad | Registrar token de bot | Valor en reposo ≠ token en claro; nunca en claro | RN-14, ADR-07 |
| TC-05 | `NombreDescriptivo` nulable | Clase de equivalencia (null) | Alta sin nombre descriptivo | Insert aceptado con `null` | data-dictionary §1 |
| TC-06 | `CreadoEn` ticks UTC | Round-trip | Guardar y releer `CreadoEn` | Igual instante, sin pérdida | ADR-02 |

## 2. Incidente (+ hijas)

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
| --- | --- | --- | --- | --- | --- |
| TC-07 | `Modo` enum | Valor de dominio | `simulacion`; `ejecucion` | Ambos aceptados | RC-10, RN-09 |
| TC-08 | Coherencia modo↔resultado | Caso negativo **[gap-BD]** | `Modo=simulacion` + `Resultado=Ejecutada` | Rechazo del dominio (simulación no ejecuta); la BD lo aceptaría (gap `ck_incidente_simulacion_no_ejecuta` ausente) | RN-09, RC-10 |
| TC-09 | `Resultado` enum | Valor de dominio | `NoAccionable`,`Simulada`,`Ejecutada`,`Fallida` | Los cuatro válidos; un quinto valor lo rechaza el dominio | RN-01 |
| TC-10 | FK cascada Incidente→hijas | Integridad referencial | Incidente con N `MensajeAccionado` + M `CanalAfectado`; borrar incidente | Se borran todas las hijas en la misma unidad | RN-11 |
| TC-11 | `ReversionAutorId`/`ReversionFecha` nulables | Clase de equivalencia | (a) sin reversión: ambos null; (b) con reversión: autor+fecha | (a) y (b) aceptados; parciales inconsistentes los valida CU-07 | CU-07 |
| TC-12 | `ix_incidente_servidor_fecha` | Cobertura de índice | Varios incidentes del servidor; consultar por fecha | Orden determinista por `(ServidorId, Instante)` | CU-06, RN-04 |
| TC-13 | `Instante`/`ReversionFecha` ticks | Round-trip nullable | Guardar/leer fechas (una null) | Round-trip exacto; null preservado | ADR-02 |

## 3. MensajeAccionado / CanalAfectado

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
| --- | --- | --- | --- | --- | --- |
| TC-14 | `ContenidoCopiado` NOT NULL | Caso negativo | Insertar mensaje accionado sin contenido | Rechazo (columna requerida) | RN-11, RN-05 |
| TC-15 | `NombreCanal`/`NombreUsuario` default `""` | Valor borde | Insertar sin nombres (filas previas a captura) | Se guardan como `""`, no null | CU-06 |
| TC-16 | FK `IncidenteId` **[FK-lógica/PRAGMA]** | Integridad referencial | Insertar hija con `IncidenteId` inexistente | Rechazo con FK habilitadas (PRAGMA on); si off, huérfano (gap) | RC-01, RN-11 |
| TC-17 | Evidencia PII | Minimización | Revisar evidencia | Solo el contenido necesario; acceso solo admin autenticado | RN-11, ADR-06 |

## 4. Regla (contenido)

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
| --- | --- | --- | --- | --- | --- |
| TC-18 | `TipoCriterio` enum | Valor de dominio | `ExpresionRegular` (R3); `PalabrasClave` (extensión) | `ExpresionRegular` aceptado y evaluable; `PalabrasClave` declarado | RC-09 |
| TC-19 | `Criterio` validez | Caso negativo | Regex inválida al configurar la regla | Rechazo antes de persistir; nunca se persiste inválido | RN-03 |
| TC-20 | `SensibleAMayusculas` bool | Clase de equivalencia | `true`; `false` sobre el mismo texto | Coincidencia distinta según el flag | CU-04 |
| TC-21 | `ix_regla_servidor` | Cobertura de índice | Varias reglas de un servidor | Recuperación por `SnowflakeServidor` | CU-04 |

## 5. Administrador

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
| --- | --- | --- | --- | --- | --- |
| TC-22 | `ux_administrador_cuenta` | Colisión (unicidad) | Dos administradores con igual `IdentificadorCuenta` | Segundo insert rechazado | RC-06 |
| TC-23 | Unicidad de fila (RC-06) **[gap-BD]** | Caso negativo | Alta de un segundo administrador (cuenta distinta) | Rechazo del servicio (a lo sumo una fila); la BD no lo impide (sin `ck_admin_unico`) | RC-06, RN-12 |
| TC-24 | `ResguardoPassword` sensible | Confidencialidad | Alta con contraseña | Se guarda hash PHC (PBKDF2); nunca en claro; no reversible | RN-13, ADR-03 |

## 6. Exencion

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
| --- | --- | --- | --- | --- | --- |
| TC-25 | `ux_exencion_sujeto` | Colisión (unicidad) | Repetir `(servidor, tipo, sujeto)` | Rechazo `EXENCION_DUPLICADA` | CU-15 |
| TC-26 | `TipoSujeto` enum | Valor de dominio | `rol`; `usuario`; `canal` | Los tres aceptados; un cuarto lo rechaza el dominio | Exencion.cs |
| TC-27 | Unicidad compuesta | Valor borde (no colisión) | Mismo `sujeto`, distinto `TipoSujeto` | Ambas aceptadas (la clave es la terna) | CU-15 |
| TC-28 | `ix_exencion_servidor` | Cobertura de índice | Varias exenciones de un servidor | Descarte de exentos por `SnowflakeServidor` | RN-07 |

## 7. GrupoDeReglas / GrupoRegla

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
| --- | --- | --- | --- | --- | --- |
| TC-29 | `ModoCoincidencia` enum | Valor de dominio | `todas`; `alguna`; `almenosn` | Los tres aceptados por el dominio | RN-15 |
| TC-30 | `MinimoCoincidencias` en `almenosn` | Valor borde | `N=0`; `N=1`; `N=cantidad`; `N=cantidad+1` | `1..cantidad` válidos; `0` y `cantidad+1` → `CONFIG_GRUPO_N_INVALIDO` | RN-15 |
| TC-31 | Composición mínima (RC-04) | Caso negativo | Grupo con 0 reglas | `CONFIG_GRUPO_SIN_REGLAS` | RC-04, RN-15 |
| TC-32 | FK cascada Grupo→GrupoRegla | Integridad referencial | Borrar un grupo con reglas asociadas | Se borran sus `GrupoRegla` | RC-03 |
| TC-33 | Consistencia `ClaseRegla` | Caso negativo **[gap-BD]** | `contenido` con `ReglaContenidoId=null`; o `conducta` con `ClaveReglaConducta=null` | Rechazo del servicio; la BD no lo valida (sin CHECK) | GrupoReglaEntidad.cs |

## 8. Evento / EventoGrupo / Accion

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
| --- | --- | --- | --- | --- | --- |
| TC-34 | `Modo` default | Valor por defecto | Alta de evento sin especificar modo | Queda `simulacion` (seguro por defecto) | RN-09, RC-10 |
| TC-35 | `ix_evento_servidor_prioridad` | Cobertura de índice | Eventos con prioridades 0,1,2 | Evaluación ordenada por prioridad (menor = antes) | RN-04, RC-05 |
| TC-36 | `ModoCombinacionGrupos` enum | Valor de dominio | `todos`; `alguno` | Ambos aceptados | RN-15 |
| TC-37 | FK cascada Evento→hijas | Integridad referencial | Borrar evento con grupos y acciones | Se borran `EventoGrupo` y `Accion` | RC-03, RN-05 |
| TC-38 | `VentanaBorradoDias` rango | Valor borde | `0`; `7`; `8` | `0` y `7` válidos; `8` se **topa a 7** (no se rechaza) con `BORRADO_VENTANA_FUERA_DE_RANGO` | RC-11, RN-02 |
| TC-39 | `VentanaBorradoDias` negativa | Valor borde/negativo **[gap-BD]** | `-1` | Normalizado por el dominio; la BD lo aceptaría (sin CHECK) | RC-11, RN-02 |
| TC-40 | `Accion.Tipo` enum | Valor de dominio | `Banear`,`Timeout`,`AsignarRol`,`ReportarACanalPrivado`,…; un valor inexistente | Los del catálogo válidos; el inexistente lo rechaza el dominio | Accion.cs |
| TC-41 | `ix_accion_evento_orden` | Cobertura de índice | Acciones con orden 0,1,2 | Ejecución determinista por `OrdenEjecucion` | RN-05 |

## 9. ParametrosServidor

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
| --- | --- | --- | --- | --- | --- |
| TC-42 | `ux_parametros_servidor` | Colisión (unicidad) | Dos filas para el mismo `SnowflakeServidor` | Segunda rechazada (una fila por servidor) | CU-11 |
| TC-43 | Columnas nulables | Clase de equivalencia (null) | Fila con las tres columnas null | Aceptada; se usan los defaults del descriptor | RN-10 |
| TC-44 | `UmbralCanalesDistintos` dominio | Valor borde **[gap-BD]** | `2` (positivo); `0`/`-1` | Positivo válido; `0`/negativo rechazados/normalizados por el dominio | CU-01, RN-10 |

## Cobertura y gaps

- **Cobertura por tabla:** las 13 tablas tienen al menos un TC; las principales (Servidor,
  Incidente+hijas, Regla, Exencion, Evento/Grupo/Accion, Administrador, ParametrosServidor) tienen
  clases de equivalencia, bordes, colisión de unicidad y casos negativos. Total: **44 TC**.
- **Gap-BD (validación no en la base):** TC-03, TC-08, TC-23, TC-33, TC-39, TC-44 dependen de que la
  **capa de Dominio/servicio** haga cumplir invariantes que la BD **no** fuerza (sin `CHECK`). Si se
  bordea el servicio e se escribe directo en el archivo, la base aceptaría datos inválidos.
- **Gap-FK-lógica:** las relaciones por snowflake (`Incidente.ServidorId`, `Regla`/`Exencion`/
  `Evento`/`GrupoDeReglas`/`ParametrosServidor.SnowflakeServidor`) y `ReversionAutorId` /
  `ReglaContenidoId` / `EventoGrupo.GrupoDeReglasId` **no** son FK forzadas; su integridad la sostiene
  la aplicación (no hay TC de cascada para ellas porque la BD no la ejecuta).
- **Gap-datos:** matriz derivada del esquema sin acceso a datos reales; no cubre volumen,
  distribución ni rendimiento. Los datos sintéticos de apoyo viven en
  [`fixtures/`](fixtures/) (seed fijo, PII-safe).
