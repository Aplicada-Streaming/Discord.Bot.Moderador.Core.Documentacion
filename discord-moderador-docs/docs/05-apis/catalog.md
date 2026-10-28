---
doc_id: API-CAT-001
doc_type: api-catalog
title: "Catálogo de APIs — DiscordModeradorBot.Servicio"
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: "equipo-discordmoderador (propuesto)"
last_review: 2026-07-15
review_cycle_days: 180
audience: [desarrolladores, integradores, mantenedores]
classification: uso-interno
traces:
  - src/DiscordModeradorBot.Servicio/Program.cs
  - src/DiscordModeradorBot.Servicio/Aplicacion/Puertos/
---

# Catálogo de APIs

## Alcance y omisión declarada

La pieza `DiscordModeradorBot.Servicio` es un **monolito web** que combina panel Blazor Server,
bot de moderación y persistencia en un solo proceso auto-hospedado.

> **Omisión declarada (Marco §6):** la pieza **no expone una REST API pública** ni un contrato
> **OpenAPI/Swagger**. No hay documento de esquema publicado porque no existe superficie REST para
> terceros. Los únicos endpoints HTTP son los endpoints internos minimal-API de **autenticación
> del panel** (mismo origen, formularios POST del propio Blazor) y un conjunto de endpoints de
> **sembrado solo para el entorno de pruebas E2E**. La comunicación con Discord es saliente, vía la
> librería Discord.Net (no una API entrante propia).

## Endpoints internos de autenticación (minimal-API)

Todos son `POST` de formulario del propio panel, validan **antiforgery** (protección CSRF) y
responden con un **redirect** (no JSON). Fuente: `src/DiscordModeradorBot.Servicio/Program.cs`.

| Método | Ruta | Propósito | Auth | Traza |
| --- | --- | --- | --- | --- |
| POST | `/api/auth/configuracion-inicial` | Alta first-run del administrador único; valida confirmación y política de contraseña; si el setup ya está completo, redirige a `/ingresar`. | Anónimo + antiforgery | CU-08, RN-13 |
| POST | `/api/auth/ingresar` | Ingreso del administrador; emite la cookie de sesión con claims del rol `administrador`; aplica control de intentos por identificador+IP. | Anónimo + antiforgery | CU-09, RN-12, ADR-03 |
| POST | `/api/auth/salir` | Cierre de sesión; limpia la cookie y redirige a `/ingresar`. | Sesión + antiforgery | CU-09 |
| POST | `/api/auth/cambiar-contrasena` | Cambio de contraseña del administrador autenticado; exige la actual y valida la nueva contra la política. | **RequireAuthorization** + antiforgery | RN-13 |

Notas de comportamiento (sin enumeración de cuentas):
- El login devuelve un mensaje neutro sin distinguir qué campo falló (`AUTH_CREDENCIALES_INVALIDAS`,
  CU-09 CA-02) y bloquea temporalmente tras demasiados intentos (`AUTH_DEMASIADOS_INTENTOS`).
- Los códigos de error viajan como query string en el redirect (por ejemplo
  `?error=SETUP_CONTRASENA_DEBIL`, `?error=ANTIFORGERY`).

## Endpoints de sembrado E2E (solo entorno `E2E`)

Se montan **únicamente** cuando `ASPNETCORE_ENVIRONMENT=E2E`; en desarrollo y producción **no
existen** (no son superficie de ataque). Preparan estado determinista para la suite Playwright.
Fuente: `Program.cs` (`MapearEndpointsSembradoE2E`).

| Método | Ruta | Propósito |
| --- | --- | --- |
| POST | `/e2e/seed/administrador` | Crea (idempotente) un administrador con credenciales conocidas para los flujos de login. |
| POST | `/e2e/seed/incidente-baneo` | Crea un incidente de baneo ejecutado con copia de mensajes/canales, candidato a desbaneo; devuelve el id persistido. |

## Contratos internos (puertos de aplicación)

La pieza sí define contratos **internos** (puertos hexagonales) en
`src/DiscordModeradorBot.Servicio/Aplicacion/Puertos/`, implementados por la Infraestructura. No
son API de red; son las fronteras entre Aplicación e Infraestructura. Mapeo puerto→implementación
en [`architecture-local.md`](../pieces/discordmoderadorbot-servicio/architecture-local.md#puertos-y-adaptadores).

| Puerto | Rol |
| --- | --- |
| `IAdaptadorGateway` | Acciones de moderación contra la plataforma (reportar, banear con borrado, timeout, expulsar, roles, desbanear, probar configuración). |
| `IFabricaClienteGateway` / `IClienteGatewayServidor` | Fábrica y ciclo de vida de conexiones de gateway por servidor (modo Discord). |
| `IEstadoConexionGateway` | Estado de conexión por servidor para el panel. |
| `IReloj` | Abstracción de tiempo inyectable (evaluación determinista de ventanas). |
| `IServicioCifrado` / `IServicioHashContrasena` | Cifrado AES de tokens y hashing PBKDF2 de contraseñas. |
| `IRepositorio*` (Servidores, Incidentes, ReglasContenido, Exenciones, Configuracion, Administrador) | Persistencia por agregado (EF Core SQLite). |

## Referencias

- [Pieza — README](../pieces/discordmoderadorbot-servicio/README.md) · [Arquitectura local](../pieces/discordmoderadorbot-servicio/architecture-local.md) · [Snippets](../pieces/discordmoderadorbot-servicio/snippets.md)
- Glosario: [`GLOSSARY.md`](../../GLOSSARY.md).
</content>
