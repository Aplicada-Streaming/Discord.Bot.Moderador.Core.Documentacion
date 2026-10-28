---
doc_id: PIECE-SNIP-001
doc_type: code-snippets
title: "Snippets con procedencia — DiscordModeradorBot.Servicio"
version: 0.1.0
status: draft
origin: reverse-engineered
confidence: alta
owner: "equipo-discordmoderador (propuesto)"
last_review: 2026-07-15
review_cycle_days: 180
audience: [desarrolladores, mantenedores]
classification: uso-interno
traces:
  - src/DiscordModeradorBot.Servicio/Aplicacion/MotorDeModeracion.cs
  - src/DiscordModeradorBot.Servicio/Dominio/Configuracion/RegistroDescriptores.cs
  - src/DiscordModeradorBot.Servicio/Dominio/Moderacion/Accion.cs
  - src/DiscordModeradorBot.Servicio/Infraestructura/RegistroDependencias.cs
  - src/DiscordModeradorBot.Servicio/Program.cs
---

# Snippets con procedencia

Corpus curado (Marco §13). Cada bloque cita su origen real (`Fuente: ruta#Lx-Ly`), qué demuestra,
precondiciones y resultado. Los fragmentos son mínimos; las elisiones se marcan con `// …`. Los
símbolos citados existen en el código (fuente de verdad). Las rutas son relativas a la raíz del
repositorio `Discord.Bot.Moderador.Core`.

## a. Uso canónico del pipeline

```csharp
public async Task<Incidente?> ProcesarAsync(MensajeEntrante mensaje, CancellationToken ct = default)
{
    ArgumentNullException.ThrowIfNull(mensaje);

    // Etapa 1 — Descarte de exentos, ANTES de todo lo demás (RN-07, CU-15).
    if (await EsExentoAsync(mensaje, ct))
    {
        // … log; el pipeline termina sin tocar estado ni registrar incidente
        return null;
    }

    // Politicas del servidor del mensaje, por prioridad ascendente (CU-11, RN-04).
    var politicas = (await _cargador.CargarAsync(mensaje.ServidorId, ct))
        .OrderBy(p => p.Prioridad)
        .ToList();
    if (politicas.Count == 0)
    {
        return null;
    }
    // … Etapa 2 contenido, Etapa 3 estado de conducta, Etapa 4 evaluacion + acciones
}
```

- **Fuente:** `src/DiscordModeradorBot.Servicio/Aplicacion/MotorDeModeracion.cs#L137-L167`
- **Demuestra:** el punto de entrada del pipeline de moderación y el orden de las primeras etapas
  (descarte de exentos → carga de políticas por prioridad).
- **Precondiciones:** `MotorDeModeracion` resuelto por DI (scoped); un `MensajeEntrante` con
  `ServidorId`, `UsuarioId`, `CanalId`, `Contenido` e `Instante`.
- **Resultado:** el `Incidente` generado por la política coincidente, o `null` si el mensaje está
  exento, no hay políticas o ninguna coincide.

## b. Punto de extensión: agregar un descriptor de parámetro

```csharp
public static DescriptorParametro<int> UmbralCanalesDistintos { get; } =
    new(
        clave: "umbral-canales-distintos",
        etiqueta: "Umbral de canales distintos",
        leyenda: "Cantidad de canales distintos en los que un mismo usuario debe publicar, " +
                 "dentro de la ventana de detección, para que se marque una ráfaga distribuida. " +
                 "El discriminador es la cantidad de canales distintos, no el volumen de mensajes.",
        valorPorDefecto: 3,
        minimo: 2,
        maximo: 10,
        ejemplos: new[] { 2, 3, 4 });
```

- **Fuente:** `src/DiscordModeradorBot.Servicio/Dominio/Configuracion/RegistroDescriptores.cs#L17-L27`
- **Demuestra:** el patrón para declarar un parámetro configurable como fuente única de verdad
  (default, límites, leyenda, ejemplos). Un descriptor nuevo se agrega como otra propiedad estática
  de `RegistroDescriptores` con el mismo shape.
- **Precondiciones:** el tipo del valor debe ser `struct, IComparable<T>` (restricción de
  `DescriptorParametro<T>`, `DescriptorParametro.cs#L130-L131`).
- **Resultado:** el motor y el panel leen default/límites del descriptor; un valor fuera de rango se
  normaliza vía `NormalizarOPorDefecto` o `NormalizarConTope` en vez de hardcodear el límite
  (ADR-12, RN-10).

## c. Punto de extensión: agregar un `TipoAccion` / acción

```csharp
public enum TipoAccion
{
    Reportar,
    Banear,
    BaneoConBorradoRetroactivo,
    Desbanear,
    Timeout,
    Expulsar,
    AsignarRol,
    QuitarRol,
    ReportarACanalPrivado,
}
```

- **Fuente:** `src/DiscordModeradorBot.Servicio/Dominio/Moderacion/Accion.cs#L13-L26`

El motor despacha cada tipo contra el puerto del adaptador en un `switch`; un `TipoAccion` nuevo se
materializa agregando su rama aquí:

