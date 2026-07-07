# How to add news to the launcher

Each news item is its own `.json` file inside the `news/` folder of the repository — not one
giant file with all of them stacked inside. To publish a new item, just upload a new file to that
folder; to edit an existing one, edit its file; to remove one, delete the file. The launcher
downloads it on its own, the same way it does `launcher-manifest.json` — **no rebuild or
redistribution needed** for a new item to reach every player.

If any file inside `news/` ends up malformed, that specific item just doesn't show up — it
doesn't break the other news items or the rest of the launcher.

## How to name and upload the file

Upload the file inside `news/`, with whatever name you want (for example
`news/2026-07-10-north-dungeon.json`) — the file name doesn't matter at all, it's just so you can
keep things organized in the repo. Only the content inside matters.

## Structure of each news item

```json
{
  "id": "north-dungeon",
  "date": "2026-07-10",
  "title": "New dungeon added",
  "image": "https://raw.githubusercontent.com/delayis/dlauncher/main/news/images/dungeon.jpg",
  "summary": "We've added a new dungeon north of spawn, with bosses and exclusive loot.",
  "body": "Full text shown when the news item is opened, can be quite a bit longer than the summary."
}
```

- **`id`**: unique identifier for the news item, internal only (never shown). Just needs to not
  repeat across news items.
- **`date`**: in `YYYY-MM-DD` format (e.g. `2026-07-10`). Besides being what's displayed, **it's
  used to sort news items automatically** (see below) — if it doesn't follow that format, the
  order can come out wrong.
- **`title`**: title, shown both in the quick preview and when the item is opened.
- **`image`**: the full URL of the cover image. See the section below on exactly how this works.
- **`summary`**: short text for the quick preview (1-2 lines, gets cut off with "…" if longer).
- **`body`**: the full text shown when you click and open the news item.

## The order is now automatic

Unlike how it used to work (a hand-written list where you had to remember to place the new item
at the top), the launcher now gathers every item in `news/` and sorts them itself by the `date`
field, most recent first. It doesn't matter where you upload it or in what order — only that the
`date` field is correct.

## How the image actually works

The `image` field is a full, direct URL to the image — the launcher **does not download the
image or process it in any way**: it just hands that URL as-is to the launcher's window, which
loads it the same way any webpage loads an `<img>` tag. That means:

- The image can live literally anywhere: in the same `news/` folder of the repo (like in the
  example above), on Imgur, as a Discord attachment, on your own website... any public, direct
  URL to an image file works.
- If you upload the image to the same GitHub repository, the URL you need is the
  `raw.githubusercontent.com` one (click the image inside GitHub → "Download" or "Raw" button →
  copy that URL), not the regular GitHub page link that displays it.
- Since the image is requested directly by the launcher's window (it doesn't go through our own
  code), a change to the image shows up as fast as whoever is serving it allows — if you use
  GitHub raw, it can take up to 5 minutes to refresh due to its own cache, unlike the ~60 seconds
  for the rest of the data (see the migration guide for more detail on this).

## What image size fits perfectly

The image is shown in two places with different shapes:

- **Quick preview (card)**: a 320×180 pixel box — **16:9** ratio.
- **Full view (when the news item is opened)**: up to 460 wide × 220 tall at most — a ratio
  slightly wider than 16:9.

In both cases, if you upload an image that isn't exactly that ratio, it doesn't get cropped or
stretched — it's shown in full, with a dark bar filling the leftover space (like the letterboxing
on a widescreen movie). But if you want it to fit with no bars at all in the quick preview, use an
image in **16:9 ratio**, which is exactly the ratio of a normal Minecraft screenshot (1920×1080,
2560×1440, etc. are all 16:9) — it doesn't need to be that exact size, any 16:9 image fits equally
well since it gets scaled.

## Compress the image before uploading it

Even if you upload a 2560×1440 screenshot, the launcher never displays it at that real size — it
shows it inside a small box (320×180 in the quick preview). That means you're making people
download a multi-MB file just to show it tiny, which only makes it take longer to load.

Before uploading a screenshot, resize it with any image editor (Paint, Photoshop, or a website
like https://squoosh.app/) down to around **1280 pixels wide** (keeping the 16:9 ratio, so it
ends up around 1280×720). It looks exactly the same inside the launcher, but the file is much
lighter.

## Quick checklist before publishing a news item

- ✅ Image in 16:9 ratio (or as close as possible).
- ✅ Image resized to ~1280px wide before uploading, not the original full-resolution screenshot.
- ✅ `date` in the correct `YYYY-MM-DD` format (the order depends on it, you no longer need to
  place it anywhere by hand).
- ✅ `id` different from any other existing news item.
- ✅ `image` is the direct (raw) URL to the file, not the link to the page that displays it.
