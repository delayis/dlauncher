# How to edit `mod-exceptions.json`

This file controls which "extra" mods (outside the base list in `launcher-manifest.json`) a
player is allowed to keep without the launcher deleting them. **The launcher never downloads
anything from here** — the player installs the `.jar` themselves, and this file just stops it
from being removed.

If you break the JSON (missing comma, unclosed brace...), nothing serious happens: the launcher
validates the file before using it. If it's malformed, it's simply ignored and the launcher keeps
using the last known-good version — it never breaks the game for players. But your change also
won't apply until it's fixed, so it's worth double-checking before you consider the commit done.

## Structure

```json
{
  "global": [
    "xaeros-minimap.jar"
  ],
  "perPlayer": {
    "0369b5d781cd4bc7b46f0216927b4e3f": {
      "lastKnownUsername": "SamsaraMode",
      "mods": ["debug-tool.jar"]
    }
  }
}
```

- **`global`**: list of filenames allowed for **every** player. If you don't want any, leave it
  as `[]` (an empty list) — never delete the key entirely.
- **`perPlayer`**: one entry per player. The key is their **UUID** (no dashes), not their
  username.
  - `lastKnownUsername`: only there so you can tell who's who at a glance. It does nothing — the
    launcher never reads it to match a player, that's always done by UUID.
  - `mods`: list of files allowed only for that specific player, on top of whatever is in
    `global`.

`global` and `perPlayer` are always combined (not either/or): a mod in both lists is simply
allowed either way, nothing breaks.

## How to get a player's UUID

In the launcher's **Account** panel, the player has a **"Copy my UUID"** button. That copies it
straight to their clipboard so they can send it to you (Discord, chat, whatever) to request a mod
exception, either global or per-player.

## Full example (several global mods + several players)

```json
{
  "global": [
    "xaeros-minimap.jar",
    "xaeros-worldmap.jar",
    "appleskin.jar"
  ],
  "perPlayer": {
    "0369b5d781cd4bc7b46f0216927b4e3f": {
      "lastKnownUsername": "SamsaraMode",
      "mods": [
        "debug-tool.jar",
        "jei-1.20.1.jar",
        "worldedit-cui.jar"
      ]
    },
    "e1a2c3d4b5f60718293a4b5c6d7e8f90": {
      "lastKnownUsername": "OtroJugador",
      "mods": [
        "shulkerboxtooltip.jar",
        "inventoryhud.jar",
        "toolstats.jar"
      ]
    },
    "9f8e7d6c5b4a39281706f5e4d3c2b1a0": {
      "lastKnownUsername": "TercerJugador",
      "mods": [
        "mousetweaks.jar",
        "controlling.jar",
        "clienttweaks.jar",
        "betterf3.jar"
      ]
    }
  }
}
```

Here, `SamsaraMode` and `OtroJugador` each have 3 of their own mods plus the 3 global ones (6
total), and `TercerJugador` has 4 of their own plus the 3 global ones (7 total). The UUIDs for
`OtroJugador` and `TercerJugador` are placeholders — swap them for real UUIDs.

## Common mistakes to avoid

- ❌ Using the player's username as the `perPlayer` key instead of their UUID
  (`"SamsaraMode": {...}` doesn't work — it has to be the UUID).
- ❌ Forgetting the comma between two entries in the same object/list.
- ❌ Leaving a trailing comma after the last item (e.g. `["a.jar", "b.jar",]` — a trailing comma
  isn't valid JSON).
- ❌ Forgetting double quotes around filenames (`[xaeros.jar]` instead of `["xaeros.jar"]`).
- ❌ Writing the mod name without `.jar` at the end, or different from the actual file name.
- ❌ Editing this file thinking you also need to touch `launcher-manifest.json` — they're
  independent, no need to duplicate anything there.

## Before you commit: validate the JSON

Paste the content into https://jsonlint.com/ (or any JSON validator) and make sure it doesn't
flag any errors before saving the change on GitHub. It takes less than a minute and avoids your
edit getting silently ignored for being malformed.
