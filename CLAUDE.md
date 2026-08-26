# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **client-only** SPT BepInEx mod: shows a notification at raid start listing which bosses spawned and their
zone (if Intel Center unlocked), a real-time "detected in your vicinity" notification when you get near one, a
rebindable key (`O`, default) to re-show the notification list at any time (useful if a boss has since moved
zones), and a "has been eliminated" notification once a boss dies. Optional 3D floating markers above detected
bosses' heads.

This is a real GitHub-level fork (`gh repo view` reports `isFork: true`, `parent: Mattexe21/BossNotifier`,
itself descended from the original [m-barneto/BossNotifier](https://github.com/m-barneto/BossNotifier)) - not a
fresh repo with copied-in code. Updated to build against this user's current SPT client. See "Porting notes"
below for exactly what changed. Same fork-isolation rule as the TaskItemIndicator fork: only `origin` (this
user's own fork) is configured as a remote, no `upstream` - never push anywhere but `origin`, never open a PR
or issue against `Mattexe21/BossNotifier` or `m-barneto/BossNotifier` without the user explicitly asking.

## SPT version

Client-side companion to this user's other SPT 4.1.3 mods (SkillPointsMod, ItemCountTooltip, TaskItemIndicator
fork, NoMoreKillZones, QuestKeyInfo — see their own `CLAUDE.md` files under `C:\Dev\`). Reference source for
verifying real client APIs: `C:\Dev\spt-reference\client-dlls\Assembly-CSharp.dll`, decompiled at
`C:\Dev\spt-reference\client-dlls\decompiled\`.

## Build

```
dotnet build -p:SPT_PATH="C:\Games\SPT 4.1"
```

Single project, `netstandard2.1` (same as TaskItemIndicator's client project — BepInEx plugins target this, not
`net10.0` like the server-side mods), reads DLL reference paths from the `SPT_PATH` MSBuild property (same
convention as TaskItemIndicator, no default — must be passed on the command line). Output at
`bin\Debug\netstandard2.1\BossNotifier.dll`. No test/lint scripts.

## Porting notes (from the upstream 4.0.11-targeting source)

The upstream `Plugin.cs` turned out to need almost no changes to build clean against this user's actual client
DLL:

- **`NotificationManagerClass` was renamed to `NotificationManager`** somewhere between the client version
  upstream targeted and this one — the only compile error the whole file produced. Confirmed the new name via
  a real, unrelated existing usage already decompiled in this environment
  (`spt-singleplayer\SPT.SinglePlayer.Patches.MainMenu\PluginErrorNotifierPatch.cs`,
  `NotificationManager.DisplayMessageNotification(...)`), not guessed - then a straight rename across all 4
  call sites.
- **Everything else compiled unchanged** against the real `Assembly-CSharp.dll`/`Comfort.dll`/`UnityEngine*.dll`
  in this install: `BossLocationSpawn.Init`/`.ShallSpawn`/`.BossType`/`.BornZone`, the `BotBoss` constructor,
  `GameWorld.OnGameStarted`/`.OnPersonAdd`/`.AllAlivePlayersList`, `Player.OnPlayerDeadOrUnspawn`,
  `ClientAppUtils.GetMainApp().GetClientBackEndSession().Profile.Hideout.Areas`, `EAreaType.IntelligenceCenter`,
  `KeyboardShortcut`/`UnityInput.Current` for the rebindable key, all still present with matching signatures. A
  clean compile against the real DLL is meaningful evidence these symbols are unchanged (a rename or signature
  change would have been a compile error, same as `NotificationManagerClass` was) — but it's not proof the
  *runtime behavior* behind them is identical, only that the shape is. See "Known gaps" below.
- **BepInEx plugin GUID changed** from `Mattexe.BossNotifier` to `com.imperator.bossnotifier`, matching this
  user's other mods' naming (`com.imperator.*`) — deliberate, not an oversight, since this is a new build under
  this user's own repo rather than a literal continuation of the upstream one (no existing install to preserve
  compatibility with; the user didn't have any prior version of this mod installed).
- **The optional Fika multiplayer companion project (`BossNotifier.Fika/`) was not ported.** Upstream ships it
  as a separate assembly that syncs boss-kill state across a Fika co-op lobby via network packets
  (`AllBossesPacket`/`BossDeathPacket`). This user's other mods are all single-player-oriented and nothing in
  this session's context indicates Fika is installed - skipped to avoid building/shipping something unused and
  untestable. The base plugin's `ShouldFunction()`/`FikaIsPlayerHost` soft-dependency check (only runs main
  logic if not a non-host Fika client) was left in as upstream had it — harmless no-op without Fika installed
  (`Type.GetType(...)` returns null, the check short-circuits to always-true), and means Fika support could be
  added back later by just re-adding that companion project without touching this one.

## Architecture

Single file (`Plugin.cs`), same "the whole plugin, one file" structure as the upstream source and this user's
TaskItemIndicator fork:

- **`BossLocationSpawnPatch`** — `[PatchPostfix]` on `BossLocationSpawn.Init`, fires once per boss spawn point
  the game decides will actually spawn (`__instance.ShallSpawn`) as the raid is generated server-side-adjacent
  (this runs client-side but reflects the server's spawn decision). Builds `bossesInRaid: Dictionary<name,
  zone>` — the "who's in this raid and roughly where" data shown at raid start.
- **`BotBossPatch`** — `[PatchPostfix]` on `BotBoss`'s constructor, fires when a boss's AI brain actually
  activates in the world (i.e. real-time, as opposed to the raid-generation-time spawn-point patch above).
  Tracks `spawnedBosses` (for the ✓ "detected" marker) and queues a one-time "X has been detected in your
  vicinity" notification per boss/goon-group.
- **`NewGamePatch`** — `[PatchPrefix]` on `GameWorld.OnGameStarted`, the raid-start hook that creates/inits the
  `BossNotifierMono` MonoBehaviour (everything below lives on that component, not the plugin class itself,
  since it needs Unity lifecycle methods `Update`/`Start`/`OnDestroy`).
- **`BossNotifierMono`** — the actual runtime logic:
  - `GenerateBossNotifications()` builds the message list from `bossesInRaid`/`spawnedBosses`/`deadBosses`,
    grouping Knight/Big Pipe/Birdeye into one "Goons" line, skipping Cultists during daytime (7am-10pm per the
    `IsDay()` check), and gating location text/detection-checkmark behind the configured Intel Center level
    requirements.
  - `Update()` polls the rebindable show-key (`O` default) and marker-toggle key (`P` default) every frame via
    `UnityInput.Current`, drains the vicinity-notification queue, and (if markers enabled) repositions/rescales/
    redraws every tracked boss's floating `TextMesh` marker each frame (billboard-facing the camera, distance
    fade, optional line-of-sight raycast to hide through walls).
  - Boss death is detected via `Player.OnPlayerDeadOrUnspawn` subscribed per-boss the moment a marker is
    created (both from `OnPersonAdd` for bosses that spawn after raid start, and
    `InitializeExistingBossMarkers()` for ones already in the world when this component starts) — triggers
    `GenerateBossNotifications()` again so the elimination message appears without waiting for the next manual
    `O` press.
- **Config** (F12 menu, `BepInEx\config\com.imperator.bossnotifier.cfg`): General (show-key, show-on-raid-start
  toggle), Intel Center unlock levels for plain/location/detection notifications (0-4, 4 = disabled), and Boss
  Markers (enable, toggle key, through-walls, character glyph, name/distance display, size/scale/visibility
  distance, color).

## Known gaps

- **Compiles clean against the real client DLL, but genuinely untested in-game.** Nothing in this environment
  can run the actual game client — the raid-start notification list, zone-name resolution for zones not in the
  hardcoded `zoneNames` table (falls back to a camelCase-splitting heuristic, `GetZoneName`), the vicinity
  detection notification, boss-death detection, the `O`/`P` rebindable keys, and the floating markers are all
  only confirmed via a clean compile (real method/field signatures) plus reading the decompiled source they
  call into, not by observing them run. This is the next thing to test in-game.
- **Not deployed to a Fika co-op setup** — see "Porting notes" above; the base mod's solo-safe fallback means
  it should still function correctly in a Fika lobby as a *client* (each player's own game shows their own boss
  detections/markers), just without the upstream's cross-lobby kill-sync packets.
- **`zoneNames` table is copied as-is from upstream** — not re-verified against this exact game version's real
  zone IDs. If a zone shows up as an unresolved raw ID or an oddly-split camelCase string in a notification,
  that zone's ID probably needs adding to the table (or the game added/renamed zones since upstream's table was
  built) — regenerate/extend the same way it'd have been built originally (real zone IDs from
  `SPT_Data\database\locations\<map>\base.json`), not guessed.
