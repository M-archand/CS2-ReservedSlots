# External Integrations

**Analysis Date:** 2026-04-14

## Plugin Framework: CounterStrikeSharp (CSS)

**Role:** The entire plugin is a CSS plugin. CSS is not a library — it is the host runtime that loads the DLL into the CS2 server process.

**Integration surface:**
- `BasePlugin` — base class providing `Load()`, `Localizer`, `Logger`, `AddTimer()`, `RegisterListener<>()`
- `BasePluginConfig` / `IPluginConfig<T>` — config lifecycle; CSS auto-generates and deserializes the JSON config
- `[GameEventHandler]` attribute — registers methods as CS2 game event hooks (`EventPlayerConnectFull`, `EventPlayerTeam`, `EventPlayerDisconnect`, `EventRoundStart`)
- `Listeners.OnMapStart` — server lifecycle callback
- `TimerFlags.STOP_ON_MAPCHANGE` — timer behavior tied to map lifecycle

**CSS namespaces used directly:**
- `CounterStrikeSharp.API` — `Server`, `Utilities`
- `CounterStrikeSharp.API.Core` — `BasePlugin`, `CCSPlayerController`, `CEnvInstructorHint`, `GameEventInfo`, `HookResult`, `HookMode`
- `CounterStrikeSharp.API.Modules.Utils` — `CsTeam`, `PlayerConnectedState`
- `CounterStrikeSharp.API.Modules.Admin` — `AdminManager` (flag/permission checks)
- `CounterStrikeSharp.API.Modules.Timers` — `TimerFlags`
- `CounterStrikeSharp.API.Modules.Cvars` — `ConVar` (reads `sv_visiblemaxplayers`)
- `CounterStrikeSharp.API.Core.Attributes.Registration` — `[GameEventHandler]`
- `CounterStrikeSharp.API.ValveConstants.Protobuf` — `NetworkDisconnectionReason` (kick reason codes)

## CS2 Server CVars

**`sv_visiblemaxplayers`:**
- Read via `ConVar.Find("sv_visiblemaxplayers")` on every map start (with a 3-second deferred timer)
- Controls the publicly-advertised max player count; used in Method 3 (overflow mode) as the public slot ceiling
- Value cached in `_cachedVisibleMaxPlayers` (`int?`) to avoid repeated lookups
- Relevant code: `RefreshVisibleMaxPlayers()` in `src/ReservedSlots/ReservedSlots.cs`

**`sv_gameinstructor_disable` / `sv_gameinstructor_enable`:**
- Temporarily enabled server-side via `Server.ExecuteCommand` and per-player via `player.ReplicateConVar` to display the kick countdown hint
- Reset to `false` after the hint expires

## CS2 Admin Flag System

**Provider:** CounterStrikeSharp's `AdminManager`

**How used:**
- `AdminManager.GetPlayerAdminData(player)` — retrieves a player's admin record
- `adminData.GetAllFlags()` — returns the full set of CSS permission flags for that player
- `AdminManager.PlayerHasPermissions(p, "@css/generic")` — checks if a player can see kick notifications
- Flags are configured in the plugin config (`Reserved Flags`, `Admin Reserved Flags`) and matched against CSS flag strings such as `@css/vip`, `@css/ban`, `@css/root`
- CSS admin data is populated by whichever admin backend the server operator configures (flat file, SimpleAdmin, etc.) — this plugin has no direct dependency on that backend

## Optional Integration: CS2 Discord Utilities

**Status:** Optional; not present in this repository's main source file.

**Reference from README:**
- A separate release zip (`ReservedSlots_DU_Support.zip`) supports [CS2-Discord-Utilities](https://github.com/NockyCZ/CS2-Discord-Utilities)
- That variant allows Discord role IDs to be used alongside CSS flags in `Reserved Flags` and `Admin Reserved Flags` config lists
- The standard build in this repo (`ReservedSlots.cs`) does not import Discord Utilities — it uses CSS flags only

## Localization System

**Provider:** CounterStrikeSharp built-in (`Localizer`)

**Files:**
- `lang/en.json` — English (default fallback)
- `lang/lv.json` — Latvian
- `lang/pt-BR.json` — Brazilian Portuguese

**Keys used in code:**
- `Hud.ServerIsFull` — shown to player being kicked via `CEnvInstructorHint` entity
- `Hud.ReservedPlayerJoined` — shown to player being kicked via `CEnvInstructorHint` entity
- `Chat.PlayerWasKicked.ServerIsFull` — broadcast to all players or admins on kick
- `Chat.PlayerWasKicked.ReservedPlayerJoined` — broadcast to all players or admins on kick

**Format:** Standard CSS localization JSON; supports `{0}` positional placeholders and color tags like `{darkred}`, `{default}`

## Logging

**Provider:** `Microsoft.Extensions.Logging` via CSS (`Logger` property inherited from `BasePlugin`)

**Patterns:**
- `Logger.LogInformation(...)` — kicked player records when `Log Kicked Players` config is `true`
- `Logger.LogWarning(...)` — CVar lookup failures, invalid config values, null-player edge cases
- `Console.ForegroundColor` / `Console.WriteLine` — direct console output for startup validation errors (empty flag list)

**Output destination:** Controlled by CSS host (Serilog sinks to console and file by default)

## Data Storage

**Databases:** None — no database connection of any kind.

**File Storage:** No file I/O beyond reading the auto-generated JSON config (handled entirely by CSS).

**In-memory state only:**
- `waitingForSelectTeam` (`List<int>`) — player slots pending team selection before kick decision
- `reservedPlayers` (`Dictionary<int, bool>`) — tracks which connected players have reservation immunity
- `waitingForKick` (`Dictionary<int, KickReason>`) — tracks players pending a delayed kick
- `_cachedVisibleMaxPlayers` (`int?`) — cached `sv_visiblemaxplayers` CVar value

**Persistence:** None — all state is ephemeral; cleared on disconnect events and map change (timer `STOP_ON_MAPCHANGE` flag)

## Webhooks & Callbacks

**Incoming:** None — no HTTP endpoints or external webhooks.

**Outgoing:** None — no HTTP calls, no API calls, no external network I/O.

## CI/CD & Deployment

**CI Pipeline:** None — no GitHub Actions workflows or other CI config files are present in the repository.

**Distribution:**
- Releases published manually as `.zip` files on GitHub Releases
- Deployed by unzipping into `csgo/addons/counterstrikesharp/plugins/` on the game server

## Environment Configuration

**Required env vars:** None — the plugin uses no environment variables.

**Config location on server:** `csgo/addons/counterstrikesharp/configs/plugins/ReservedSlots/ReservedSlots.json`

**Secrets:** None — no API keys, tokens, or credentials of any kind.

---

*Integration audit: 2026-04-14*