```csharp
switch (accion.Tipo)
{
    case TipoAccion.BaneoConBorradoRetroactivo or TipoAccion.Banear:
        var ventanaBorrado = TimeSpan.FromDays(accion.VentanaBorradoEfectivaDias);
        var resBaneo = await _adaptador.BanearConBorradoAsync(servidor, usuario, ventanaBorrado, ct);
        // … registrar resultado
        return resBaneo;

    case TipoAccion.Timeout:
        var duracion = accion.DuracionTimeoutEfectiva;
        var resTimeout = await _adaptador.AplicarTimeoutAsync(servidor, usuario, duracion, ct);
        // … registrar resultado
        return resTimeout;
    // … Expulsar, AsignarRol/QuitarRol, default
}
```

- **Fuente:** `src/DiscordModeradorBot.Servicio/Aplicacion/MotorDeModeracion.cs#L490-L505`
- **Demuestra:** cómo una acción se ejecuta contra `IAdaptadorGateway`. Para una acción nueva:
  (1) agregar el valor a `TipoAccion`; (2) si es contención sobre el usuario, incluirla en
  `EsContencionSobreUsuario` (`MotorDeModeracion.cs#L412-L414`); (3) añadir su método al puerto
  `IAdaptadorGateway` e implementarlo en ambos adaptadores; (4) agregar la rama en el `switch`.
- **Precondiciones:** la acción llega con su `OrdenEjecucion` y parámetros propios
  (`VentanaBorradoDias`, `DuracionTimeout`, `RolObjetivo`) desde la política.
- **Resultado:** la acción se ejecuta en el orden declarado (RN-05); su `ResultadoAccion` se mapea
  al `ResultadoModeracion` del incidente (una jerarquía superior o permisos faltantes lo dejan
  `NoAccionable` sin caer el pipeline, RN-01).

## d. Composición de dependencias y selección de gateway

```csharp
// Selección del adaptador del gateway por el flag Moderacion:Gateway (ADR-13).
if (modoGateway == ModoGateway.Discord)
{
    services.AddSingleton<AdaptadorGatewayDiscord>();
    services.AddSingleton<IAdaptadorGateway>(sp => sp.GetRequiredService<AdaptadorGatewayDiscord>());
    services.AddSingleton<IFabricaClienteGateway>(sp => sp.GetRequiredService<AdaptadorGatewayDiscord>());
    services.AddSingleton<GestorConexionesGateway>();
    services.AddHostedService<GestorConexionesGatewayHostedService>();
}
else
{
    // Modo SIMULADO (default): adaptador simulado + walking skeleton, sin red ni token.
    services.AddSingleton<AdaptadorGatewaySimulado>();
    services.AddSingleton<IAdaptadorGateway>(sp => sp.GetRequiredService<AdaptadorGatewaySimulado>());
    services.AddHostedService<WalkingSkeletonHostedService>();
}
```

- **Fuente:** `src/DiscordModeradorBot.Servicio/Infraestructura/RegistroDependencias.cs#L57-L76`
- **Demuestra:** cómo el flag `Moderacion:Gateway` decide qué adaptador (`IAdaptadorGateway`) y qué
  servicio en segundo plano se registran, sin ramificar el resto de la composición.
- **Precondiciones:** `AgregarServiciosModeracion(cadenaConexionSqlite, modoGateway)` se invoca
  desde `Program.cs#L52-L54`, donde `modoGateway` se lee de configuración con default `Simulado`.
- **Resultado:** en `Simulado` el bot corre sin red ni token; en `Discord` el gestor abre una
  conexión real por servidor activo con su token descifrado.

## e. Endpoint de autenticación (minimal-API)

```csharp
app.MapPost("/api/auth/salir", async (HttpContext ctx, IAntiforgery antiforgery) =>
{
    // Validación antiforgery del POST de logout (protección CSRF).
    if (!await EsRequestValidaAsync(antiforgery, ctx))
    {
        return Results.Redirect("/ingresar");
    }

    await ctx.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
    return Results.Redirect("/ingresar");
});
```

- **Fuente:** `src/DiscordModeradorBot.Servicio/Program.cs#L231-L241`
- **Demuestra:** el patrón de los endpoints `/api/auth/*`: validación antiforgery del POST del
  formulario Blazor + operación de cookie (aquí `SignOutAsync`) + redirect. Blazor Server
  interactivo no puede setear/limpiar cookies desde un componente, por eso el sign-in/out lo emiten
  estos endpoints minimal-API.
- **Precondiciones:** un formulario del panel con token antiforgery válido que postea a la ruta; el
  middleware de autenticación/antiforgery configurado en `Program.cs`.
- **Resultado:** la cookie de sesión se limpia y se redirige a `/ingresar`. El endpoint hermano
  `/api/auth/ingresar` (`Program.cs#L179-L229`) emite el sign-in con los claims del rol
  `administrador`; `/api/auth/cambiar-contrasena` lleva `.RequireAuthorization()`.

## Referencias

- [`README.md`](./README.md) · [`architecture-local.md`](./architecture-local.md) · [`configuration.md`](./configuration.md)
- Catálogo de endpoints: [`docs/05-apis/catalog.md`](../../05-apis/catalog.md).
- Glosario: [`GLOSSARY.md`](../../../GLOSSARY.md).
</content>
</invoke>
