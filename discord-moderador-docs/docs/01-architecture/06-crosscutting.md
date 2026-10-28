---
doc_id: ARCH-XCUT-001
doc_type: crosscutting
title: Conceptos transversales (arc42 §8) — seguridad, observabilidad, resiliencia, config, persistencia, i18n
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: "equipo-discordmoderador (propuesto)"
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, ops, arquitectura, agentes-automaticos]
classification: uso-interno
traces: [ADR-02, ADR-03, ADR-06, ADR-07, ADR-08, ADR-09, ADR-12]
---

# Conceptos transversales (arc42 §8)

> Conceptos que atraviesan todo el sistema y no pertenecen a un solo componente. Fuente de verdad:
> `Program.cs`, `Infraestructura/Seguridad/`, `Dominio/Configuracion/` y `MotorDeModeracion.cs`.

## Resumen ejecutivo

El sistema aplica **defensa en profundidad** con superficie mínima (un actor, un proceso, sin
dependencias externas más allá de Discord). La seguridad combina **cifrado AES-GCM de tokens en
reposo**, **hash PBKDF2 de la contraseña del administrador**, **cookie endurecida + antiforgery +
control de intentos** y un **tope de tiempo anti-ReDoS** en las reglas de contenido. La
observabilidad se apoya en **logging estructurado del pipeline** (cada etapa se registra) y en
diagnóstico específico de ráfaga. La resiliencia sigue el principio **"nunca tumbar el pipeline"**
(ADR-08): errores y limitaciones se degradan a resultados clasificados, no a excepciones. La
configuración está **dirigida por descriptores** (ADR-12) y la persistencia usa **SQLite en modo
WAL** (ADR-02). El idioma de dominio, código y UI es **español**.

## 1. Seguridad

| Concepto | Mecanismo | Implementación | Traza |
|---|---|---|---|
| **Cifrado de tokens en reposo** | AES-256-GCM (cifrado autenticado; nonce+tag por texto) | `ServicioCifradoAes.cs`; clave derivada por SHA-256 de `DISCORDMODERADOR_CLAVE_MAESTRA` | RN-14, ADR-07 |
| **Resguardo de contraseña** | PBKDF2-HMAC-SHA256, formato PHC auto-descriptivo, 210 000 iteraciones, comparación de tiempo constante | `ServicioHashContrasenaPbkdf2.cs` | RN-13, ADR-03 |
| **Autenticación** | Cookie `DiscordModerador.Auth`: HttpOnly, SameSite=Strict, SecurePolicy=SameAsRequest, expiración 8 h deslizante | `Program.cs` (AddCookie) | CU-09, RN-12, ADR-03 |
| **Antiforgery (CSRF)** | Token validado en todos los POST (`/api/auth/*`); cookie HttpOnly/Strict | `Program.cs` (`EsRequestValidaAsync`) | endurecimiento |
| **Control de intentos** | 5 fallos / ventana 15 min → bloqueo 15 min; clave `usuario|IP` (sin enumeración de cuentas) | `ControlIntentosAutenticacion.cs` | CU-09 (AUTH_DEMASIADOS_INTENTOS) |
| **Tope anti-ReDoS** | `matchTimeout` por regla (default 100 ms); un exceso NO propaga, se trata como no coincidencia señalada | `EvaluadorReglaContenido.cs`; `RegistroDescriptores.TopeTiempoEvaluacionContenidoMs` | CU-04, ADR-08 |
| **Mínimo privilegio (despliegue)** | Usuario de servicio sin privilegios; systemd endurecido; contenedor no-root | `discord-moderador-bot.service`, `Dockerfile` | ADR-05, ADR-14 |

> **Clave maestra en desarrollo:** si `DISCORDMODERADOR_CLAVE_MAESTRA` no está definida, se deriva
> una clave de desarrollo **determinista** (`clave-de-desarrollo-no-usar-en-produccion`). Es solo
> para no romper el arranque local; **en producción la variable es obligatoria** (ADR-07). La cookie
> usa `SameAsRequest` porque el panel se sirve por HTTP auto-hospedado sin certificado (ADR-05);
> forzar `Secure=Always` rompería la sesión sobre HTTP.

## 2. Logging y observabilidad

- **Trazabilidad del pipeline:** cada etapa registra su decisión con `ILogger` estructurado
  (descarte por exención, coincidencia de política, simulación, supresión por antirrebote,
  resultado de cada acción de contención, publicación del reporte). Los mensajes citan CU/RN para
  correlación.
