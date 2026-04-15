# Testing Patterns

**Analysis Date:** 2026-04-14

## Test Coverage

There are no tests in this codebase.

No test files, no test projects, no test framework configuration. This is typical for CounterStrikeSharp game server plugins, which execute inside a live CS2 server process and depend on game engine APIs (`Utilities.GetPlayers()`, `Server.MaxPlayers`, `ConVar.Find()`, event hooks, timers) that have no offline mock or stub infrastructure.

## Test Framework

None configured.

No `*.Test.csproj`, no `*.Tests.csproj`, no NUnit/xUnit/MSTest package references. The single project file `src/ReservedSlots/ReservedSlots.csproj` has one dependency:

```xml
<PackageReference Include="CounterStrikeSharp.API" Version="1.0.365" />
```

## How to Build

```bash
cd src/ReservedSlots
dotnet build
```

Output: `src/ReservedSlots/bin/Debug/net8.0/ReservedSlots.dll`

## How to Test (Manual / Live Server)

Testing is done by deploying the compiled `.dll` to a running CS2 dedicated server:

1. `dotnet build` inside `src/ReservedSlots/`
2. Copy `bin/Debug/net8.0/ReservedSlots.dll` to `csgo/addons/counterstrikesharp/plugins/ReservedSlots/`
3. Copy `lang/` directory into the same plugin folder
4. Place or auto-generate `ReservedSlots.json` config at `configs/plugins/ReservedSlots/`
5. Restart the server or hot-reload the plugin via the CSS console

Verification is done in-game: observe kick behavior as the server fills, join with and without reservation flags, and watch server console log output.

## What Would Need Mocking to Unit Test

The following CounterStrikeSharp API surface is called directly with no abstraction layer. Any future unit test project would need to wrap or mock all of these:

| Dependency | Used In | File Location |
|---|---|---|
| `Utilities.GetPlayers()` | `GetPlayersCount()`, `GetPlayersCountWithReservationFlag()`, `getPlayerToKick()`, `LogMessage()` | `ReservedSlots.cs` |
| `Server.MaxPlayers` | `OnPlayerConnect()` | `ReservedSlots.cs` |
| `AdminManager.GetPlayerAdminData(player)` | `GetPlayersReservedType()` | `ReservedSlots.cs` |
| `AdminManager.PlayerHasPermissions(p, flag)` | `LogMessage()` | `ReservedSlots.cs` |
| `ConVar.Find("sv_visiblemaxplayers")` | `RefreshVisibleMaxPlayers()` | `ReservedSlots.cs` |
| `player.Disconnect(reason)` | `PerformKick()` | `ReservedSlots.cs` |
| `AddTimer(delay, callback, flags)` | `PerformKick()`, `Load()`, `ShowKickHint()` | `ReservedSlots.cs` |
| `RegisterListener<T>()` | `Load()` | `ReservedSlots.cs` |
| `Utilities.CreateEntityByName<CEnvInstructorHint>()` | `ShowKickHint()` | `ReservedSlots.cs` |

## Testable Logic (No Mocking Required)

These methods contain pure logic that could be exercised by unit tests with minimal or no mocking:

- `GetKickDelay(KickReason)` — switch expression on config values and a reason enum; no external calls
- `GetPublicSlots(int maxPlayers)` — arithmetic on `maxPlayers`, `Config.reservedSlots`, and the cached `_cachedVisibleMaxPlayers`
- `SetKickImmunity(player, type)` — switch expression on `Config.kickImmunity` int; only touches the `reservedPlayers` dictionary
- `HandleOverflowModeJoin(...)` — threshold calculation; delegates to `GetPublicSlots` and `GetPlayersCount`
- `GetPlayersReservedType(player)` — flag-matching logic; testable if `AdminManager` is injected or wrapped

## Test Coverage Gaps

All functionality is untested. The highest-risk untested areas:

**Slot-counting comparisons (`GetPlayersCount`, `GetPlayersCountWithReservationFlag`):**
- File: `src/ReservedSlots/ReservedSlots.cs` (lines ~452–505)
- Risk: Off-by-one errors in the four `reservedSlotsMethod` branches could allow players who should be kicked to join, or kick players who should not be kicked.

**`getPlayerToKick` selection logic:**
- File: `src/ReservedSlots/ReservedSlots.cs` (lines ~445–486)
- Risk: Spectator-priority filtering combined with sort-by-metric can interact unexpectedly. The random fallback uses `Guid.NewGuid()` ordering, which is non-deterministic and untestable as-is.

**`OnPlayerConnect` branching (`reservedSlotsMethod` 0–3):**
- File: `src/ReservedSlots/ReservedSlots.cs` (lines ~188–232)
- Risk: Four reservation methods with `openSlot` interactions. A regression in one method can silently break another.

**`PerformKick` timer/delay path:**
- File: `src/ReservedSlots/ReservedSlots.cs` (lines ~303–340)
- Risk: The player re-fetch after delay (`Utilities.GetPlayerFromSlot(slot)`) guards against stale references, but the guard path is never exercised by tests.

**`GetPlayersReservedType` flag matching:**
- File: `src/ReservedSlots/ReservedSlots.cs` (lines ~234–265)
- Risk: The `ulong.TryParse` filter that skips numeric-looking flag strings could silently drop valid flags if the input format is unexpected.

**`OnConfigParsed` validation:**
- File: `src/ReservedSlots/ReservedSlots.cs` (lines ~68–103)
- Risk: Invalid config values are corrected to defaults with a warning; no test verifies fallback values are applied correctly.

## CI/CD

No CI pipeline is configured. No `.github/workflows/`, no `azure-pipelines.yml`, no `Makefile`. Builds and deployments are performed manually.

---

*Testing analysis: 2026-04-14*
