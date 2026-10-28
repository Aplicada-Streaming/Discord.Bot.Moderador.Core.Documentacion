---
doc_id: README-MASTER-000
doc_type: index
title: Índice maestro — Documentación de la solución Administrador de Bots Moderador para Discord
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: equipo-discordmoderador (propuesto)
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, ops, qa, producto, arquitectura, agentes-automaticos]
classification: uso-interno
traces: []
---

# Documentación — Administrador de Bots Moderador para Discord

> **Para humanos:** este es el punto de entrada. Empezá por [Visión](docs/00-overview/vision.md) y el
> [Mapa del sistema](docs/00-overview/system-map.md), y bajá al tema que necesites con la tabla de
> navegación.
> **Para agentes:** el estado documental es calculable leyendo solo
> [`docs-manifest.yaml`](docs-manifest.yaml) + el frontmatter de cada `.md`. La política de agentes
> está en [`docs/09-agents/agent-policy.md`](docs/09-agents/agent-policy.md).

## Qué es esta documentación

Conjunto documental de la solución **Administrador de Bots Moderador para Discord** (proyecto
`DiscordModeradorBot.Servicio`), generado conforme al **Marco de Documentación de Software v2.2**. Se
produjo por **ingeniería inversa** sobre el código (fuente de verdad) y citando la documentación SDD
2.2 existente del proyecto como fuente secundaria. Todo nace en `status: draft` para revisión humana.

Convive con dos artefactos hermanos en este mismo repositorio de documentación:

- [`../ia-db/`](../ia-db/) — base de conocimiento navegable (memoria operativa para agentes de IA).
- `discord-moderador-docs/` (este árbol) — la documentación de dos niveles (solución + pieza).

## Resumen de la solución

| Atributo | Valor |
|---|---|
| Tipo | Monolito web auto-hospedado (`web-monolith`: panel + bot + base en un proceso) |
| Stack | .NET 10 · Blazor Server + MudBlazor · Discord.Net · EF Core + SQLite (WAL) |
| Función | Modera servidores de Discord: detecta **ráfaga distribuida** y contenido no deseado, y contiene al emisor con trazabilidad |
| Despliegue | Self-contained linux-arm (systemd, Raspberry Pi) y contenedor Docker x64 |
| Versión | 1.0 (Nerdbank.GitVersioning; v1 funcionalmente completo) |
| Fuente de verdad | `../../Discord.Bot.Moderador.Core/` (código) |

## Navegación — «Necesitás saber… → Leé esto»

| Necesitás saber… | Leé |
|---|---|
| Qué es y para qué sirve la solución | [docs/00-overview/vision.md](docs/00-overview/vision.md) |
| Qué piezas la componen | [docs/00-overview/system-map.md](docs/00-overview/system-map.md) |
| Cómo encaja todo (arquitectura C4) | [docs/01-architecture/](docs/01-architecture/) |
| Cómo se comporta en un flujo real | [docs/01-architecture/04-runtime-views.md](docs/01-architecture/04-runtime-views.md) |
| Qué requisitos cumple y su trazabilidad | [docs/02-requirements/README.md](docs/02-requirements/README.md) |
| El modelo de datos (diccionario + ER) | [docs/03-data/](docs/03-data/) |
| Casos y datos de prueba derivados del modelo | [docs/03-data/test-data-matrix.md](docs/03-data/test-data-matrix.md) |
| Por qué se decidió así (ADRs) | [docs/04-decisions/README.md](docs/04-decisions/README.md) |
| Qué endpoints/contratos expone | [docs/05-apis/catalog.md](docs/05-apis/catalog.md) |
| Postura de seguridad y datos personales | [docs/06-security-compliance/](docs/06-security-compliance/) |
| Cómo operarlo (instalar, respaldar, diagnosticar) | [docs/07-operations/README.md](docs/07-operations/README.md) |
| Cómo arrancar como desarrollador | [docs/08-onboarding/developer-setup.md](docs/08-onboarding/developer-setup.md) |
| Detalle de la pieza (README, configuración, snippets) | [docs/pieces/discordmoderadorbot-servicio/](docs/pieces/discordmoderadorbot-servicio/) |
| Vocabulario controlado | [GLOSSARY.md](GLOSSARY.md) |
| Qué puede/no puede hacer un agente | [docs/09-agents/agent-policy.md](docs/09-agents/agent-policy.md) |

## Estructura

```text
discord-moderador-docs/
├── README.md              ← este índice maestro
├── GLOSSARY.md            ← vocabulario controlado (normativo)
├── docs-manifest.yaml     ← manifiesto máquina-legible (gap documental)
└── docs/
    ├── 00-overview/       ← vision, system-map
    ├── 01-architecture/   ← C4 (contexto, contenedores, componentes), runtime, deployment, crosscutting
    ├── 02-requirements/   ← índice y trazabilidad (SRS autoritativo en SDD 2.2)
    ├── 03-data/           ← diccionario, ER (dbml), políticas de acceso, matriz de pruebas, fixtures
    ├── 04-decisions/      ← catálogo de ADRs
    ├── 05-apis/           ← catálogo de endpoints internos
    ├── 06-security-compliance/ ← threat model, protección de datos
    ├── 07-operations/     ← runbooks
    ├── 08-onboarding/     ← developer-setup
    ├── 09-agents/         ← política de agentes
    └── pieces/discordmoderadorbot-servicio/ ← doc de nivel pieza
```

## Estado y gap documental

Todos los documentos están en `status: draft` (generados por ingeniería inversa; promoción a
`approved` es humana). El gap documental y las omisiones declaradas (p. ej. sin OpenAPI porque no hay
REST API pública; SRS funcional referenciado a la SDD en lugar de duplicado) están en
[`docs-manifest.yaml`](docs-manifest.yaml) → `omitido_declarado`. El reporte de gap y de derivas de
esta generación se detalla en la respuesta de ejecución del prompt.

## Relación con la documentación SDD 2.2 del proyecto

El proyecto ya tiene un conjunto documental propio (SDD 2.2) en
`../../Discord.Bot.Moderador.Core/SDD2.2D/docs/`. Este set **no lo reemplaza ni lo duplica**: lo toma
como fuente secundaria autoritativa para requisitos, ADRs y reglas de negocio, y aporta lo que el
Marco v2.2 pide y la SDD no tiene en ese formato: la **ia-db** navegable, el **manifiesto
máquina-legible**, el **mapa de piezas**, los **diagramas C4/runtime como código**, el **ER en dbml**
y la **matriz de casos derivada del modelo**.
