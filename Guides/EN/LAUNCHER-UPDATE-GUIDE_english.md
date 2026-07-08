# How to notify players of a launcher update

This is different from `launcher-manifest.json`/`mod-exceptions.json`/`news/`: those update
*content* (mods, news) without touching the launcher itself. This notifies players when the
**launcher itself** (the `.exe`) has a newer version available — but it doesn't install it
automatically, it only shows a notice with a button to download it. We chose this on purpose
instead of a silent automatic update: this way there's no need to sign every build with a
cryptographic key or maintain that infrastructure, at the cost of the player having to click
"Download" and run the installer themselves (same as now).

## How it works

On startup, the launcher requests `launcher-version.json` from GitHub (same pattern as the
manifest: GitHub's contents API, ~60s cache) and compares that version against its own (the one
compiled into the `.exe`, which can't accidentally get out of sync). If GitHub's version is
newer, a notice appears at the very top of the launcher with a "Download" button that opens that
URL in the browser. If there's no update, or if the check fails (no internet, file not created
yet...), nothing shows up at all — it never blocks or delays startup.

Relevant code: `check_for_update()` in [update_check.rs](src-tauri/src/update_check.rs).

## Structure of `launcher-version.json`

This file sits standalone at the root of the repository, same as `launcher-manifest.json`:

```json
{
  "version": "0.2.0",
  "url": "https://github.com/delayis/dlauncher/releases/download/v0.2.0/Crisis%20Launcher_0.2.0_x64-setup.exe",
  "notes": "Fixed the account menu, added news."
}
```

- **`version`**: the new version, in `x.y.z` format (should match what you set in
  `src-tauri/Cargo.toml` and `src-tauri/tauri.conf.json` when building that version). Compared
  number by number (`0.10.0` is newer than `0.9.0`), not as plain text.
- **`url`**: direct download link for the new installer — usually the `setup.exe` or `.msi`
  asset you upload to a GitHub Release.
- **`notes`**: free text, not shown in the launcher yet (we're just keeping it in case we want to
  display it later), but still useful as a changelog record.

## When shipping a new version

1. Bump the version in `Cargo.toml` and `tauri.conf.json` (they should match).
2. Build with `3_CREAR_EXE_WINDOWS.bat`.
3. Upload the installer (`setup.exe` and/or `.msi`) as an asset on a new GitHub Release.
4. Edit `launcher-version.json` with the new version and that asset's URL.

Until you do step 4, nobody sees any notice — you can build and upload the release calmly before
"activating" the notice for everyone.

## What depends on GitHub (and how to migrate)

Same as with news: the `url` field in `launcher-version.json` is a direct link, completely
agnostic of where it's hosted — it can point to a GitHub Release, your own server, or anywhere
else, without touching code. The only thing tied to GitHub is how `launcher-version.json` itself
gets requested, in the `LAUNCHER_VERSION_URL` constant in
[update_check.rs](src-tauri/src/update_check.rs) — to move it elsewhere, change that constant and
remove the `Accept: application/vnd.github.raw+json` header (specific to GitHub's API), exactly
as explained for the manifest and news in their respective migration guides.
