---
doc_id: ONB-SETUP-001
doc_type: developer-setup
title: "Setup del desarrollador — DiscordModeradorBot.Servicio"
version: 0.1.0
status: draft
origin: ai-assisted
confidence: alta
owner: "equipo-discordmoderador (propuesto)"
last_review: 2026-07-15
review_cycle_days: 180
audience: [desarrolladores nuevos, onboarding]
classification: uso-interno
traces:
  - global.json
  - README.md
  - src/DiscordModeradorBot.Servicio/Program.cs
  - src/DiscordModeradorBot.Servicio/SMOKE-TEST-DISCORD.md
  - scripts/ci/verificar-cobertura.ps1
---

# Setup del desarrollador

Guía de arranque para un desarrollador nuevo de la pieza `DiscordModeradorBot.Servicio`
(solución `Discord.Bot.Moderador.Core`). Referencia de la pieza:
[`README.md`](../pieces/discordmoderadorbot-servicio/README.md).

## Objetivo

Dejar un entorno local capaz de compilar, correr el servicio en modo **Simulado**, ejecutar la
suite de tests con cobertura y (opcionalmente) los e2e de Playwright, y conocer el camino para
pasar a modo **Discord** real.

## Prerrequisitos

- **SDK de .NET 10** — versión `10.0.300` (fijada por `global.json` con `rollForward: latestFeature`).
  Verificar con `dotnet --version` (>= 10.0.300).
- **PowerShell (`pwsh`)** — para los scripts de cobertura y publicación.
- **Git** — el versionado (Nerdbank.GitVersioning) necesita el historial; clonar completo (sin
  `--depth 1`).
- **(Opcional) Navegador Chromium de Playwright** — solo para los e2e; se instala una vez (ver §6).
- **(Opcional, modo Discord)** — una aplicación-bot de Discord y la clave maestra de cifrado.

## Procedimiento

### 1. Clonar y verificar el SDK

```bash
git clone <URL-del-repo> Discord.Bot.Moderador.Core
cd Discord.Bot.Moderador.Core
dotnet --version   # debe ser >= 10.0.300
```

*Resultado esperado:* el repo clonado con historial completo y el SDK correcto.

### 2. Compilar

```bash
dotnet build
```

*Resultado esperado:* build sin errores (los warnings no son error en build local; en CI se usa
`-warnaserror`).

### 3. Correr en modo Simulado

```bash
dotnet run --project src/DiscordModeradorBot.Servicio
```

Arranca con `Moderacion:Gateway=Simulado` (sin red ni token): aplica migraciones, activa WAL,
levanta el panel y un walking skeleton que inyecta escenarios de demostración. La URL se informa
por consola. Páginas: `/incidentes`, `/servidores`, `/configuracion`, `/exenciones`.

*Resultado esperado:* el panel responde y aparecen incidentes de demostración.

### 4. First-run del administrador

- En entorno `Development` se siembra un admin de desarrollo si no existe (usuario `admin`,
  contraseña de desarrollo configurable por `Seed:AdminDesarrollo:*`; ver
  [`configuration.md`](../pieces/discordmoderadorbot-servicio/configuration.md)).
- Fuera de `Development`, el alta es real: abrir `/configuracion-inicial`, crear las credenciales y
  luego ingresar en `/ingresar`.

*Resultado esperado:* sesión iniciada con el rol `administrador`.

### 5. Correr tests + cobertura

```bash
# Unit + integración (sin red ni navegador)
dotnet test DiscordModeradorBot.slnx

# Cobertura con los umbrales del proyecto
dotnet test --settings coverlet.runsettings --results-directory TestResults
pwsh scripts/ci/verificar-cobertura.ps1
```

*Resultado esperado:* tests en verde y el gate de cobertura cumpliendo los umbrales (global líneas
>= 75% / branches >= 65%; módulo de detección >= 90%).

### 6. Instalar Playwright y correr los e2e (opcional)

Los e2e viven **fuera** de la solución para no afectar el gate unitario. Instalar el navegador una
vez y luego correr por path:

```bash
dotnet build tests/DiscordModeradorBot.Servicio.E2E
pwsh tests/DiscordModeradorBot.Servicio.E2E/bin/Debug/net10.0/playwright.ps1 install chromium
dotnet test tests/DiscordModeradorBot.Servicio.E2E/DiscordModeradorBot.Servicio.E2E.csproj
```

Sin el navegador instalado, los e2e se **omiten** (no fallan). La suite usa el entorno `E2E`, que
habilita los endpoints de sembrado `/e2e/seed/*` (ver [catálogo de APIs](../05-apis/catalog.md)).

*Resultado esperado:* e2e en verde, o omitidos si no hay Chromium.

### 7. Pasar a modo Discord real (opcional)

Seguir la guía del repo `src/DiscordModeradorBot.Servicio/SMOKE-TEST-DISCORD.md`. En resumen:

1. Crear la aplicación-bot y habilitar los intents privilegiados (**MessageContent** y **GuildMembers**).
2. Invitar el bot con permisos y subirlo en la jerarquía del servidor.
3. Exportar la clave maestra de cifrado:
   ```bash
   export DISCORDMODERADOR_CLAVE_MAESTRA="<clave-maestra-sintetica-solo-local>"
   ```
4. Poner `Moderacion__Gateway=Discord`.
5. Registrar el token del servidor **por el panel** (queda cifrado en reposo con AES; nunca en el repo).
6. Correr la **prueba de configuración** desde el panel y probar primero en modo **Simulación**
   antes de pasar a **Ejecución**.

> Nunca uses tokens ni claves reales en documentación, tests o commits. La clave maestra y los
> tokens viven solo en el entorno del dispositivo (systemd `EnvironmentFile`) o del contenedor.

*Resultado esperado:* el bot conecta al servidor real, la prueba de configuración pasa y los
incidentes se generan en modo Simulación antes de habilitar Ejecución.

## Referencias

- Pieza: [`README.md`](../pieces/discordmoderadorbot-servicio/README.md) ·
  [`configuration.md`](../pieces/discordmoderadorbot-servicio/configuration.md) ·
  [`architecture-local.md`](../pieces/discordmoderadorbot-servicio/architecture-local.md)
- Publicación en la Raspberry: `scripts/servicio/README.md` (repo).
- Glosario: [`GLOSSARY.md`](../../GLOSSARY.md).
</content>
