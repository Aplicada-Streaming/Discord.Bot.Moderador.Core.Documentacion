---
doc_id: OVERVIEW-VISION-001
doc_type: vision
title: Visión, alcance y stakeholders
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: media
owner: equipo-discordmoderador (propuesto)
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, ops, producto, arquitectura]
classification: uso-interno
traces: [NB-01, NB-02, NB-03, NB-04, NB-05, NB-06, NB-07]
---

# Visión, alcance y stakeholders

> Qué es la solución, para quién y hasta dónde llega. Reconstruido desde el `README.md` del proyecto
> y la documentación SDD 2.2 existente (fuente secundaria:
> `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/00_contexto/` y `01_necesidades_negocio/`). Las
> afirmaciones de intención de producto se marcan con `confidence: media` porque provienen de la doc
> del proyecto, no del código.

## Resumen ejecutivo

El **Administrador de Bots Moderador para Discord** es un servicio auto-hospedado que **modera
automáticamente** servidores de Discord. Su capacidad distintiva es detectar el patrón de **ráfaga
distribuida** —un mismo usuario que publica en varios canales distintos en una ventana de segundos,
típico de cuentas comprometidas o bots de spam— y **contener al emisor** (baneo con borrado
retroactivo, timeout, expulsión o gestión de roles) antes de que el ataque se propague, dejando
**trazabilidad completa** de cada intervención para revisión humana.

Está pensado para **correr en hardware modesto** (una Raspberry Pi) bajo el control de **un único
administrador**, que lo opera desde un panel web y decide la configuración de moderación por servidor.

## Problema que resuelve

Un servidor de Discord grande no puede vigilarse manualmente 24/7. Los ataques de spam distribuido se
propagan en segundos y la moderación reactiva llega tarde. Las necesidades de negocio reconstruidas
(fuente SDD `01_necesidades_negocio/`) son:

| ID | Necesidad | Cómo la atiende la solución |
|---|---|---|
| NB-01 | Corte automático de la ráfaga distribuida | Detección por umbral de canales distintos + acción de contención automática |
| NB-02 | Limpieza retroactiva de los mensajes del ataque | Baneo con borrado retroactivo acotado a 7 días (tope de plataforma) |
| NB-03 | Contención de contenido no deseado | Reglas de contenido (regex) con tope de tiempo anti-ReDoS |
| NB-04 | Trazabilidad de incidentes y de falsos positivos | Incidentes persistidos con evidencia + panel de revisión |
| NB-05 | Configuración autónoma de la moderación | Configuración por servidor dirigida por descriptores, desde el panel |
| NB-06 | Operación confiable con validación previa | Prueba de configuración antes de activar un servidor; modo simulación |
| NB-07 | Mitigación de la moderación errónea | Modo simulación (no ejecuta) + reversión de baneos (desbaneo) desde el panel |

## Alcance

### Dentro de alcance (implementado, v1)

- Detección de ráfaga distribuida y evaluación de reglas de contenido y de conducta combinables.
- Acciones de contención: baneo, baneo con borrado retroactivo, timeout, expulsión, asignar/quitar
  rol, y reporte a un canal privado.
- Modo **simulación** (registra sin ejecutar) y modo **ejecución** (aplica las acciones reales).
- Antirrebote por usuario para no duplicar acciones durante un mismo ataque.
- Exenciones por usuario, rol o canal (descarte previo).
- Panel web de administración: revisión de incidentes, registro de servidores con token cifrado,
  configuración de moderación, exenciones, desbaneo, estado de conexión.
- Administrador único con autenticación por cookie y credenciales resguardadas.
- Persistencia local en SQLite; despliegue self-contained para linux-arm (systemd) y contenedor
  Docker x64.

### Fuera de alcance (v1)

- Multi-tenant o múltiples administradores con roles diferenciados (hay **un** rol administrador).
- Alta disponibilidad / réplica de base de datos (una única SQLite local).
- Moderación de contenido multimedia (imágenes, audio) — el eje de contenido opera sobre texto.
- Integración con servicios externos de reputación o listas de amenazas.

## Stakeholders

| Stakeholder | Interés | Interacción con el sistema |
|---|---|---|
| **Administrador único** | Operar la moderación, revisar incidentes, ajustar configuración | Panel web (rol `administrador`) |
| **Miembros del servidor de Discord** | No sufrir moderación errónea; estar protegidos del spam | Indirecta (son moderados; los exentos se descartan) |
| **Operador del dispositivo** | Instalar, respaldar y actualizar el servicio en la Raspberry/host | Scripts de `scripts/servicio/`, systemd |
| **Equipo de desarrollo** | Extender reglas, acciones y parámetros con bajo riesgo | Código + esta documentación + SDD 2.2 |

## Contexto regulatorio

La solución trata datos de usuarios de Discord (identificadores y copias de mensajes como evidencia).
La documentación del proyecto incluye una decisión de cumplimiento respecto de la **Ley argentina
25.326 de Protección de Datos Personales** (ADR-06). No hay una relación formal con un regulador ni un
perfil de dominio regulado activo en el sentido del Marco §A.17; el tratamiento de datos se documenta
como consideración de seguridad y retención en
[`../06-security-compliance/`](../06-security-compliance/). Ver también la política de retención de la
evidencia en [`../03-data/`](../03-data/).

## Referencias

- Inventario de piezas: [`system-map.md`](system-map.md)
- Necesidades de negocio (fuente): `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/01_necesidades_negocio/`
- Visión de producto (fuente): `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/00_contexto/vision-producto_v1.0.md`
