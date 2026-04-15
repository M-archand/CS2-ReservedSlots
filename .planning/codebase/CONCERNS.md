# Codebase Concerns

**Analysis Date:** 2026-04-14

---

## Tech Debt

**SteamID validation via string length repeated in four places:**
- Issue: `p.SteamID.ToString().Length == 17` appears independently in `OnPlayerConnect` (line 178), `GetPlayersCount` (line 492), `GetPlayersCountWithReservationFlag` (line 501), and `getPlayerToKick` (line 449). There is no shared helper.
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 178, 449, 492, 501
- Impact: Any change to the validity heuristic (e.g., if the CSS API normalizes SteamID representation) must be applied in four separate places; missing one causes silently inconsistent player counts.
- Fix approach: Extract `IsHumanPlayer(CCSPlayerController p)` that consolidates `!p.IsBot && !p.IsHLTV && p.Connected == PlayerConnectedState.PlayerConnected && p.SteamID.ToString().Length == 17`, and replace all four call sites.

**`openSlot` boolean condition missing parentheses in methods 1 and 2:**
- Issue: `(Config.openSlot && GetPlayersCount() >= maxPlayers) || !Config.openSlot && GetPlayersCount() > maxPlayers` — the right-hand side of `||` has no parentheses around `!Config.openSlot && ...`. C# precedence makes this correct, but it reads as though `!Config.openSlot` could bind to the left group.
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 195, 208
- Impact: Readability / future maintainer confusion; no current behavior bug.
- Fix approach: Add explicit parentheses: `(!Config.openSlot && GetPlayersCount() > maxPlayers)`.

**`reservedPlayers` cache abandoned in `GetPlayersCountWithReservationFlag`:**
- Issue: `reservedPlayers` was introduced to cache VIP/Admin status per slot. Commit `9c5bf5a` changed `GetPlayersCountWithReservationFlag` to call `GetPlayersReservedType(p)` directly for all connected players instead of reading the cache, making the cache only useful for immunity checks. The original intent of the cache is now split across two different mechanisms with no clear ownership.
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 63, 267–279, 497–505
- Impact: Increased per-connect cost for method 2 (O(n × flags)); cache value is now authoritative for immunity but not for counting, making the dictionary's contract unclear.
- Fix approach: Either remove the cache and always use `GetPlayersReservedType`, or restore its use in `GetPlayersCountWithReservationFlag` and document the invariant.

---

## Known Bugs

**`HighestTime` kick type declared but never implemented:**
- Symptoms: `KickType.HighestTime` (enum value 4) passes config validation (`Enum.IsDefined`) but has no `case` in the `switch` inside `getPlayerToKick`. It silently falls through to `default` (random).
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 40–47, 462–484
- Trigger: Set `"Kick Type": 4` in config.
- Workaround: Falls back to random kick with no error; advertised behavior is not delivered.

**`Server.ExecuteCommand("sv_gameinstructor_disable false")` affects all players globally:**
- Symptoms: `ShowKickHint` enables the game instructor system-wide via a server console command. The per-player cleanup only disables it for the kicked player via `player.ReplicateConVar`. Other players who had it disabled (e.g., by another plugin) will have it re-enabled and it is never restored globally.
- Files: `src/ReservedSlots/ReservedSlots.cs` line 517
- Trigger: Any kick with `delay > 1`.
- Workaround: None currently; the global state change is a side effect of every delayed kick.

**`CEnvInstructorHint` entity may leak if player disconnects between nested timers:**
- Symptoms: `ShowKickHint` schedules entity creation in a 0.25 s timer. The cleanup timer fires at `delay + 0.25f`. If the player disconnects after the inner entity is created but before the cleanup timer fires during a map change, `TimerFlags.STOP_ON_MAPCHANGE` prevents the cleanup timer from running, leaving the hint entity unreleased.
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 520–555
- Trigger: Map changes while a delayed kick is in progress.
- Workaround: The `hint.IsValid` guard at line 549 provides partial safety, but the timer simply won't execute on map change.

**Hot reload does not rebuild `reservedPlayers` for connected players:**
- Symptoms: On a plugin hot reload (without a map change), `reservedPlayers` starts empty. All currently connected VIP/Admin players lose their kick immunity record until they reconnect or trigger a new connect event.
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 105–120, 267–279
- Trigger: Plugin hot reload while players are connected.
- Workaround: Force a map change after any hot reload.

---

## Security Considerations

**No rate limiting on kick-triggering connects:**
- Risk: A player with a reservation flag can repeatedly connect and disconnect to kick a different public player each time. There is no cooldown, deduplication, or counter per reserved SteamID.
- Files: `src/ReservedSlots/ReservedSlots.cs` `OnPlayerConnect` handler
- Current mitigation: None.
- Recommendations: Track the last kick-trigger timestamp per reserved SteamID and skip kick selection if a kick was triggered within a configurable window (e.g., 30 s).

---

## Performance Bottlenecks

**`Utilities.GetPlayers()` called up to three times per connect event:**
- Problem: `OnPlayerConnect` for method 2 can invoke `GetPlayersCount()`, `GetPlayersCountWithReservationFlag()`, and then `getPlayerToKick()` sequentially, each calling `Utilities.GetPlayers()` and iterating the full list independently.
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 183–230, 488–505, 447
- Cause: No shared player list snapshot within a single handler invocation.
- Improvement path: Capture `var allPlayers = Utilities.GetPlayers()` once at the start of `OnPlayerConnect` and pass it into each helper, or compute counts lazily from a single snapshot.

