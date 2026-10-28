---
doc_id: OPS-INDEX-001
doc_type: operations-index
title: Runbooks operativos
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: equipo-discordmoderador (propuesto)
last_review: 2026-07-15
review_cycle_days: 180
audience: [ops, dev, agentes-automaticos]
classification: uso-interno
traces: [ADR-02, ADR-05, ADR-07, ADR-13, ADR-14, RN-14, RN-16, CU-10, CU-12, CU-13]
---

# Runbooks operativos

> Procedimientos de operación de la solución **Administrador de Bots Moderador para Discord**. Cada
> runbook lleva un ID `RB-` (ver [`../../GLOSSARY.md`](../../GLOSSARY.md)). Los scripts referenciados
> son la **fuente de verdad**: `../../../../Discord.Bot.Moderador.Core/scripts/servicio/`. Glosario y
> convenciones aplican; este documento no reescribe los scripts, los opera.

## Contexto de despliegue

Existen **dos vías de despliegue** documentadas: el servicio systemd self-contained `linux-arm`
(ADR-05, base de RB-01/03) y la imagen Docker `x64` gestionada por Portainer (ADR-14, base de
RB-02, que **supera** a ADR-05). Los runbooks indican la vía aplicable. La clave maestra y los
tokens **nunca** se versionan ni viven en la base (ADR-07).

