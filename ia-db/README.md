# ia-db — Discord.Bot.Moderador.Core

> **Instrucción para IA.** Este archivo es el **punto de entrada único** a la base de conocimiento
> (`ia-db`) del proyecto `DiscordModeradorBot.Servicio`. Para cargar contexto de un tema: (1) leé la
> tabla de navegación de abajo, (2) abrí el índice temático que corresponde en `indexes/`, (3)
> ampliá al índice `00_MASTER-INDEX.md` si necesitás la visión general. No recorras el repositorio
> completo ni cargues toda la documentación cuando la respuesta ya está indexada. Cada afirmación de
> los índices es trazable a una ruta `src/...`, `tests/...`, `scripts/...`, `.github/...` o a la
> documentación fuente `SDD2.2D/docs/...`. Las fuentes primarias (código) mandan sobre este índice:
> si la evidencia contradice el índice, prevalece la evidencia.

## Necesitás saber... → Leé este índice

| Necesitás saber sobre... | Leé |
|---|---|
| Visión general, stack, decisiones (ADRs), mapa de capas | [`indexes/00_MASTER-INDEX.md`](indexes/00_MASTER-INDEX.md) |
| Capas Clean, pipeline de moderación de 9 etapas, puertos/adaptadores, composición | [`indexes/01_arquitectura-y-pipeline.md`](indexes/01_arquitectura-y-pipeline.md) |
| Modelo de dominio: políticas, reglas, conducta, contenido, exenciones, snowflake | [`indexes/02_dominio.md`](indexes/02_dominio.md) |
| `MotorDeModeracion`, servicios de aplicación y puertos (`Puertos/`) | [`indexes/03_aplicacion-servicios.md`](indexes/03_aplicacion-servicios.md) |
| Gateway (Discord/Simulado), persistencia EF, seguridad, hosted services, DI | [`indexes/04_infraestructura.md`](indexes/04_infraestructura.md) |
| Panel Blazor: páginas, layout, rutas, endpoints de autenticación | [`indexes/05_panel-web.md`](indexes/05_panel-web.md) |
| Modelo de datos: 13 tablas, índices, migraciones, snowflakes como texto | [`indexes/06_modelo-de-datos.md`](indexes/06_modelo-de-datos.md) |
| Suites de prueba, cobertura, gates de calidad | [`indexes/07_pruebas-y-calidad.md`](indexes/07_pruebas-y-calidad.md) |
| Scripts, systemd, Docker, workflows CI/CD, publicación, versionado | [`indexes/08_devops-despliegue.md`](indexes/08_devops-despliegue.md) |
| Flags de configuración, variables de entorno, descriptores, postura de seguridad | [`indexes/09_configuracion-y-seguridad.md`](indexes/09_configuracion-y-seguridad.md) |
| Vocabulario del dominio (glosario condensado) | [`indexes/10_glosario.md`](indexes/10_glosario.md) |

## Resumen ejecutivo

| Dato | Valor |
|---|---|
| **Solución** | Administrador de Bots Moderador para Discord |
| **Proyecto único** | `DiscordModeradorBot.Servicio` (web-monolith) |
| **Tipo** | Monolito modular: panel Blazor Server + bot de moderación (hosted service) + persistencia, en un proceso |
| **Stack** | .NET 10 / C#, Blazor Server interactivo + MudBlazor 9.5, Discord.Net 3.20.1, EF Core 10 + SQLite (WAL) |
| **Arquitectura** | Clean por capas — `Dominio/` (núcleo puro) → `Aplicacion/` (motor + servicios + `Puertos/`) → `Infraestructura/` (adaptadores) → `Components/` (Blazor); entrada `Program.cs` |
| **Función principal** | Detectar y moderar spam/contenido indeseado en servidores de Discord mediante un pipeline configurable; panel web para administrarlo |
| **Repositorio** | `<repo privado de la organización>` (según unidad systemd) |
| **Versión** | 1.0 (`version.json`, Nerdbank.GitVersioning; sin tags aún). v1 funcionalmente completo (CU-01..CU-16 en R1..R7) |
| **Despliegue** | Self-contained linux-arm (Raspberry Pi, systemd) y contenedor Docker x64 (Portainer) |

Arquitectura en una línea: **un proceso .NET que corre en segundo plano el bot (hosted service que
enruta cada mensaje a un pipeline de 9 etapas) y en primer plano un panel Blazor para configurarlo,
con estado persistido en SQLite y estado efímero de conducta/antirrebote en memoria.**

## Estructura del repositorio indexado

```
Discord.Bot.Moderador.Core/
├── src/DiscordModeradorBot.Servicio/   ← Proyecto único (SDK Web)
│   ├── Dominio/          ← Núcleo puro: Moderacion, Conducta, Contenido, Exenciones,
│   │                       Configuracion, Servidores, Administracion, Gateway (mapeo), Snowflake
│   ├── Aplicacion/       ← MotorDeModeracion + servicios de caso de uso + Puertos/ (interfaces)
│   ├── Infraestructura/  ← Gateway (Discord/Simulado), Persistencia (EF), Seguridad, hosted
│   │                       services, RegistroDependencias
│   ├── Components/       ← Blazor: Pages/, Layout/
│   └── Program.cs        ← Composición del host + endpoints minimal API de autenticación
├── tests/
│   ├── DiscordModeradorBot.Servicio.Tests/   ← Unit + integración (xUnit/FluentAssertions/NSubstitute)
│   └── DiscordModeradorBot.Servicio.E2E/     ← Playwright (fuera de la solución)
├── perf/DiscordModeradorBot.Servicio.Benchmarks/  ← BenchmarkDotNet
├── scripts/{servicio,ci,local}/   ← Publicación, systemd, backup, gate de cobertura
├── .github/workflows/   ← ci.yml, publish.yml, docker-publish.yml, benchmark.yml
├── SDD2.2D/docs/        ← Documentación fuente (ADRs, CU, RN, RC, modelo de datos) — fuente secundaria
├── Dockerfile, Directory.Build.props, coverlet.runsettings, version.json, global.json
└── DiscordModeradorBot.slnx
```

## Restricciones para IA

- **Fuente de verdad = código** (`src/...`). La documentación `SDD2.2D/docs/...` es fuente secundaria:
  se referencia por id (ADR-NN, CU-NN, RN-NN, RC-NN), no se copia.
- **No modificar el origen** ni ejecutar la app desde tareas de indexado.
- Los índices **destilan y referencian**, no duplican. Un tema vive en un solo índice; los demás lo
  enlazan.
- Lo no verificable contra el código se **marca** (ver secciones "Gaps"), no se completa por inferencia.
- El glosario canónico completo vive fuera de esta ia-db, en
  [`../discord-moderador-docs/GLOSSARY.md`](../discord-moderador-docs/GLOSSARY.md); el índice
  `10_glosario.md` es una versión condensada que lo referencia.

## Manifiesto de generación

- **Generado por** : `/IA.Prompting.Templates/Tool-Prompts/Documentar-Fuentes-Software.md`
- **Alcance**      : proyecto `DiscordModeradorBot.Servicio`
- **Fuentes**      : `src/`, `tests/`, `perf/`, `scripts/`, `.github/`, `SDD2.2D/docs/`, raíz del repo
- **Exclusiones**  : `.git`, `bin`, `obj`, `wwwroot/lib`, binarios (Profile `Knowledge-Indexing`)
- **Generado**     : 2026-07-15 · **Versión**: 1.0
- **Actualizar**   : `/IA/IA.Prompts/Tool-Prompts/Indexado-Documentado/Actualizar-Indexado.md`
</content>
