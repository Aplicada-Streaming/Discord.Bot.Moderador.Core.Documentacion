---
doc_id: PIECE-CFG-001
doc_type: configuration
title: "Configuración — DiscordModeradorBot.Servicio"
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: "equipo-discordmoderador (propuesto)"
last_review: 2026-07-15
review_cycle_days: 180
audience: [desarrolladores, operadores, mantenedores]
classification: uso-interno
traces:
  - src/DiscordModeradorBot.Servicio/appsettings.json
  - src/DiscordModeradorBot.Servicio/Program.cs
  - src/DiscordModeradorBot.Servicio/Dominio/Configuracion/RegistroDescriptores.cs
  - src/DiscordModeradorBot.Servicio/Dominio/Administracion/ControlIntentosAutenticacion.cs
  - scripts/servicio/discord-bots-admin.env.example
---

# Configuración — `DiscordModeradorBot.Servicio`

Toda la configuración se toma de `appsettings*.json`, variables de entorno o *user-secrets*
(precedencia estándar de .NET: entorno sobrescribe archivo). **No hay secretos en el repositorio.**

## Claves de configuración y variables de entorno

| Clave de configuración | Variable de entorno | Valores | Default | Efecto |
| --- | --- | --- | --- | --- |
| `Moderacion:Gateway` | `Moderacion__Gateway` | `Simulado` \| `Discord` | `Simulado` | Selecciona el adaptador de gateway. `Simulado`: sin red ni token (walking skeleton). `Discord`: conexiones reales vía Discord.Net (`Program.cs`, `RegistroDependencias`). |
| `Persistencia:RutaBase` | `Persistencia__RutaBase` | ruta de archivo | `<BaseDirectory>/discordmoderador.db` | Ubicación del archivo SQLite. Permite a los e2e apuntar a una base temporal por corrida sin tocar la de desarrollo (`Program.cs`). |
| `Seed:AdminDesarrollo:Habilitado` | `Seed__AdminDesarrollo__Habilitado` | `true` \| `false` | `true` (solo `Development`) | Habilita/inhabilita el sembrado del admin de desarrollo. Solo actúa en entorno `Development`; fuera de él nunca se siembra (`WalkingSkeletonHostedService`). |
| `Seed:AdminDesarrollo:Usuario` | `Seed__AdminDesarrollo__Usuario` | texto | `admin` | Usuario del admin de desarrollo sembrado (solo `Development`). |
| `Seed:AdminDesarrollo:Contrasena` | `Seed__AdminDesarrollo__Contrasena` | texto | valor de desarrollo (no apto producción) | Contraseña del admin de desarrollo sembrado (solo `Development`). Preferir *user-secrets*. |
| `Logging:LogLevel:Default` | `Logging__LogLevel__Default` | nivel de log | `Information` | Nivel de log por defecto (`appsettings.json`). |
| `ASPNETCORE_ENVIRONMENT` | `ASPNETCORE_ENVIRONMENT` | `Development` \| `Production` \| `E2E` \| … | (según host) | Entorno del host. `E2E` habilita StaticWebAssets y los endpoints `/e2e/seed/*`; `Development` habilita el seed del admin y desactiva el handler de excepciones. |

### Secreto (nunca en el repo)

| Variable de entorno | Rol | Notas |
| --- | --- | --- |
| `DISCORDMODERADOR_CLAVE_MAESTRA` | Clave maestra de cifrado AES de los tokens de bot en reposo | **Requerida en producción/modo Discord.** Los tokens se cifran con esta clave (nunca se guardan en claro ni se loguean, ADR-07). Se provee por `EnvironmentFile` de systemd o entorno del contenedor; el `.gitignore` ignora `*.env`, solo se versiona la plantilla `discord-bots-admin.env.example`. |

