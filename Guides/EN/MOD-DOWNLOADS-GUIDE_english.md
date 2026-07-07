# How mod downloads and updates work

This guide explains how we decide whether a mod gets downloaded, updated, or left alone — and
what we'd change if we ever stopped hosting mods/config on GitHub.

## 1. The idea, in plain terms

Every time someone opens the launcher (or hits Play), we check whether the mod we already have
saved on the player's computer is still exactly the same as the one currently out there on the
internet. If it's the same, we do nothing — no point re-downloading something that didn't
change. If it's different, we download it again and replace the old file.

The question is: how do we know it's "the same" without downloading the whole thing every time
just to compare? That's where a small technical trick comes in, explained below in more detail
for whoever's touching the code — but that's the underlying idea, nothing more.

## 2. How we do it under the hood: the ETag

Every mod in `launcher-manifest.json` has a `url`. Every time we check the modpack, we request
that `url` and read a special header that comes back in the response, called an **ETag** — a
kind of fingerprint of the file's content, calculated by the server itself (GitHub, in our case)
without us needing to download anything to get it. We save that fingerprint in `mod_etags.json`,
in the launcher's data folder, one per mod.

The next time we check, we compare the fingerprint we just got against the one we had saved:

- **They match** → the content genuinely didn't change, we download nothing (doesn't matter if
  you re-uploaded the file, deleted and re-added it, etc. — we look at the content, not the
  action).
- **They don't match** (or we didn't have a saved fingerprint yet) → we download. If the file
  **already existed** on the player's disk, we count it as an **update**; if it **didn't
  exist**, as a **new download**. This distinction is what feeds the progress bar ("Updating
  1/3" vs "Downloading 1/7").

Relevant code: `check_modpack()` in [modpack.rs](src-tauri/src/modpack.rs).

## 3. The "we uploaded the same file and nothing got detected" case

This is expected, not a bug. The fingerprint (ETag) depends on the **content itself**
(basically a hash/checksum of the bytes), not on "the act of uploading it". If we delete an
asset on GitHub and upload a file that's **byte-for-byte identical** to the one before, GitHub
computes the exact same fingerprint it had before — because, as far as the file goes, nothing
changed. We're right not to re-download something that didn't change: that would be wasted
traffic and time.

So for us to actually detect a change, the bytes need to differ, not just "re-uploading it". In
practice this happens with:
- An actual code change, however small.
- Recompiling without changing anything: every Gradle build embeds different internal
  timestamps in the `.jar`, so the bytes change either way even with identical code (that's why,
  when testing this, just recompiling — no code changes needed — was enough to produce a
  different fingerprint).

## 4. What depends on GitHub right now (and what doesn't)

There are two completely separate things that currently live on GitHub, and it's worth not
mixing them up:

- **The mods themselves** (`crisisclient-1.0.0.jar`, etc.): where they live is just whatever we
  put in the `url` field of each entry in `launcher-manifest.json`. The code doesn't know or
  care that it's GitHub — **it's already hosting-agnostic**. We could change that URL today to
  any other place (our own server, an S3 bucket, Cloudflare R2, Backblaze...) without touching a
  single line of Rust, as long as that place returns a real `ETag` header.
- **`launcher-manifest.json` and `mod-exceptions.json` themselves** (the base config and the
  exceptions): these ARE hardcoded to the GitHub API, in two constants (`MANIFEST_URL` and
  `MOD_EXCEPTIONS_URL` in [modpack.rs](src-tauri/src/modpack.rs)). Moving *these two files*
  off GitHub would require touching code (see section 6).

## 5. What we'd need to think about if we ever leave GitHub

If down the line we decide to move mods and/or config to another place, here's what would
change underneath:

- **Does the new place return an ETag?** Most object storage (S3, R2, Backblaze, an nginx
  serving static files) does this automatically, computed from the content — our current
  mechanism would keep working exactly the same. If the place does NOT give a reliable ETag
  (some custom servers don't, or base it on modification date instead of content), we'd need to
  switch strategy (see below).
- **Caching on that place itself**: this already happened to us with GitHub —
  `raw.githubusercontent.com` cached for 5 minutes and its CDN ignored our attempts to bypass
  it, so we switched to the GitHub API (~60s cache). With any new place we'd need to check the
  same way: request the URL with `curl -I` and look at the `Cache-Control`/`Age` header before
  assuming changes show up instantly.
- **Authentication**: right now everything is public, no credentials involved. If the new place
  requires some key/token to read the files, that token would have to live inside the
  launcher's `.exe` — which would make it just as "public" as the Microsoft Client ID we talked
  about earlier: anyone could extract it from the binary. It wouldn't work for hosting truly
  private content without building our own backend with per-player authentication, which is a
  much bigger project than this.
- **Without a reliable ETag, alternatives for detecting changes**:
  - `Last-Modified` header instead of `ETag` — works similarly, but is less precise (a date, not
    a hash; two different uploads in the same second wouldn't be told apart).
  - Keeping a hash or version number by hand in the manifest (e.g. adding a `sha256` field per
    mod) and comparing that instead of the ETag — we'd lose the convenience of "nothing to track
    by hand", but it would work with any host, regardless of what headers it gives.

## 6. What we'd need to touch in the code to switch hosts

Assuming we move to a place that **does** give a reliable ETag (the simple case, covers
S3/R2/Backblaze and most static file servers):

1. **For the mods themselves**: nothing in the code — just change the `url` field of each mod
   in `launcher-manifest.json`.
2. **To move `launcher-manifest.json`/`mod-exceptions.json` off GitHub**, in
   [modpack.rs](src-tauri/src/modpack.rs):
   - Change the `MANIFEST_URL` and `MOD_EXCEPTIONS_URL` constants to the new URLs.
   - Remove (or adapt) the `Accept: application/vnd.github.raw+json` header in
     `fetch_remote_manifest()`/`fetch_remote_mod_exceptions()` — it's specific to the GitHub
     API (it asks for the "raw" content instead of base64-wrapped JSON); a regular file server
     doesn't understand or need it, so we'd simply remove that `.header(...)` line.
   - Re-check the caching situation for the new place (previous point) and adjust, if needed,
     how long we tell players to wait after a change.

If instead the new place does **not** give a reliable ETag, we'd additionally need to:
   - Add a manual hash/version field to `ModEntry` in modpack.rs (e.g. `pub sha256:
     Option<String>`), and in `check_modpack()` use that field instead of (or alongside) the
     ETag to decide `up_to_date` — downloading and computing the actual content hash if we need
     to compare against that value.
   - This would add manual upkeep (keeping the hash up to date in the manifest every time a mod
     changes), in exchange for working on any host regardless of its headers.
