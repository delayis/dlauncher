# How the launcher's automatic update works

This is different from `launcher-manifest.json`/`mod-exceptions.json`/`news/`: those update
*content* (mods, news) without touching the launcher itself. This updates the **launcher itself**
(the `.exe`) — and unlike what we originally planned (a notice with a manual download button),
this is fully silent: the launcher downloads the new version, installs it, and restarts on its
own, with the player not having to do anything or hunt for any file.

It uses Tauri's official plugin (`tauri-plugin-updater`), which requires cryptographically
signing every build so the launcher can verify an update genuinely came from us and not from
someone impersonating us — that's why there are signing keys involved, explained below.

## The signing keys

Already generated at the project root:

- **`crisis-launcher-update.key`** — the **private** key. Every build gets signed with this.
  **Never upload it anywhere public (not to `dlauncher`, not to any repository, don't share it on
  Discord).** It's already in `.gitignore` to protect it in case this project ever becomes a git
  repository. If you lose it, you won't be able to sign further builds under the same identity —
  you'd have to generate a new key and everyone would need to manually reinstall one last time.
- **`crisis-launcher-update.key.pub`** — the **public** key. This one is safe to share; it's
  already copied into `src-tauri/tauri.conf.json` (`plugins.updater.pubkey`), which is exactly
  what the launcher needs to verify an update is legitimate.

## How it works

On startup, the launcher requests a `latest.json` file (explained below) from the URL configured
in `plugins.updater.endpoints` in `tauri.conf.json`. If it lists a version newer than its own, it
downloads the installer, checks its signature against the public key baked into the launcher, and
if everything checks out, installs it and restarts the launcher on its own — the player just sees
a progress bar while it happens, nothing more.

## Building with signing (automatic)

`3_CREAR_EXE_WINDOWS.bat` already handles this: if it finds `crisis-launcher-update.key` at the
project root, it sets the required environment variables (`TAURI_SIGNING_PRIVATE_KEY` with the
file's contents, `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` empty since the key has no password) before
building — you don't need to do anything special, just keep `crisis-launcher-update.key` at the
project root on whichever machine you build from.

Once it finishes, two new `.sig` files appear alongside the usual installer:

```
target/release/bundle/nsis/Crisis Launcher_0.1.0_x64-setup.exe
target/release/bundle/nsis/Crisis Launcher_0.1.0_x64-setup.exe.sig
target/release/bundle/msi/Crisis Launcher_0.1.0_x64_en-US.msi
target/release/bundle/msi/Crisis Launcher_0.1.0_x64_en-US.msi.sig
```

The `.sig` is the signature of that specific file — if the installer changes (even just
rebuilding with no code changes), you need a fresh `.sig`, an old one won't do.

## The `latest.json` file

This is the file you need to upload to GitHub (unlike the `.sig` files, this one isn't generated
automatically — you write it by hand each time, same as editing `launcher-manifest.json`):

```json
{
  "version": "0.2.0",
  "notes": "Fixed the account menu, added news.",
  "pub_date": "2026-07-08T02:45:00Z",
  "platforms": {
    "windows-x86_64": {
      "signature": "PASTE_THE_.sig_FILE_CONTENTS_HERE",
      "url": "https://github.com/delayis/dlauncher/releases/download/v0.2/Crisis.Launcher_0.2.0_x64-setup.exe"
    }
  }
}
```

- **`version`**: the new version, must match the `Cargo.toml`/`tauri.conf.json` of that build.
- **`notes`**: free text, the only field we could eventually show to the player (the plugin makes
  it available, we just don't display it on screen yet).
- **`pub_date`**: ISO 8601 date, not critical but the plugin expects it.
- **`platforms.windows-x86_64.signature`**: the full contents of the `setup.exe`'s `.sig` file
  (open it with Notepad and copy the whole text, it's one long line).
- **`platforms.windows-x86_64.url`**: the direct download URL for the `setup.exe` — use the
  `setup.exe` (NSIS), not the `.msi`, same reason as in the other guides.

Upload `latest.json` to the root of the `dlauncher` repository. Remember: `tauri.conf.json` points
to `https://raw.githubusercontent.com/delayis/dlauncher/main/latest.json`, so it needs to live
there with exactly that name.

## When shipping a new version

1. Bump the version in `Cargo.toml` and `tauri.conf.json` (they should match).
2. Build with `3_CREAR_EXE_WINDOWS.bat` (signs automatically if the key is present, see above).
3. Create a **new** Release on GitHub (a new tag, e.g. `v0.2`) and upload the `setup.exe` there
   (and the `.msi` too if you want, though the updater doesn't use it). Don't overwrite an old
   Release.
4. Open the `setup.exe`'s `.sig` file with Notepad, copy its contents.
5. Edit (or create) `latest.json` at the repo root with the new version, the new Release's asset
   URL, and the `.sig` contents you copied.

Until you do step 5, nobody sees any update — you can build, sign, and upload the release calmly
before "activating" it for everyone.

## How to test it

Unlike the previous system (where you could just make up a URL to test the button), here the
signature has to exactly match the real file at that URL — the launcher rejects any update whose
signature doesn't check out. For a real end-to-end test:

1. Install the current version (`0.1.0`) on your machine from the signed `setup.exe` we generated.
2. Bump the version to `0.1.1` in `Cargo.toml`/`tauri.conf.json`, build again, upload that new
   Release and its matching `latest.json`.
3. Open the **installed** launcher (not the one you just built, the one that lives in Program
   Files) — it should detect `0.1.1`, download it, install it, and restart on its own.

## What depends on GitHub (and how to migrate)

The `url` field in `latest.json` is a direct link, host-agnostic just like mods/news — it can
point to a GitHub Release or anywhere else without touching code. The only thing tied to GitHub is
the `plugins.updater.endpoints` URL in `tauri.conf.json` — to move it, just change that URL to
wherever you host `latest.json` in the future. The signing keys (public/private) don't depend on
GitHub at all, they're independent of wherever you host the files.
