# Architecture

**Analysis Date:** 2026-04-14

## Pattern Overview

**Overall:** Single-class CounterStrikeSharp plugin — event-driven, hook-based architecture

**Key Characteristics:**
- One plugin class (`ReservedSlots`) handles all logic; no layering or service separation
- All state lives in instance fields on the plugin class
- Behavior is driven entirely by CS2 game events via registered hooks
- Config is a separate POCO (`ReservedSlotsConfig`) deserialized by the framework
- Localisation is handled by the CSS framework's built-in `Localizer`

## Classes

**`ReservedSlotsConfig` (`BasePluginConfig`):**
- Purpose: Strongly-typed, JSON-backed configuration POCO
- Location: `src/ReservedSlots/ReservedSlots.cs` (lines 15–32)
- Fields map 1:1 to JSON keys via `[JsonPropertyName]` attributes
- Validated and normalised in `OnConfigParsed`; invalid values are logged and reset to defaults
- Depends on: nothing — pure data

**`ReservedSlots` (`BasePlugin`, `IPluginConfig<ReservedSlotsConfig>`):**
- Purpose: Plugin entry point and sole business-logic container
- Location: `src/ReservedSlots/ReservedSlots.cs` (lines 34–563)
- Owns all runtime state, registers all event handlers, and executes all kick logic

## Enumerations

All three enums are nested inside `ReservedSlots` in `src/ReservedSlots/ReservedSlots.cs`:

| Enum | Values | Purpose |
|------|--------|---------|
| `KickType` | Random, HighestPing, HighestScore, LowestScore, HighestTime | Controls which public player is selected for removal |
| `KickReason` | ServerIsFull, ReservedPlayerJoined | Distinguishes kick context for messaging and delay selection |
| `ReservedType` | VIP, Admin, None | Classification of the connecting player |

## Runtime State

Three mutable collections maintained on the plugin instance (all keyed by **player slot** integer):

| Field | Type | Purpose |
|-------|------|---------|
| `waitingForSelectTeam` | `List<int>` | Slots of reserved players who have not yet chosen a team; kick-victim selection deferred until team pick |
| `reservedPlayers` | `Dictionary<int, bool>` | slot → isImmune; tracks all connected reserved/admin players |
| `waitingForKick` | `Dictionary<int, KickReason>` | slot → reason; players with a pending delayed kick; prevents double-kick scheduling |
| `_cachedVisibleMaxPlayers` | `int?` | Cached value of `sv_visiblemaxplayers` convar; refreshed on map start |

All collections are cleared on `OnMapStart`.

## Plugin Lifecycle

```
Framework startup
  └─ OnConfigParsed(config)     called before Load(); validates all config values,
                                falls back to defaults with LogWarning on invalid input

  └─ Load(hotReload)
       ├─ RegisterListener<OnMapStart>
       │    → clear all state collections
       │    → AddTimer(3.0f) → RefreshVisibleMaxPlayers()
       └─ RefreshVisibleMaxPlayers()   (also called immediately at Load time)

Game event handlers (registered via [GameEventHandler] attribute):
  EventPlayerConnectFull  →  OnPlayerConnect    (default/Post hook)
  EventPlayerTeam         →  OnPlayerTeam       (Pre hook)
  EventPlayerDisconnect   →  OnPlayerDisconnect (Pre hook)
  EventRoundStart         →  OnRoundStart       (default/Post hook)
```

## Event Flow

### Player connects (`OnPlayerConnect` — `EventPlayerConnectFull`)

```
EventPlayerConnectFull fires
  → guard: IsValid, SteamID is 17 chars, flags configured
  → GetPlayersReservedType(player) → ReservedType (Admin / VIP / None)
  → if VIP or Admin: SetKickImmunity(player, type)
        → adds slot to reservedPlayers with isImmune bool per kickImmunity config
  → branch on Config.reservedSlotsMethod:
      0 (default): kick if GetPlayersCount() >= maxPlayers
      1:           kick if GetPlayersCount() > maxPlayers - reservedSlots
      2:           kick if (GetPlayersCount() - GetPlayersCountWithReservationFlag()) > maxPlayers - reservedSlots
      3:           HandleOverflowModeJoin (uses sv_visiblemaxplayers as ceiling)
  → for public (None) players:  PerformKick(player, ServerIsFull)
  → for VIP/Admin players:      PerformKickCheckMethod(player)
        → kickCheckMethod 0: getPlayerToKick() immediately → PerformKick(victim)
        → kickCheckMethod 1: add player.Slot to waitingForSelectTeam (defer to team pick)
```

### Deferred kick via team selection (`OnPlayerTeam` — Pre hook)

```
EventPlayerTeam fires (Pre)
  → if player.Slot in waitingForSelectTeam:
      → remove slot from list
      → getPlayerToKick(reservedPlayer) → victim
      → PerformKick(victim, ReservedPlayerJoined)
```

### Reserved player disconnects before team select (`OnPlayerDisconnect` — Pre hook)

```
EventPlayerDisconnect fires (Pre)
  → if slot in waitingForSelectTeam:
      → still find and kick a public player
        (reserved player freed a slot but was "in-flight"; public player still needs removal)
  → if slot in waitingForKick: remove entry (player already leaving — cancel pending kick)
  → if slot in reservedPlayers: remove entry
```

### Round start (`OnRoundStart`)

