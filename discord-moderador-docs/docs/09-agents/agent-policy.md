---
doc_id: AGENTS-POLICY-001
doc_type: agent-policy
title: Política de agentes documentales
version: 0.1.0
status: draft
origin: human
owner: equipo-discordmoderador (propuesto)
last_review: 2026-07-15
review_cycle_days: 180
audience: [agentes-automaticos, arquitectura, dev]
classification: uso-interno
traces: []
---

# Política de agentes documentales

> Perímetro de acción de cualquier agente de IA que genere o actualice esta documentación
> (Marco §12.4). Un agente debe cumplir esta política **antes** de cualquier otra acción.

## Reglas

1. Los agentes crean y actualizan documentos **solo en `status: draft`**. La promoción
   `draft → in-review → approved` la ejecuta un **humano** revisor registrado.
2. Los agentes **nunca modifican** el código fuente del origen
   (`../../../../Discord.Bot.Moderador.Core/`): la ejecución sobre el origen es de **solo lectura**.
   Todo lo generado vive bajo `Discord.Bot.Moderador.Core.Documentacion/`.
3. Los agentes **nunca modifican**: ADRs `approved`/`superseded`, changelogs publicados, evidencia de
   auditoría, ni `GLOSSARY.md` sin una revisión humana.
4. Todo contenido **inferido** lleva `origin` (`reverse-engineered` o `ai-assisted`) y `confidence`
   (`alta`/`media`/`baja`) en el frontmatter.
5. Ante **contradicción entre fuentes** (código vs. SDD vs. este set), el agente **reporta** la
   contradicción; no elige una versión en silencio. Ante conflicto entre el código y cualquier
   documento, **gana el código** (es la fuente de verdad).
6. Los agentes **no ejecutan** la aplicación, **no se conectan** a Discord ni a una base de datos real,
   **no manejan tokens ni la clave maestra**, y **no hacen** commit, push ni pull request.
7. Ningún dato de ejemplo contiene PII real ni credenciales: los ejemplos son **sintéticos** o
   seudonimizados con formato válido (Marco §14/§15).
8. Un agente **no aprueba su propio output**: la verificación la hace un humano o un agente distinto
   con política explícita (Marco §10 crit. 6).

## Cómo un agente calcula el estado documental

Sin leer prosa, un agente puede determinar el gap documental leyendo únicamente:

1. [`../../docs-manifest.yaml`](../../docs-manifest.yaml) — qué documentos deben existir, su
   `type`, ruta y vigencia máxima.
2. El **frontmatter** de cada `.md` — `doc_id`, `status`, `origin`, `confidence`, `last_review`.

El gap es `requerido − existente − vencido`. La regeneración/actualización de la ia-db y de este set
se describe en el manifiesto de generación de [`../../../ia-db/README.md`](../../../ia-db/README.md).

## Fuentes que un agente puede leer (solo lectura)

| Fuente | Rol | Ruta |
|---|---|---|
| Código del proyecto | Fuente de verdad | `../../../../Discord.Bot.Moderador.Core/src/`, `tests/`, `scripts/`, `.github/` |
| Documentación SDD 2.2 | Fuente secundaria (citar, no copiar) | `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/` |
| ia-db | Memoria navegable | `../../../ia-db/` |
| Este conjunto documental | Salida propia | `Discord.Bot.Moderador.Core.Documentacion/discord-moderador-docs/` |
