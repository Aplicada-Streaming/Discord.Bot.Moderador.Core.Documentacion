---
doc_id: PIECE-CHANGELOG-001
doc_type: changelog
title: Changelog — DiscordModeradorBot.Servicio
status: draft
origin: reverse-engineered
confidence: media
owner: equipo-discordmoderador (propuesto)
last_review: 2026-07-15
audience: [dev, ops, producto]
classification: uso-interno
---

# Changelog — `DiscordModeradorBot.Servicio`

Formato basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/); versionado
SemVer derivado de Git (Nerdbank.GitVersioning, `version.json` = 1.0).

> **Reconstruido por ingeniería inversa** (`origin: reverse-engineered`). Este changelog se
> reconstruye a partir del historial de git (ventana de desarrollo aproximada 2026-06-19 →
> 2026-07-11) y de las siete rebanadas verticales R1..R7 descritas en el código y en
> `SDD2.2D/docs/`. No se afirman fechas de release exactas: la solución declara `version.json` = 1.0
> **sin un tag `v1.0` publicado**, por lo que `[1.0.0]` refleja el estado "v1 funcionalmente
> completo", no una fecha de publicación verificada.

## [Unreleased]

### Añadido
- Dockerización del servicio (imagen x64) con pipeline CI/CD (ADR-14).
- Diagnóstico de ráfaga en consola y parámetros de detección configurables por servidor
  (CU-11, CU-01).

### Corregido
- Servido de `blazor.web.js` en el contenedor publicado (`UseStaticFiles`) para que la
  interactividad `InteractiveServer` arranque.
- Estabilización de los e2e del panel (asistente de reglas, navegación por pasos).

### Pendiente / decisiones abiertas
- Calibración con datos reales de los defaults de detección (umbral de canales y ventana).

## [1.0.0] — v1 funcionalmente completo (sin tag de release)

Los 16 casos de uso del intake implementados en siete rebanadas verticales, integración real con
Discord.Net, CI/CD con quality gates, endurecimiento de seguridad y pruebas e2e.

### Añadido

**R1 — Detección y simulación**
- Detección de la *ráfaga distribuida*: un usuario que publica en varios canales distintos dentro
  de una ventana corta (`EvaluadorRafagaDistribuida`, CU-01).
- Modo **Simulación** como modo seguro por defecto: registra lo que haría sin ejecutar acción real
  (RN-09).
- Registro de descriptores de parámetro como fuente única de verdad: `umbral-canales-distintos`
  (default 3) y `ventana-deteccion` (default 2 s) (`RegistroDescriptores`, ADR-12, RN-10).

**R2 — Ejecución real (baneo, borrado, reporte)**
- Camino de **Ejecución**: toma de copia de mensajes antes de remover (RN-11) y ejecución de
  acciones en orden (RN-05).
- Baneo con borrado retroactivo acotado a 7 días (CU-02/CU-03, RN-02) y reporte del incidente a un
  canal privado (CU-05).

**R3 — Reglas de contenido**
- Segundo eje de defensa: reglas de contenido sin estado evaluadas antes del estado de conducta
  (`EvaluadorReglaContenido`, CU-04).
- Tope de tiempo por evaluación (`tope-tiempo-evaluacion-contenido`, default 100 ms): protección
  ante regex con retroceso catastrófico/ReDoS; una regla que excede el tope se trata como no
  coincidencia sin colgar el pipeline (ADR-08).

**R5 — Exenciones**
- Etapa 1 del pipeline "descarte de exentos": el mensaje se descarta si el emisor, alguno de sus
  roles o el canal están exentos, antes de evaluar contenido/conducta (`EvaluadorExenciones`,
  CU-15, RN-07). Página `/exenciones`.

**R6 — Antirrebote, catálogo de acciones y resultado**
- Antirrebote por usuario (`ventana-antirrebote`, default 10 s): suprime acciones repetidas sobre
  un usuario ya accionado en la ráfaga vigente (CU-16, RN-06, ADR-09).
- Catálogo de acciones completo: Timeout, Expulsar, AsignarRol, QuitarRol, además del baneo y el
  reporte (`TipoAccion`, intake §4).
- Mapeo del `ResultadoAccion` del adaptador al `ResultadoModeracion` del incidente: ante jerarquía
  superior/permisos faltantes el incidente se registra como `NoAccionable`, igual se reporta y el
  pipeline continúa (RN-01).

**R7 — Configuración normalizada y prueba de configuración**
- Modelo de configuración normalizado (grupos, eventos y acciones) que **dirige** la moderación:
  las políticas se cargan de la configuración persistida del panel, no hay política hardcodeada
  (`CargadorPoliticasDesdeConfiguracion`, CU-11).
- Composición booleana de grupos de reglas (contenido y/o conducta) (RN-15).
- Prueba de configuración previa a la activación del servidor (`ServicioPruebaConfiguracion`,
  CU-12, RN-16).

**Autenticación y seguridad**
- Administrador único: alta first-run (`/configuracion-inicial`, CU-08) e ingreso (`/ingresar`,
  CU-09) por endpoints minimal-API con antiforgery; cambio de contraseña autenticado
  (`/api/auth/cambiar-contrasena`, RN-13).
- Sesión por cookie con `HttpOnly` + `SameSite=Strict` + `SecurePolicy=SameAsRequest`, expiración
  deslizante de 8 h (ADR-03, ADR-05).
- Hashing PBKDF2 (formato PHC) de contraseñas y cifrado AES en reposo de los tokens de bot con la
  clave maestra `DISCORDMODERADOR_CLAVE_MAESTRA` (ADR-07).
- Control de intentos de login en memoria (5 intentos / ventana 15 min / enfriamiento 15 min,
  CU-09).

**Persistencia**
- SQLite en modo WAL con migración al arranque (ADR-02). Migraciones MIG0001..MIG0009 (esquema
  inicial, normalización de evidencia y canal de salida, regla de contenido, administrador y
  reversión de incidente, exención, configuración normalizada, parámetros por servidor, nombre de
  canal y usuario en mensaje accionado).

**Integración y DevOps**
- Modo de gateway seleccionable (`Moderacion:Gateway`): `Simulado` (walking skeleton) | `Discord`
  (Discord.Net + gestor de conexiones por servidor) (ADR-13).
- CI/CD con quality gates (formato, build Release con analizadores y warnings como error, tests,
  cobertura por umbrales, dependencias vulnerables, SBOM) y publicación self-contained `linux-arm`
  por tag.

### Seguridad
- Pin directo de `SQLitePCLRaw.*` (bundle 3.0.3 / lib 3.50.3) para forzar SQLite 3.50.x y mitigar
  la vulnerabilidad alta GHSA-2m69-gcr7-jv3q traída transitivamente por EF Core SQLite.

---

Referencias: [`README.md`](./README.md) · SDD 2.2 del repo (`SDD2.2D/docs/06_backlog-tecnico/`,
`07_plan-sprint/`) para el detalle de las rebanadas.
</content>