---

## Fragile Areas

**`waitingForSelectTeam` and `waitingForKick` use slot integers; slots are engine-reused:**
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 62–64
- Why fragile: Player slots (0–63) are reused by the engine after a player disconnects. If the disconnect event fires after a new player occupies the same slot, the new player could inherit a stale `waitingForSelectTeam` or `waitingForKick` entry. `OnPlayerDisconnect` cleans up on disconnect, but the ordering between "new player connects to slot N" and "old player's disconnect event fires for slot N" is not guaranteed in all edge cases.
- Safe modification: When adding to either dictionary in `OnPlayerConnect`, explicitly remove any pre-existing entry for that slot before inserting.
- Test coverage: None.

**`getPlayerToKick` calls `FirstOrDefault().player` on a value-tuple list:**
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 462–484
- Why fragile: `playersList.FirstOrDefault()` on `List<(CCSPlayerController player, int, int, CsTeam)>` returns the zero-value tuple (with `player == null`) if the list is empty. This is guarded by the `if (!playersList.Any()) return null` at line 459, but if that guard is ever removed, the method silently returns `null` instead of throwing, and callers may not check.
- Safe modification: Use `playersList.Count > 0` with an indexed access or explicit null pattern rather than relying on the `default` value of a value tuple.
- Test coverage: None.

**`_cachedVisibleMaxPlayers` is null during the first 3 s of each map:**
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 66, 107–120, 419–438
- Why fragile: `OnMapStart` clears state and schedules `RefreshVisibleMaxPlayers` in a 3 s timer. Any player connecting during this window falls back to `maxPlayers - Config.reservedSlots` in `GetPublicSlots`. For method 3 this may miscalculate the overflow threshold. The `OnRoundStart` fallback helps but does not cover the window before the first round start.
- Safe modification: Consider calling `RefreshVisibleMaxPlayers()` synchronously once in `Load` (prior behavior before commit `03948f1`) as an initial seed before the map-start timer can fire.

---

## Scaling Limits

**`reservedSlots` not validated against `maxPlayers`:**
- Current capacity: Any integer is accepted.
- Limit: If `reservedSlots >= maxPlayers`, `GetPublicSlots` clamps to 0 and all public slots are treated as unavailable, but no warning is emitted. The plugin appears to work while silently refusing all non-reserved players.
- Scaling path: Add a check in `OnConfigParsed` that warns (via `Logger.LogWarning`) if `reservedSlots >= Server.MaxPlayers`.

---

## Dependencies at Risk

**`CounterStrikeSharp.API` pinned to exact version `1.0.365`:**
- Risk: CounterStrikeSharp receives frequent breaking updates with CS2 game patches. An exact version pin means the plugin stops loading whenever the server admin updates CSS to a newer version.
- Files: `src/ReservedSlots/ReservedSlots.csproj` line 10
- Impact: Plugin becomes non-functional after any CSS major/minor version bump.
- Migration plan: Switch to a floating version (`1.0.*`) or document clearly that the csproj must be updated alongside CSS.

**`NetworkDisconnectionReason` is a Valve internal protobuf enum:**
- Risk: This enum is not part of the stable CSS public API and can be renumbered or renamed in Valve game updates. Config value 135 is used as a magic number default.
- Files: `src/ReservedSlots/ReservedSlots.cs` lines 74–78, 305, 316
- Impact: Compilation or runtime failure after a CSS update that restructures the enum.
- Migration plan: Define a named constant for value 135 and add a comment linking to the CSS docs URL referenced in the README.

---

## Missing Critical Features

**`Joining Player Kick Delay` and `In-Server Player Kick Delay` are undocumented:**
- Problem: Config fields `joiningPlayerKickDelay` and `inServerPlayerKickDelay` (lines 24–25) are functional but absent from the README configuration table. Operators have no documented way to know these options exist.
- Blocks: Operators cannot configure per-reason kick delays without reading source code.

**No runtime config reload command:**
- Problem: No admin command exists to reload configuration without a full server restart or plugin reload.
- Blocks: Live adjustment of reservation settings without interrupting gameplay.

---

## Test Coverage Gaps

**Zero automated tests:**
- What's not tested: All kick decision logic, player counting for each method (0–3), `GetPlayersReservedType` flag resolution, kick type ordering (all 5 `KickType` values), `GetKickDelay` routing, `GetPublicSlots` clamping, `openSlot` threshold calculations, hot reload behavior, spectate-first filtering.
- Files: `src/ReservedSlots/ReservedSlots.cs` (entire file).
- Risk: All validation is manual on a live server. The git history contains at minimum five reactive bug-fix commits (`0831146`, `0e713f1`, `03948f1`, `e657730`, `9c5bf5a`, `cfa6181`) addressing regressions — a pattern consistent with the absence of a test suite.
- Priority: High. This plugin controls who is kicked from a live game server. A wrong condition is immediately player-visible and disruptive.

---

*Concerns audit: 2026-04-14*
