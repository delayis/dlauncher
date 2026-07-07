# How to edit `launcher-manifest.json`

This is the launcher's base configuration file: Minecraft version, loader, and the list of mods
that get installed for everyone by default. It's downloaded automatically every time someone
opens the launcher or hits "Play" — editing it and pushing to GitHub is enough, **no need to
rebuild or redistribute the launcher** for a change here to reach every player.

Same as `mod-exceptions.json`: if the JSON ends up malformed, the launcher ignores it and keeps
using the last known-good version — it won't break the game for anyone, but your change also
won't apply until it's fixed.

## Structure

```json
{
  "launcherName": "Crisis Launcher",
  "gameDirName": "CrisisLauncher",
  "minecraftVersion": "1.20.1",
  "loader": "forge",
  "loaderVersion": "47.4.20",
  "portableMcVersion": "5.0.4",
  "microsoftClientId": "86f4bf3c-db1f-4695-ab11-4026b9af8cf5",
  "mods": [
    {
      "name": "crisisclient-1.0.0.jar",
      "url": "https://github.com/delayis/dlauncher/releases/download/v.10/crisisclient-1.0.0.jar"
    }
  ]
}
```

- **`launcherName`**: name shown in the status message after checking mods. Cosmetic only.
- **`gameDirName`**: name of the folder holding everything (save data, mods, configs) inside
  Windows' app data folder. **Don't change this lightly**: doing so makes the launcher start
  using a brand-new empty folder instead of the player's existing one — nothing gets deleted, but
  they'd stop seeing their existing progress/config.
- **`minecraftVersion`** / **`loader`** / **`loaderVersion`**: which Minecraft version and which
  loader (Forge or otherwise) gets installed. `loader` accepts `"forge"`, `"fabric"`, `"quilt"` or
  `"neoforge"`.
- **`portableMcVersion`**: recommended PortableMC version, informational only (shows up in the
  status message).
- **`microsoftClientId`**: the Azure app's Client ID for "Sign in with Microsoft". Don't touch it
  unless you know what you're doing — see the earlier thread about the "Invalid app registration"
  error if it ever needs changing.
- **`mods`**: the base list, installed for everyone. Each entry has:
  - `name`: the exact `.jar` filename that should end up in the mods folder.
  - `url`: where to download it from. The launcher compares the ETag (provided automatically by
    GitHub) to know whether it needs to re-download — no need to track a version number by hand,
    the file's content changing is enough.

## ⚠️ The `"server"` field does nothing

You might see (or be tempted to add) a block like this:

```json
"server": { "host": "localhost", "port": 25565 }
```

**The launcher completely ignores it.** The actual server the "Play" button connects to is
configured inside the `crisis-client` mod itself (`ClientConfig.java`, `serverHost` /
`serverPort` fields), not here. Changing this in the manifest has zero effect — if you need to
change which server the game connects to, you have to edit the mod, not this file.

## How to add, update, or remove a mod

- **Add**: append a new entry to `mods` with its `name` and `url`. It'll be downloaded the next
  time anyone opens the launcher.
- **Update** (same mod, new content): upload the new `.jar` to the same URL (same asset name on
  the same release), without touching the manifest. The launcher detects the content changed
  (via the ETag) and re-downloads it on its own.
- **Remove**: delete the entry from `mods`. The next time a player opens the launcher, that
  `.jar` is automatically removed from their mods folder (unless it's protected by
  `mod-exceptions.json` — see that guide).

## How long a change takes to show up

The launcher fetches this file from the GitHub API, which caches for up to ~60 seconds (in
practice it's usually near-instant). No need to wait 5 minutes or restart anything more than
once — just hit "Play".

## Common mistakes to avoid

- ❌ Forgetting the comma between two entries in `mods`.
- ❌ Leaving a trailing comma after the last item.
- ❌ Changing `gameDirName` thinking it's purely cosmetic (see warning above).
- ❌ Using a `url` that isn't a direct HTTPS link to the file (it needs to download the `.jar`
  itself, not an HTML page from GitHub).
- ❌ Editing the `"server"` block expecting it to change which server the game connects to (it
  does nothing, see warning above).

## Before you commit: validate the JSON

Paste the content into https://jsonlint.com/ and make sure it doesn't flag any errors before
saving the change on GitHub.
