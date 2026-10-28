---
doc_id: DATA-ACC-001
doc_type: access-policies
title: Políticas de acceso a datos — persistencia embebida
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: "equipo-discordmoderador (propuesto)"
last_review: 2026-07-15
review_cycle_days: 120
audience: [dev, qa, arquitectura, agentes-automaticos]
classification: uso-interno
traces: [RN-14, RN-13, ADR-02, ADR-03, ADR-06, ADR-07, RC-07, RN-11, CU-05, CU-06, CU-07]
---

# Políticas de acceso a datos — persistencia embebida

> Glosario canónico: [`../../GLOSSARY.md`](../../GLOSSARY.md). Diccionario:
> [`data-dictionary.md`](data-dictionary.md). Gap: documentado desde el código, sin acceso a una
> base viva ni cadena de conexión.

## 1. Modelo de acceso

- **Motor:** base relacional **embebida (SQLite) en archivo local**, en modo **WAL** (write-ahead
  logging) (ADR-02). No hay servidor de base de datos, ni escucha de red, ni puerto expuesto.
- **Proceso único:** un solo proceso del servicio `DiscordModeradorBot.Servicio` abre el archivo.
  El acceso es **in-process** vía EF Core; el panel (Blazor Server), el worker de moderación y la
  API comparten el mismo `ContextoPersistencia` dentro del proceso. No hay clientes externos.
- **No hay roles ni usuarios de base de datos.** SQLite no tiene autenticación ni cuentas por rol;
  la autorización es del **sistema operativo** sobre el archivo (permisos del usuario que corre el
  servicio) y de la **aplicación** (el administrador único autenticado, RN-12/RN-13). El aislamiento
  entre servidores moderados es lógico, por `SnowflakeServidor`, no por roles de BD.
- **No multi-tenant.** Una instancia, una base, un administrador (RC-06). Cada `Servidor` es un
  contexto del firewall multi-contexto; las dependientes se aíslan por su snowflake.

## 2. Credenciales de base de datos

**No hay credenciales de base de datos en el repositorio.** Al ser un archivo local embebido, no
existe usuario/contraseña de conexión ni cadena con secreto: la cadena de conexión apunta a la ruta
del archivo. La protección del archivo es responsabilidad del SO y del empaquetado (servicio del
sistema con su usuario dedicado). Las credenciales que sí existen son de **aplicación** y viven
resguardadas (ver §3): el hash del administrador y el token de bot cifrado.

## 3. Datos sensibles en reposo y su protección

| Dato | Ubicación | Protección | Autoriza / obtiene | Traza |
| --- | --- | --- | --- | --- |
| Token de bot | `Servidor.TokenCifrado` | **Cifrado simétrico AES** en reposo; se descifra solo en memoria para operar | Clave maestra externa a la base (ver §3.1) | RN-14, RC-07, ADR-07 |
| Contraseña del administrador | `Administrador.ResguardoPassword` | **Hash PHC (PBKDF2)** no reversible; nunca en claro | La fija el administrador en el primer ingreso (CU-08) | RN-13, ADR-03 |

### 3.1 Clave maestra del cifrado del token

- La **clave maestra vive fuera de la base de datos**, provista por **variable de entorno** al
  proceso (ADR-07). No se persiste en la base ni en el repositorio.
- **Este documento no contiene el secreto** ni su valor: solo describe el mecanismo. La clave la
  aprovisiona el operador del despliegue (quien instala el servicio del sistema) en el archivo de
  entorno del servicio, con permisos restringidos al usuario del servicio.
- **Rotación / rollback:** el rollback de versión debe **preservar la clave maestra**; con la clave
  intacta, los tokens cifrados siguen siendo válidos. Un cambio de clave exige recifrar los tokens.
- **Consecuencia si se viola (RN-14):** un token nunca se persiste ni se reporta en claro; su
  filtración con la base pero sin la clave maestra no otorga control del bot. El descifrado ocurre
  solo en memoria, al conectar o al probar la configuración (CU-10, CU-12, CU-13).

## 4. Superficie de acceso a datos personales

- El **panel** expone evidencia de incidentes (contenido copiado, nombres de usuario/canal) solo al
  administrador autenticado (RN-12), para revisión (CU-06) y reversión (CU-07).
- El **canal de salida** designado (CU-05) recibe reportes de incidentes; es un canal privado del
  servidor, no un almacén adicional de la base.
- No hay export masivo ni endpoint de lectura anónima de la base.

## 5. Retención y minimización (ADR-06, Ley 25.326)

- El sistema trata identificadores de Discord y **copias de mensajes de usuarios como evidencia**
  (`MensajeAccionado.ContenidoCopiado`, `NombreUsuario`, `NombreCanal`; `Incidente.UsuarioId`). Es
  dato personal bajo la Ley argentina **25.326**; la decisión de cumplimiento es **ADR-06**
  (minimización y retención conforme al marco legal).
- **Borrado:** eliminar un `Incidente` **cascadea** a su evidencia (`MensajeAccionado`,
  `CanalAfectado`) en la misma unidad confirmada (RN-11), lo que da un mecanismo de purga por
  incidente.
- **Gap declarado:** el esquema/código **no** implementa TTL, expiración ni purga automática de la
  evidencia; la conservación es indefinida hasta borrado manual. La política de retención concreta
  (plazo, disparador) está pendiente de definición operativa bajo ADR-06 y no está codificada.
- **Minimización:** solo se copia el contenido necesario como evidencia de la intervención; el token
  y la contraseña nunca se guardan en claro (§3).

## 6. Respaldo y durabilidad

- WAL mejora concurrencia y durabilidad ante corte del proceso (ADR-02). El estado de conducta
  (ventanas de actividad y antirrebote) **no se persiste**: vive en memoria y se pierde al reiniciar
  (por diseño). El respaldo del archivo de base es responsabilidad de operaciones y debe cubrir el
  archivo principal más los sidecar de WAL. Gap: no hay política de backup codificada en el repo.
