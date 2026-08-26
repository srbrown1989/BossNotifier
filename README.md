# BossNotifier

A client mod that notifies you which bosses are present in your raid.

> This fork updates [Mattexe21/BossNotifier](https://github.com/Mattexe21/BossNotifier) (itself an update of
> the original [m-barneto/BossNotifier](https://github.com/m-barneto/BossNotifier)) to build against a newer
> SPT client. See `CLAUDE.md` for exactly what changed in the port.

## Features

- 🔔 Notification at raid start showing which bosses spawned
- 📍 Shows boss spawn locations (if Intel Center unlocked)
- ✓ Real-time detection when you get near a boss
- 🌙 Cultist notifications only appear at night (7 PM - 7 AM)
- ⌨️ Press 'O' to show boss notifications again (can change by F12 menu)
  - Press F12 in-game to access BepInEx configuration:
    - Keyboard shortcut (default: O)
    - Show notifications on raid start
    - Intel Center level requirements 

## Installation

1. Build from source (see below)
2. Drop `BossNotifier.dll` into `BepInEx\plugins\`
3. Start the game

## Build

```
dotnet build -p:SPT_PATH="C:\Games\SPT 4.1"
```

Reads DLL reference paths from the `SPT_PATH` MSBuild property (pass it on the command line, or set as an env
var) - no default. The optional `BossNotifier.Fika/` companion project (co-op kill-sync via Fika) is present in
this repo as shipped upstream but isn't part of the default build target here; see `CLAUDE.md` for why.

## Credits

- Original mod by [Mattdokn](https://github.com/m-barneto/BossNotifier)
- Updated for SPT 4.0.11 by [Mattexe21](https://github.com/Mattexe21/BossNotifier)
- This fork: updated to build against a newer SPT client

## License

MIT License
