# How the news system works internally (and how to migrate it)

This guide explains how the launcher gathers news items from several separate files, exactly how
the image field works, and what we'd need to change if we ever stop hosting this on GitHub. It's
the technical counterpart to [NEWS-GUIDE.md](NEWS-GUIDE.md), which explains how to publish a news
item without going into this level of detail.

## 1. The idea, in plain terms

Instead of a single file with every news item stacked inside (which would keep growing and be
easy to break when hand-editing), each news item is its own `.json` file inside the `news/`
folder of the repository. On startup, the launcher asks "what files currently exist inside
`news/`?", downloads the content of each one, and merges them all into a single list sorted by
date to display.

## 2. How we do it internally: listing the folder

Unlike `launcher-manifest.json` (a standalone file whose exact URL we already know ahead of
time), with `news/` we don't know how many files exist or what they're named — so we first ask
GitHub's API for the **folder listing** (not a specific file). GitHub responds with something
like:

```json
[
  { "name": "2026-07-10-north-dungeon.es.json", "path": "news/2026-07-10-north-dungeon.es.json", "type": "file" },
  { "name": "2026-06-28-welcome.en.json", "path": "news/2026-06-28-welcome.en.json", "type": "file" }
]
```

With that list, we request the content of each `.json` file one by one (the same way we request
`launcher-manifest.json`), merge them all into one list, and sort them by the `date` field, most
recent first — automatic, without anyone having to place them in any particular order by hand.

If a specific file is malformed, that news item is simply skipped and we continue with the
rest — a syntax error in one shouldn't take down the whole panel.

Relevant code: `fetch_remote_news()` in [news.rs](src-tauri/src/news.rs).

## 3. How the language of each news item is detected

The language doesn't travel inside the JSON content — it's decided **only by the file name**,
looking at the suffix right before `.json` (`detect_lang()` in [news.rs](src-tauri/src/news.rs)
compares that suffix against the fixed `KNOWN_LANGS` list = `es`/`en`/`fr`/`de`/`ru`, the same
codes already used by `src/i18n.ts`). If the name doesn't end in one of those codes, the file gets
dropped right during the listing step, before we even request its content.

We download and cache **every news item in every language at once**, each already tagged with its
detected `lang`. Filtering by whatever language the player has selected in Settings happens
afterward, in the frontend (`main.tsx`, filtering the received `news` array by
`lang === language`) — no need to ask GitHub for anything again when someone switches language,
the change is instant since everything is already downloaded.

## 4. How the image actually works (the part we don't do ourselves)

This is the key difference versus mods: a mod is a file Minecraft needs to have on disk to load
it, so the launcher genuinely **does** have to download it into the mods folder. A news image,
on the other hand, is only ever **displayed on screen** — so the `image` field of each item is
nothing more than a text URL, and that URL gets handed as-is to the launcher's window (which
under the hood is literally an embedded browser). The window does exactly what any webpage would
do when it encounters an `<img src="...">` tag: it requests that URL on its own and displays
whatever it gets back.

In practice this means:

- **Our Rust code never touches the image.** It doesn't download it, doesn't save it to disk,
  doesn't process it. All the image work lives in the `image` field (plain text) + the embedded
  browser.
- **The image can live anywhere public**, without us changing a single line of code: GitHub,
  Imgur, a Discord attachment, a public S3 bucket, your own website... The only requirement is
  that it's a direct URL to an image file (not an HTML page that displays it).
- **Migrating where the images live has nothing to do with migrating where the news JSON
  lives.** These are two completely independent things — we could move the news listing to
  another host and leave the images on GitHub (or the other way around) with no issue at all.

There's one practical exception to "the `image` field is a direct URL, as-is": most people who
upload an image to GitHub copy the regular page link that displays it
(`github.com/.../blob/...`), not the raw URL an `<img>` tag needs to load it. Instead of
requiring people to know how to convert that by hand, `fetch_remote_news()` calls
`normalize_image_url()`, which detects that pattern and rewrites it to `raw.githubusercontent.com`
automatically before caching it. Any URL that isn't a GitHub page is left untouched, since it's
assumed to already be a direct link.

## 5. What depends on GitHub right now (and what doesn't)

- **The images**: don't depend on GitHub at all, they're already host-agnostic (see point 4) —
  the URL can point anywhere right now, without touching code.
- **The listing and content of `news/`**: this IS hardcoded to GitHub's API, in the `NEWS_DIR_URL`
  constant in [news.rs](src-tauri/src/news.rs), plus the per-file URL built inside
  `fetch_remote_news()`. Moving *this* off GitHub does require code changes (see point 7).
- **Language detection**: doesn't depend on GitHub at all — it's a plain text comparison on the
  file name (`detect_lang()`), doesn't matter where the listing comes from.

## 6. What we'd need to think about if we ever leave GitHub

- **Does the new host let you "list a folder"?** This is the most unusual part of this system
  compared to the manifest or mod exceptions (which are standalone files with a fixed URL). An
  S3/R2 bucket supports this (there's a call to "list objects with this prefix"), a custom server
  would need an endpoint that returns that listing by hand. If the new host doesn't offer anything
  like that, the simplest fallback is to keep a small index file (e.g. `news/index.json` with just
  the list of file names) that IS read from a fixed URL, and use it to know which files to
  request next — we'd lose the "just upload the file and you're done" convenience, and would need
  to remember to add its name to the index too, but it works on any host.
- **Caching on the new host**: exactly the same consideration we already ran into with the
  manifest — before assuming a change shows up instantly, check with `curl -I` how long the new
  host caches things (`Cache-Control`/`Age`), both for the listing and for each individual file.
- **The images don't change at all** even if we migrate the news JSON to another host — they keep
  working exactly the same, wherever they live.

## 7. What we'd need to change in the code to switch hosts

Assuming the new host does support listing a folder (the simple case, covers S3/R2 and similar),
in [news.rs](src-tauri/src/news.rs):

- Change the `NEWS_DIR_URL` constant and the per-file URL built inside `fetch_remote_news()`.
- Adapt the parsing of `GithubDirEntry` to match how that host returns its own listing (the
  `name`/`path`/`type` fields are specific to GitHub's response shape).
- Remove the `Accept: application/vnd.github.raw+json` header when requesting each file's
  content — it's specific to GitHub's API, a regular file server doesn't need it.

If the new host does NOT support listing a folder, we'd additionally need to:

- Keep an index file (`news/index.json` or similar) with the list of file names, and change
  `fetch_remote_news()` to read that index first instead of requesting a directory listing from
  the API.

In no case do we need to touch anything in the frontend (`main.tsx`) or how images are
displayed — that part is completely independent of where the data lives.
