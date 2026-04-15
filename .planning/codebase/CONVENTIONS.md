# Coding Conventions

**Analysis Date:** 2026-04-14

## Language & Runtime

All code is C# targeting .NET 8.0 (`net8.0`). Implicit usings are enabled (`<ImplicitUsings>enable</ImplicitUsings>`). Nullable reference types are enabled (`<Nullable>enable</Nullable>`). There is a single source file: `src/ReservedSlots/ReservedSlots.cs`.

## Naming Patterns

**Classes:**
- PascalCase: `ReservedSlots`, `ReservedSlotsConfig`

**Public methods:**
- PascalCase: `OnPlayerConnect`, `PerformKick`, `LogMessage`, `SetKickImmunity`, `GetPlayersReservedType`, `PerformKickCheckMethod`

**Private methods:**
- PascalCase for all private methods except one outlier: `getPlayerToKick` uses camelCase while all others (`GetKickDelay`, `GetPublicSlots`, `RefreshVisibleMaxPlayers`, `GetVisibleMaxPlayers`, `HandleOverflowModeJoin`, `SendConsoleMessage`, `GetPlayersCount`, `GetPlayersCountWithReservationFlag`, `ShowKickHint`) use PascalCase.
- **Rule:** Use PascalCase for all private methods. `getPlayerToKick` is a known inconsistency; new code must use PascalCase.

**Public instance fields:**
- camelCase: `waitingForSelectTeam`, `reservedPlayers`, `waitingForKick`

**Private instance fields:**
- Underscore prefix + camelCase: `_cachedVisibleMaxPlayers`

**Config properties:**
- camelCase: `reservedFlags`, `adminFlags`, `reservedSlots`, `reservedSlotsMethod`, `openSlot`, `kickImmunity`, `kickReason`, `kickDelay`, `joiningPlayerKickDelay`, `inServerPlayerKickDelay`, `kickCheckMethod`, `kickType`, `kickPlayersInSpectate`, `logKickedPlayers`, `displayKickedPlayers`
- JSON property names use space-separated title case via `[JsonPropertyName]`, e.g. `"Reserved Flags"`, `"Kick Delay"`

**Enums:**
- PascalCase name, PascalCase members: `KickType`, `KickReason`, `ReservedType`