| RB | Objetivo | Vía |
|---|---|---|
| [RB-01](#rb-01--instalaractualizar-en-la-raspberry-systemd) | Instalar/actualizar en la Raspberry (systemd) | linux-arm / systemd |
| [RB-02](#rb-02--backup-y-restore-de-sqlite) | Backup y restore de SQLite | Docker (backup-sqlite.sh) |
| [RB-03](#rb-03--definir-o-rotar-la-clave-maestra) | Definir o rotar la clave maestra | ambas |
| [RB-04](#rb-04--pasar-de-modo-simulado-a-discord) | Pasar de modo Simulado a Discord | ambas |
| [RB-05](#rb-05--diagnosticar-un-servidor-que-no-conecta) | Diagnosticar un servidor que no conecta | ambas |

---

## RB-01 — Instalar/actualizar en la Raspberry (systemd)

- **Objetivo**: desplegar o actualizar el servicio en el dispositivo ARM preservando el entorno y la
  clave maestra (tokens cifrados siguen válidos). Materializa ADR-05.
- **Prerrequisitos**: acceso `sudo` en el dispositivo; paquete `discord-bots-admin_<X.Y.Z>_linux-arm.zip`
  y su `.sha256`; `unzip` disponible. Para publicar, estación x64 con .NET SDK.
- **Pasos**:
  1. **Publicar** (estación x64 / CI): `./scripts/servicio/publicar.sh <X.Y.Z>` → genera
     `artefactos/discord-bots-admin_<X.Y.Z>_linux-arm.zip` + `.sha256` (verifica ejecutable y
     `libe_sqlite3.so`).
  2. **Verificar integridad** en el dispositivo: `sha256sum -c discord-bots-admin_<X.Y.Z>_linux-arm.zip.sha256`.
  3. **Actualización**: `sudo systemctl stop discord-moderador-bot` (omitir en primera instalación).
  4. **Instalar**: `sudo ./instalar.sh --paquete discord-bots-admin_<X.Y.Z>_linux-arm.zip`
     (idempotente; crea usuario `discordbot`, directorios y la unidad systemd; **preserva** el `.env`).
  5. **Primera vez**: completar la clave maestra (ver RB-03) y `sudo systemctl enable --now discord-moderador-bot`.
     **Actualización**: `sudo systemctl start discord-moderador-bot`.
- **Resultado esperado**: `systemctl status discord-moderador-bot` muestra `active (running)`.
- **Validación**: `journalctl -u discord-moderador-bot -n 50` sin errores de arranque; el panel
  responde en `ASPNETCORE_URLS` (por defecto `http://127.0.0.1:5080`).
- **Errores posibles**:
  - `no se encontró el ejecutable`: pasar `--paquete <zip>` o correr desde la carpeta del paquete.
  - Servicio no arranca por falta de clave: completar `.env` (RB-03) y reiniciar.
  - Rollback: reinstalar el paquete previo con `--conservar-entorno` (por defecto); no toca la base
    ni el `.env` (`scripts/servicio/README.md §4`).
- **Fuentes**: `scripts/servicio/{publicar.sh,instalar.sh,discord-moderador-bot.service,README.md}`.

## RB-02 — Backup y restore de SQLite

- **Objetivo**: respaldar y restaurar la base SQLite (evidencia de incidentes y configuración) con
  un checkpoint WAL limpio (ADR-02).
- **Prerrequisitos**: vía Docker: `docker`, `zip`; contenedor `discord-bot-moderador` en ejecución.
  Espacio en el directorio destino.
- **Pasos (backup)**:
  1. `./scripts/servicio/backup-sqlite.sh [directorio-destino]` (por defecto `./backups`).
  2. El script **detiene** el contenedor (checkpoint WAL limpio, ADR-02), copia
     `/app/data/discordmoderador.db`, **reinicia** el contenedor y comprime en
     `discordmoderador_<YYYYMMDD_HHMMSS>.zip`.
- **Pasos (restore)**:
  1. `docker stop discord-bot-moderador`
  2. `unzip discordmoderador_<TS>.zip -d /tmp/`
  3. `docker cp /tmp/discordmoderador_backup.db discord-bot-moderador:/app/data/discordmoderador.db`
  4. `docker start discord-bot-moderador`
- **Resultado esperado**: archivo zip con timestamp en el destino; el servicio vuelve a `running`.
- **Validación**: tras el restore, el panel lista los incidentes y servidores esperados; los tokens
  siguen descifrándose (la clave maestra del entorno no cambió — ver RB-03).
- **Errores posibles**:
  - Base bloqueada / WAL sin checkpoint: no copiar en caliente; el script detiene el contenedor a
    propósito para evitarlo.
  - En systemd/linux-arm el archivo vive en `/var/lib/discord-bots-admin/`; adaptar el copiado
    (detener el servicio con `systemctl stop` antes de copiar).
- **Fuentes**: `scripts/servicio/backup-sqlite.sh`; ADR-02 (WAL).

## RB-03 — Definir o rotar la clave maestra

- **Objetivo**: definir por primera vez o rotar `DISCORDMODERADOR_CLAVE_MAESTRA`, el secreto con el
  que se cifran los tokens en reposo (ADR-07, RN-14).
- **Prerrequisitos**: acceso al archivo de entorno (`/etc/discord-bots-admin/discord-bots-admin.env`
  en systemd, permisos `600`; variables de entorno del contenedor en Docker). Resguardo previo de la
  clave actual si se va a rotar.
- **Pasos (definir por primera vez)**:
  1. Generar una clave fuerte: `openssl rand -base64 32`.
  2. Pegarla en `DISCORDMODERADOR_CLAVE_MAESTRA=` del `.env` (`sudoedit` en systemd) y guardar.
  3. Arrancar/reiniciar el servicio.
  4. **Resguardar** la clave fuera del dispositivo (gestor de contraseñas del operador).
- **Pasos (rotar)**: rotar la clave **invalida los tokens ya cifrados** (se cifraron con la clave
  anterior). Procedimiento: 1) definir la nueva clave; 2) reiniciar; 3) **re-registrar** cada token
  por el panel (CU-10); 4) probar cada servidor (RB-04 / CU-12).
- **Resultado esperado**: el servicio arranca y opera; en producción la clave **no** es la de
  desarrollo determinista.
- **Validación**: un servidor previamente registrado supera la prueba de configuración (RB-05); no
  aparecen errores de descifrado en el journal; el token nunca aparece en logs (RN-14).
- **Errores posibles**:
  - **Clave omitida en producción**: el cifrado usa la clave de desarrollo determinista (riesgo
    crítico R3 en [`../06-security-compliance/threat-model.md`](../06-security-compliance/threat-model.md)).
  - **Clave perdida**: los tokens cifrados dejan de descifrarse; re-registrarlos por el panel.
  - El rollback/instalación **preserva** el `.env` (no re-registrar tokens tras actualizar).
- **Fuentes**: `scripts/servicio/{discord-bots-admin.env.example,README.md §5}`;
  `ServicioCifradoAes.cs`; ADR-07.

## RB-04 — Pasar de modo Simulado a Discord

- **Objetivo**: llevar un servidor de la operación simulada (sin red ni token) a la moderación real
  contra Discord, validando antes de accionar (ADR-13, RN-16).
- **Prerrequisitos**: aplicación-bot creada en el portal de Discord con **token**; **intents
  privilegiados** habilitados (MESSAGE CONTENT y SERVER MEMBERS); bot invitado con permisos (banear,
  expulsar, moderar, gestionar roles, ver canales, enviar mensajes); rol del bot **por encima** de
  los usuarios a moderar (RN-01); snowflakes del servidor y del canal de salida; clave maestra
  definida (RB-03).
- **Pasos**:
  1. Poner el gateway en modo Discord: `Moderacion__Gateway=Discord` (variable de entorno o
     `appsettings` local **no** versionado). Reiniciar. Default del producto: `Simulado`.
  2. En el panel: completar first-run/login (CU-08/CU-09), ir a **Servidores** y **registrar** el
     servidor: snowflake, **token** (se guarda cifrado, RN-14) y snowflake del canal de salida
     (queda **registrado pero inactivo**, CU-10).
  3. **Probar** la configuración (CU-12): login efímero en memoria que verifica token, intents,
     recepción de eventos, permisos, canal de salida y jerarquía (advertencia, no bloquea).
  4. Sin bloqueantes, **Probar y activar**: el servidor pasa a **Activo** y el gestor abre su
     conexión real (CU-13).
  5. **Probar primero en SIMULACIÓN** (RN-09): dejar la política en modo Simulación, generar la
     ráfaga y verificar el reporte `[SIMULACIÓN]` sin accionar a nadie. Recién entonces pasar a
     **Ejecución**.
- **Resultado esperado**: servidor Activo y Conectado; en simulación se reporta sin accionar; en
  ejecución se reporta y acciona en el orden declarado (RN-05).
- **Validación**: la columna **Conexión** muestra `Conectado`; el reporte llega al canal de salida.
- **Errores posibles**: `PRUEBA_TOKEN_INVALIDO`, intents deshabilitados, `PRUEBA_PERMISOS_FALTANTES`,
  `PRUEBA_CANAL_SALIDA_AUSENTE` → corregir en el portal/servidor y volver a probar (RN-16 bloquea la
  activación con faltantes bloqueantes).
- **Fuentes**: `src/DiscordModeradorBot.Servicio/SMOKE-TEST-DISCORD.md`; ADR-13, RN-16.

## RB-05 — Diagnosticar un servidor que no conecta

- **Objetivo**: diagnosticar por qué un servidor registrado no conecta o no acciona.
- **Prerrequisitos**: sesión de administrador en el panel; acceso al portal de Discord del bot.
- **Pasos** (de más probable a menos):
  1. **Estado de conexión** (columna Conexión, CU-13): `Desconectado (DesconectadoTokenInvalido)` →
     el token fue revocado/rotado: re-validar/re-registrar (CU-12); `Desconectado
     (DesconectadoTransitorio)` → caída de red, el SDK reconecta solo (esperar).
  2. **Prueba de configuración** (CU-12): correr **Probar** y leer los chequeos bloqueantes.
  3. **Intents privilegiados**: si la prueba marca intents deshabilitados, habilitar MESSAGE CONTENT
     y SERVER MEMBERS en el portal (Bot → Privileged Gateway Intents) y reintentar.
  4. **Permisos y canal de salida**: verificar banear/expulsar/moderar/gestionar roles y que el bot
     puede escribir en el canal de salida.
  5. **Jerarquía de roles** (RN-01): si acciona pero deja incidentes **No accionable**, subir el rol
     del bot por encima de los usuarios objetivo (advertencia en CU-12, no bloquea la activación).
  6. **Logs**: `journalctl -u discord-moderador-bot -n 100` (systemd) o `docker logs
     discord-bot-moderador` (Docker); el token nunca debe aparecer (si aparece, es un bug, RN-14).
- **Resultado esperado**: identificar el chequeo bloqueante o la causa de la desconexión.
- **Validación**: tras corregir, la prueba pasa sin bloqueantes y la conexión vuelve a `Conectado`.
- **Errores posibles**: confundir una desconexión transitoria (se resuelve sola) con token inválido
  (requiere acción); modo aún en `Simulado` (no abre conexión real — ver RB-04).
- **Fuentes**: `SMOKE-TEST-DISCORD.md §5/§7/§8`; ADR-08/ADR-13; RN-01, RN-16.

## Referencias

- Scripts (fuente de verdad): `../../../../Discord.Bot.Moderador.Core/scripts/servicio/`
- Smoke-test de integración real: `../../../../Discord.Bot.Moderador.Core/src/DiscordModeradorBot.Servicio/SMOKE-TEST-DISCORD.md`
- DevOps (fuente): `../../../../Discord.Bot.Moderador.Core/SDD2.2D/docs/09_devops/`
- Decisiones: [`../04-decisions/README.md`](../04-decisions/README.md) (ADR-02, ADR-05, ADR-07, ADR-13, ADR-14)
- Seguridad: [`../06-security-compliance/threat-model.md`](../06-security-compliance/threat-model.md),
  [`../06-security-compliance/data-protection.md`](../06-security-compliance/data-protection.md)