> Este documento **no** incluye valores de secretos. Para el procedimiento de integración real
> con Discord (habilitar intents, registrar token cifrado por el panel, prueba de configuración),
> ver `src/DiscordModeradorBot.Servicio/SMOKE-TEST-DISCORD.md` y
> [`developer-setup.md`](../../08-onboarding/developer-setup.md).

## Descriptores de parámetro de moderación

Los parámetros del dominio no se hardcodean: su default, límites, leyenda y ejemplos viven en
`Dominio/Configuracion/RegistroDescriptores.cs` (fuente única de verdad, ADR-12, RN-10). El panel
edita estos valores por servidor; un valor fuera de rango se normaliza al default
(`NormalizarOPorDefecto`) o se acota al tope (`NormalizarConTope`), según el parámetro.

| Clave del descriptor | Etiqueta | Default | Mín | Máx | Efecto |
| --- | --- | --- | --- | --- | --- |
| `umbral-canales-distintos` | Umbral de canales distintos | 3 | 2 | 10 | Cantidad de canales distintos en los que un usuario debe publicar, dentro de la ventana, para marcar una ráfaga distribuida (CU-01). |
| `ventana-deteccion` | Ventana de detección (segundos) | 2.0 | 0.5 | 60.0 | Tiempo dentro del cual se cuentan los canales distintos (CU-01). |
| `ventana-borrado-retroactivo` | Ventana de borrado retroactivo (días) | 1 | 0 | 7 | Días hacia atrás que se purgan al banear. 0 = no borra; un valor mayor al máximo se **acota** al tope de plataforma (7 días), no se rechaza (RN-02). |
| `tope-tiempo-evaluacion-contenido` | Tope de tiempo de evaluación de contenido (ms) | 100 | 1 | 5000 | Presupuesto por evaluación de una regla de contenido (regex); si se excede, se aborta esa regla y se trata como no coincidencia — protección ReDoS (ADR-08). |
| `ventana-antirrebote` | Ventana de antirrebote por usuario (segundos) | 10.0 | 0.5 | 300.0 | Tiempo durante el cual no se repite la misma acción sobre un usuario ya accionado en la ráfaga vigente; las coincidencias adicionales se suprimen (RN-06). |

> El tope de plataforma de borrado retroactivo (`VentanaBorradoMaximaDias = 7`) es una constante
> del mismo registro (RC-11, RN-02).

## Parámetros del control de intentos de login

Constantes por defecto de `Dominio/Administracion/ControlIntentosAutenticacion.cs` (estado en
memoria, CU-09, ADR-09). El singleton se registra con estos defaults en `RegistroDependencias`.

| Parámetro | Default | Efecto |
| --- | --- | --- |
| `MaximoIntentosPorDefecto` | 5 | Fallos permitidos antes del bloqueo temporal. |
| `VentanaMinutosPorDefecto` | 15.0 min | Ventana en la que se cuentan los fallos. |
| `EnfriamientoMinutosPorDefecto` | 15.0 min | Duración del bloqueo tras superar el máximo (`AUTH_DEMASIADOS_INTENTOS`). |

## Configuración de sesión y antiforgery

Valores fijados en `Program.cs` (no configurables por archivo; endurecimiento de seguridad):

- Cookie de sesión `DiscordModerador.Auth`: `HttpOnly`, `SameSite=Strict`,
  `SecurePolicy=SameAsRequest`, expiración deslizante de 8 horas.
- Antiforgery: cookie `HttpOnly` + `SameSite=Strict` + `SecurePolicy=SameAsRequest`; todos los
  POST de `/api/auth/*` validan el token.

## Referencias

- [`README.md`](./README.md) · [`architecture-local.md`](./architecture-local.md) · [`snippets.md`](./snippets.md)
- Glosario: [`GLOSSARY.md`](../../../GLOSSARY.md).
- SDD 2.2 (secundaria): `SDD2.2D/docs/05_arquitectura_tecnica/decisiones-arquitectura_v1.0.md` (ADR-12), `reglas-de-negocio/` (RN-02, RN-06, RN-10).
</content>
</invoke>