**Parameters:**
- camelCase: `player`, `reason`, `mapName`, `config`, `type`
- Event parameters use `@event` (prefixed with `@` to avoid C# keyword collision with `event`)

## File & Project Structure

```
src/ReservedSlots/
├── ReservedSlots.cs        # All plugin logic — single file, ~564 lines
├── ReservedSlots.csproj    # Project definition

lang/
├── en.json                 # English localization strings
├── lv.json                 # Latvian localization strings
└── pt-BR.json              # Portuguese (Brazil) localization strings
```

All plugin logic lives in a single `.cs` file. There is no separation into multiple files or namespaces beyond the one `ReservedSlots` namespace.

## Class Organization

Within `src/ReservedSlots/ReservedSlots.cs`, the ordering is:

1. `using` imports (top of file, ungrouped)
2. `namespace ReservedSlots;` (file-scoped)
3. `ReservedSlotsConfig` class with `[JsonPropertyName]`-decorated properties
4. `ReservedSlots` main plugin class containing:
   - Module metadata properties (`ModuleName`, `ModuleAuthor`, `ModuleVersion`)
   - Inner `enum` declarations (`KickType`, `KickReason`, `ReservedType`)
   - Public instance fields (state collections)
   - `OnConfigParsed` lifecycle method
   - `Load` lifecycle method (listener and timer registration)
   - Game event handlers (decorated with `[GameEventHandler]` or `[GameEventHandler(HookMode.Pre)]`)
   - Public helper methods
   - Private helper methods

## Import Organization

All `using` directives appear at the top of the file, ungrouped with no blank lines between them:

```csharp
using CounterStrikeSharp.API;
using CounterStrikeSharp.API.Core;
using CounterStrikeSharp.API.Modules.Utils;
using CounterStrikeSharp.API.Core.Attributes.Registration;
using CounterStrikeSharp.API.Modules.Admin;
using System.Text.Json.Serialization;
using CounterStrikeSharp.API.Modules.Timers;
using Microsoft.Extensions.Logging;
using CounterStrikeSharp.API.Modules.Cvars;
using CounterStrikeSharp.API.ValveConstants.Protobuf;
using System.Drawing;
```

No ordering between BCL (`System.*`) and third-party (`CounterStrikeSharp.*`, `Microsoft.*`) imports.

## Config Pattern

Config is a dedicated class inheriting `BasePluginConfig`. Each property gets `[JsonPropertyName("Human Readable Name")]`. Default values are set inline via property initializers. The plugin implements `IPluginConfig<ReservedSlotsConfig>` and receives the parsed config via `OnConfigParsed`, where out-of-range values are validated with `Logger.LogWarning` and corrected to safe defaults:

```csharp
public class ReservedSlotsConfig : BasePluginConfig
{
    [JsonPropertyName("Reserved Slots")] public int reservedSlots { get; set; } = 1;
    [JsonPropertyName("Kick Delay")]     public int kickDelay { get; set; } = 5;
    // ...
}

public void OnConfigParsed(ReservedSlotsConfig config)
{
    Config = config;
    if (Config.reservedSlotsMethod < 0 || Config.reservedSlotsMethod > 3)
    {
        Logger.LogWarning("[Reserved Slots] Invalid 'Reserved Slots Method' value {Value}...", Config.reservedSlotsMethod);
        Config.reservedSlotsMethod = 0;
    }
    // ...
}
```

## Error Handling

**Null checks on player objects:** Every event handler and method that accesses player data starts with a null + validity guard:

```csharp
var player = @event.Userid;
if (player != null && player.IsValid && player.SteamID.ToString().Length == 17)
{ ... }
```

**SteamID length guard:** Human players are distinguished from bots/HLTV by checking `player.SteamID.ToString().Length == 17`. This check appears in multiple places (`OnPlayerConnect`, `GetPlayersCount`, `GetPlayersCountWithReservationFlag`, `getPlayerToKick`).

**Nullable return types:** Methods that may produce no result return `CCSPlayerController?` and callers always check for null before use.

**Early returns (guard clauses):** Methods bail out at the top on invalid state rather than nesting logic:

```csharp
public void PerformKick(CCSPlayerController? player, KickReason reason)
{
    if (player == null || !player.IsValid)
        return;
    // ...
}
```

**ConVar null guard:** `RefreshVisibleMaxPlayers` handles a missing convar by logging a warning and retaining the previous cached value — it never throws.

## Logging

Two logging approaches coexist:

**1. `Microsoft.Extensions.Logging.ILogger` (preferred):**
Injected via the CounterStrikeSharp base class. Used for all operational messages after startup.

- Warnings use structured message templates with named placeholders:
  ```csharp
  Logger.LogWarning("[Reserved Slots] Invalid 'Kick Reason' value {Value}, falling back to 135.", Config.kickReason);
  ```
- Informational kick messages use string interpolation (inconsistency — should migrate to structured templates):
  ```csharp
  Logger.LogInformation($"Player {name} ({steamid}) was kicked, because the server is full.");
  ```

**2. `SendConsoleMessage(text, ConsoleColor)` (startup only):**
A private static helper that writes directly to `Console` with a color. Used only for pre-logger config errors (e.g., empty `reservedFlags` list) where the ILogger may not yet be ready.

## Control Flow Patterns

**Switch expressions** for concise value-to-value mapping:

```csharp
bool isImmune = Config.kickImmunity switch
{
    1 => type == ReservedType.Admin,
    2 => type == ReservedType.VIP,
    _ => type == ReservedType.Admin || type == ReservedType.VIP
};

int delay = reason switch
{
    KickReason.ServerIsFull when Config.joiningPlayerKickDelay >= 0 => Config.joiningPlayerKickDelay,
    KickReason.ReservedPlayerJoined when Config.inServerPlayerKickDelay >= 0 => Config.inServerPlayerKickDelay,
    _ => Config.kickDelay
};
```

**Switch statements** for multi-case logic with side effects (kick method dispatch, log/chat dispatch, reserved-slots-method dispatch).

**LINQ** is used extensively for player list filtering and counting:

```csharp
Utilities.GetPlayers().Count(p =>
    !p.IsHLTV && !p.IsBot &&
    p.Connected == PlayerConnectedState.PlayerConnected &&
    p.SteamID.ToString().Length == 17)
```

## Timer Usage

Deferred actions use `AddTimer(delay, callback, TimerFlags.STOP_ON_MAPCHANGE)`. The player's `Slot` (int) is captured before the delay and the player reference is re-fetched inside the callback to guard against stale handles:

```csharp
var slot = player.Slot;
waitingForKick.Add(slot, reason);
AddTimer(delay, () =>
{
    var delayedPlayer = Utilities.GetPlayerFromSlot(slot);
    if (delayedPlayer != null && delayedPlayer.IsValid)
        delayedPlayer.Disconnect((NetworkDisconnectionReason)Config.kickReason);

    if (waitingForKick.ContainsKey(slot))
        waitingForKick.Remove(slot);
}, TimerFlags.STOP_ON_MAPCHANGE);
```

`TimerFlags.STOP_ON_MAPCHANGE` is applied to every timer, preventing timers from firing after a map change.

## State Management

Three public mutable collections hold runtime plugin state, all keyed on `player.Slot` (int):

- `waitingForSelectTeam`: `List<int>` — slots awaiting team selection before kick decision
- `reservedPlayers`: `Dictionary<int, bool>` — slots with a reservation flag; `bool` = whether the player is kick-immune
- `waitingForKick`: `Dictionary<int, KickReason>` — slots scheduled for delayed kick with their reason

All three are cleared on `OnMapStart`. `OnPlayerDisconnect` individually removes entries for the disconnecting player's slot.

## HUD / In-Game Notifications

Kick countdown messages use `CEnvInstructorHint` entities spawned via `Utilities.CreateEntityByName<CEnvInstructorHint>("env_instructor_hint")`. The hint is attached to the player's pawn and auto-destroyed after the kick delay. This requires enabling `sv_gameinstructor_disable false` server-side and replicating `sv_gameinstructor_enable true` to the individual player. See `ShowKickHint` in `src/ReservedSlots/ReservedSlots.cs`.

## Localization

Player-facing strings are never hardcoded inline. They are accessed via `Localizer["Key"]` and `Localizer["Key", arg0]`. Keys follow dot-namespaced format:

```
Hud.ServerIsFull
Hud.ReservedPlayerJoined
Chat.PlayerWasKicked.ServerIsFull
Chat.PlayerWasKicked.ReservedPlayerJoined
```

Lang files live in `lang/` at the project root, in JSON format with locale-code filenames (`en.json`, `lv.json`, `pt-BR.json`). When adding new user-facing strings, add to all three lang files.

## Formatting & Style Tools

No `.editorconfig`, `dotnet-format`, or other linter/formatter configuration is present. Code style is maintained manually. No style enforcement in CI.

---

*Convention analysis: 2026-04-14*