- **Diagnóstico de ráfaga (CU-01):** cuando un usuario está activo en más de un canal, se loguea
  cuántos canales distintos acumuló en la ventana y el umbral aplicado, sin ensuciar el log con la
  charla normal de un solo canal. Solo cuando hay parámetros por servidor (operación real).
- **Salida de logs:** al `journald` en systemd (`SyslogIdentifier=discord-moderador-bot`) y a
  stdout/stderr en Docker. **Nunca** se loguean tokens ni contraseñas (RN-13, RN-14).
- **Versión de build** visible en el panel (`BUILD_VERSION`) para identificar exactamente qué
  imagen/paquete corre.

## 3. Resiliencia y manejo de errores (ADR-08)

Principio rector: **una limitación o un fallo aislado no debe tumbar el pipeline ni la conexión**.

| Situación | Comportamiento | Traza |
|---|---|---|
| Contención imposible (jerarquía superior / permisos faltantes) | Incidente `NoAccionable`; **igual se reporta** y se registra; el pipeline continúa | RN-01, ADR-08 |
| Regla de contenido que excede el tope de tiempo | Se trata como no coincidencia, se registra `CONTENIDO_EVALUACION_EXCEDE_TIEMPO`, sigue | CU-04, ADR-08 |
| Evento de configuración inconsistente | Se omite ese evento/grupo/regla y se continúa con los demás | RN-15, RC-04, ADR-08 |
| Fallo al conectar un servidor (modo Discord) | Se deja desconectado (transitorio), no cae el gestor; permite reintento | CU-13, ADR-08 |
| Error al enrutar un mensaje al motor | Se descarta ese mensaje, la conexión sigue viva | ADR-08 |
| Caída del proceso (producción) | `systemd Restart=always` reinicia (sostiene el SLO de disponibilidad) | ADR-05 |

**Clasificación por peor resultado:** el incidente refleja el desenlace más severo de sus acciones
de contención (`NoAccionable` > `Fallida` > `Ejecutada`), reclasificando en cuanto se conoce cada
resultado.

## 4. Configuración dirigida por descriptores (ADR-12)

- `RegistroDescriptores` es la **fuente única de verdad** de defaults y límites de cada parámetro
  del dominio: umbral de canales, ventana de detección, ventana de borrado (tope 7 días), tope de
  tiempo de contenido y ventana de antirrebote. Nada de eso se hardcodea en la lógica (RN-10).
- Un valor configurado **fuera de rango se normaliza al default** (`NormalizarOPorDefecto`), no se
  rechaza: la moderación nunca se queda sin parámetro válido.
- La configuración persistida del panel **dirige** la moderación: el `CargadorPoliticasDesdeConfiguracion`
  materializa eventos→grupos→reglas→acciones por servidor (no hay política hardcodeada).

## 5. Persistencia (ADR-02)

- **EF Core + SQLite** en un **archivo local**, con **modo WAL** (`PRAGMA journal_mode=WAL` al
  arranque) para tolerar las escrituras concurrentes del bot y del panel dentro del proceso.
- **13 tablas** (servidores, parámetros, eventos, grupos, reglas, acciones y sus relaciones,
  exenciones, incidentes con mensajes/canales accionados, administrador). Los **snowflakes se
  almacenan como TEXTO** para preservar identidad y orden estable (RN-08, RC-02).
- La migración se aplica al arranque (`Database.Migrate()`); el respaldo se apoya en
  `scripts/servicio/backup-sqlite.sh`. **Un único punto de datos** (sin réplica).
- **Estado efímero en memoria** (NO persistido, ADR-09): estado de conducta, antirrebote y control
  de intentos. Se pierde al reiniciar, lo cual es aceptable por diseño (contención temporal).

## 6. Idioma e internacionalización

- El sistema es **monolingüe en español**: dominio, identificadores de código, logs, mensajes de la
  UI y documentación están en español (idioma del producto y del equipo). No hay capa de
  localización ni recursos multi-idioma; **no hay i18n** implementada. Un futuro soporte multi-idioma
  sería un cambio transversal a la UI y a los mensajes.

## Referencias

- Componentes del motor (tope de tiempo, clasificación): [`03-components.md`](03-components.md)
- Flujos (login, antirrebote, simulación): [`04-runtime-views.md`](04-runtime-views.md)
- Despliegue (clave maestra, endurecimiento): [`05-deployment.md`](05-deployment.md)
- Glosario: [`../../GLOSSARY.md`](../../GLOSSARY.md)
- Fuente secundaria (SDD 2.2): `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/05_arquitectura_tecnica/adrs/`