```
EventRoundStart fires
  → if _cachedVisibleMaxPlayers == null: RefreshVisibleMaxPlayers()
```

## Kick Pipeline

```
PerformKick(player, reason)
  → guard: null / !IsValid — return early
  → snapshot name + steamid (player may disconnect before timer fires)
  → GetKickDelay(reason)
        priority 1: joiningPlayerKickDelay  (if reason == ServerIsFull and value >= 0)
        priority 2: inServerPlayerKickDelay (if reason == ReservedPlayerJoined and value >= 0)
        fallback:   Config.kickDelay
  → if delay > 1 (delayed path):
        → if slot already in waitingForKick: return (idempotency guard)
        → add slot to waitingForKick
        → ShowKickHint(player, reason, delay)  — env_instructor_hint entity on pawn
        → AddTimer(delay, STOP_ON_MAPCHANGE):
              → re-resolve player by slot: Utilities.GetPlayerFromSlot(slot)
              → if still valid: Disconnect(NetworkDisconnectionReason)
                                LogMessage(name, steamid, reason)
              → remove slot from waitingForKick
  → if delay <= 1 (immediate path):
        → player.Disconnect(NetworkDisconnectionReason)
        → LogMessage(name, steamid, reason)
```

## Victim Selection (`getPlayerToKick`)

Candidate pool built from `Utilities.GetPlayers()` filtered to:
- Not bot, not HLTV
- `Connected == PlayerConnected`
- SteamID is 17 chars (human player)
- Not the joining reserved player (the `client` parameter)
- Not already in `waitingForKick`
- Not in `reservedPlayers` with `isImmune == true`

Spectator-priority: if `kickPlayersInSpectate == true` and any spectators exist, non-spectators are removed from the pool.

Selection by `Config.kickType`:
- `0` Random — shuffle via `Guid.NewGuid()`
- `1` HighestPing — sort descending by `Ping`
- `2` HighestScore — sort descending by `Score`
- `3` LowestScore — sort ascending by `Score`
- `4` HighestTime — enum value exists, no sort case; falls through to random default

## HUD Notification (`ShowKickHint`)

Used to warn the player being kicked during the delay window:

1. Validates player pawn is valid
2. Enables game instructor system: `sv_gameinstructor_disable false` + `ReplicateConVar` on client
3. Schedules 0.25 s timer to allow pawn to settle
4. Creates `CEnvInstructorHint` entity via `Utilities.CreateEntityByName`, sets caption, timeout, icon, color
5. Calls `hint.AcceptInput("ShowHint", pawn, pawn)`
6. Schedules `delay + 0.25 s` timer to `Kill` the hint entity and revert `sv_gameinstructor_enable` on client

## Immunity Model

`reservedPlayers` dict: `slot → bool (isImmune)` — populated on connect, cleaned on disconnect.

| `kickImmunity` config | Who is immune to being selected as kick victim |
|-----------------------|-----------------------------------------------|
| 0 (default) | Both Admin and VIP players |
| 1 | Admin only |
| 2 | VIP only |

Players with `isImmune == false` are tracked in `reservedPlayers` but can still be chosen as kick victims.

## Reserved Slot Methods

| `reservedSlotsMethod` | Behavior |
|-----------------------|---------- |
| 0 (default) | Server always keeps 1 slot open; VIP triggers victim kick at max, non-VIP kicked if server full |
| 1 | N slots hard-reserved; non-VIP blocked when count > `maxPlayers - reservedSlots` |
| 2 | Same as 1 but reserved players are excluded from occupancy count |
| 3 | Overflow: uses `sv_visiblemaxplayers` as the public ceiling; VIP/Admin can push beyond it |

## Player Count Methods

| Method | What it counts |
|--------|---------------|
| `GetPlayersCount()` | All real connected non-bot/HLTV players (static) |
| `GetPlayersCountWithReservationFlag()` | Subset above who have any reserved or admin flag |

## Configuration Validation

`OnConfigParsed` validates and auto-corrects:
- `kickReason` must be a defined `NetworkDisconnectionReason` value → fallback: `135`
- `reservedSlotsMethod` ∈ [0, 3] → fallback: `0`
- `kickImmunity` ∈ [0, 2] → fallback: `0`
- `kickType` must be a defined `KickType` enum value → fallback: `0`
- `displayKickedPlayers` ∈ [0, 2] → fallback: `2`

## Cross-Cutting Concerns

**Logging:**
- `Logger.LogInformation` / `Logger.LogWarning` via CSS-injected `ILogger`
- `SendConsoleMessage()` (static) for colored console output during config error paths only
- Kicked-player events broadcast via `Server.PrintToChatAll` (all) or per-admin `p.PrintToChat` (admins with `@css/generic`)

**Localisation:**
- `Localizer["key"]` and `Localizer["key", arg]` — CSS built-in backed by `lang/*.json`
- Keys: `Hud.ServerIsFull`, `Hud.ReservedPlayerJoined`, `Chat.PlayerWasKicked.ServerIsFull`, `Chat.PlayerWasKicked.ReservedPlayerJoined`

**Permission checks:**
- `AdminManager.GetPlayerAdminData(player).GetAllFlags()` to determine `ReservedType`
- `AdminManager.PlayerHasPermissions(p, "@css/generic")` to identify admin chat recipients
- Numeric SteamID strings in `adminFlags`/`reservedFlags` are filtered out via `ulong.TryParse` guard

---

*Architecture analysis: 2026-04-14*
