# Discord.Bot.Moderador.Core.Documentacion

Conjunto documental de la solución **Administrador de Bots Moderador para Discord**
(proyecto `DiscordModeradorBot.Servicio`), generado conforme al **Marco de Documentación de Software
v2.2** por ingeniería inversa sobre el código fuente (fuente de verdad) y citando la documentación
SDD 2.2 existente del proyecto como fuente secundaria. Todo el contenido está en `status: draft` para
revisión humana.

## Dos artefactos

| Artefacto | Para qué sirve | Entrada |
|---|---|---|
| [`ia-db/`](ia-db/) | Base de conocimiento navegable: memoria operativa para que un agente cargue contexto de un tema sin releer las fuentes completas | [`ia-db/README.md`](ia-db/README.md) |
| [`discord-moderador-docs/`](discord-moderador-docs/) | Documentación de dos niveles (solución + pieza): arquitectura C4, runtime, datos/ER, decisiones, seguridad, operaciones, onboarding | [`discord-moderador-docs/README.md`](discord-moderador-docs/README.md) |

## Origen

- **Código (fuente de verdad):** `../Discord.Bot.Moderador.Core/`
- **Documentación SDD 2.2 (fuente secundaria):** `../Discord.Bot.Moderador.Core/SDD2.2D/docs/`
- **Generado por:** `/IA.Prompting.Templates/Tool-Prompts/Documentar-Fuentes-Software.md` · 2026-07-15
