# Home Banners

The banner strip at the top of the Home screen is a small content system that lets you publish announcements, curated lists, featured books and seasonal collections to everyone using the app — without shipping an app update. Everything a banner shows (its colours, copy, images, links and the full article behind it) comes from a single JSON file hosted on the web. Edit that file, and the app changes the next time it launches. There is no banner content baked into the app itself; the app is just a renderer for whatever the feed describes.

## How it fits together

When the Home screen first appears, the app asks `BannerProvider` to fetch the feed. The provider makes a single network request to a configured URL, decodes the JSON into an array of `HomeBanner` values, and hands them back. If the URL is empty or the request fails, it returns an empty array and the app simply shows nothing — banners are never required for the app to work.

Those banners are then handed to `BannerSection`, which lays them out as a horizontally-paging row of cards at the very top of Home. Each card is a compact, colourful summary. Tapping one presents `BannerDetailView` as a sheet that slides up — this is the "article", a longer page assembled from a list of content blocks. Images everywhere (card artwork, the hero, and inline article images) are loaded through `CachedImage`, which downloads once, caches the decoded image, and reuses it so things don't flicker or reload as you scroll.

In short: `HomeView` fetches → `BannerProvider` decodes → `BannerSection` shows cards → `BannerDetailView` shows the article. The relevant source files are `Models/HomeBanner.swift`, `Views/Home/BannerSection.swift`, `Views/Home/BannerDetailView.swift`, `Support/CachedImage.swift` and `Support/Color+Hex.swift`.

## The data feed

The feed is a JSON **array**, where each element is one banner. It lives at whatever URL is set in `BannerProvider.endpoint`. Today that points at a raw file in the `micro_shelf_website` GitHub repository, but it can be any HTTPS URL that returns the expected JSON — a static file on a host or CDN, an export from a CMS like Airtable, or a small API. The app neither knows nor cares where the JSON comes from; it only needs the array in the right shape.

To change which banners appear, you edit that file and the change is picked up on the next launch. To change the *source* entirely, you change the one `endpoint` string.

## Anatomy of a banner

Every banner needs four things: a unique `id`, a `title`, a `subtitle`, and a base colour (`tintHex`, a hex string with no leading `#`). The `id` is used internally to tell banners apart and drive the paging dots, so it must be unique within the feed. The title and subtitle appear both on the small card and at the top of the article. The tint colour drives the card's gradient.

Beyond those, every field is optional and there to make a banner richer. A `tag` is a short eyebrow label such as "Collection" or "Book of the Month" that sits above the title in small caps. A `systemImage` is the name of an SF Symbol; it shows as a faint, oversized watermark behind the card's text for texture, and as the fallback icon in the article hero when there's no image. A `gradientHex` is an optional second colour — when present, the card and hero use a diagonal two-colour gradient from `tintHex` to `gradientHex`; when absent, they use a subtle single-colour gradient derived from the tint alone. An `imageURL` is used in two places: as the small tilted artwork on the card, and as the hero image at the top of the article.

Finally, `content` is what makes the article worth opening. It's an ordered list of blocks (described next) that the detail page renders top to bottom. If you don't provide `content`, the app falls back to a single `body` string of plain text — useful for a quick announcement, but `content` is how you build something that reads like a real page.

## The article and its content blocks

The detail page is assembled from the `content` array, and each block declares its `type`. There are four.

A **text** block (`{ "type": "text", "text": "…" }`) is a paragraph. The text supports inline Markdown, so you can use `**bold**`, `*italics*` and `[inline links](https://…)` to format it. Block-level Markdown like lists isn't supported — keep each paragraph in its own text block rather than trying to put multiple paragraphs in one string.

A **cover** block (`{ "type": "cover", "url": "…" }`) displays a book cover. It's rendered at a fixed 2:3 portrait size, centred, with a soft shadow, so it always looks like a book regardless of the exact pixel dimensions of the image you point it at. Use this for book covers.

An **image** block (`{ "type": "image", "url": "…" }`) is a general media image. It's rendered in a fixed 3:2 landscape frame that spans the article's width, with the image filling and centre-cropping that frame. Use this for photographs and wide graphics, not for covers — a portrait cover forced into a landscape frame gets cropped to a slice.

A **button** block (`{ "type": "button", "label": "…", "url": "…" }`) is a tinted call-to-action that opens its URL in the browser. You can have more than one in a row.

The deliberate split between `cover` and `image` exists because a book cover and a landscape photo are fundamentally different shapes, and trying to render both through one frame always compromises one of them. Giving each its own block with fixed, predictable boundaries means the layout stays consistent no matter what image you supply.

## The hero image

The hero is the band at the top of the article. Because a banner's `imageURL` might be a wide image *or* a book cover, the hero is built to handle either without cropping anything important: it shows a blurred, filled copy of the image as the background, and the whole image, un-cropped and centred, on top. For a cover this produces the familiar "album art" look; for a wide image it produces a nicely framed header. If a banner has no `imageURL`, the hero is the gradient with the SF Symbol in the centre instead.

## Colours

Colours are plain hex strings such as `"E0892E"`, converted to real colours by the `Color(hex:)` helper. A banner's gradient is computed from `tintHex` and, optionally, `gradientHex`. This same gradient is reused as the card background and as the hero background when there's no image, so a banner feels visually consistent from card to article.

## Fetching and caching

It's worth being precise about *when* the app reads the feed, because it has caused confusion before. The app fetches the feed **once, on a cold launch**, the first time Home appears — there is no polling, no refresh when the app returns from the background, and no pull-to-refresh. The request is made with a cache policy that ignores the app's own local URLSession cache, so the app itself won't serve you a stale copy.

However, the GitHub raw URL the feed currently lives behind is served through a CDN (Fastly) with a five-minute cache (`Cache-Control: max-age=300`). That means after you commit a change, the raw URL can keep handing out the *previous* version for up to five minutes, even though the file on GitHub is already updated. If you ever need to confirm a change immediately, read the file through the GitHub Contents API (which isn't CDN-cached) rather than the raw URL — the raw URL telling you the old content does not mean your edit didn't land.

## Editing and publishing banners

To add, change or remove a banner, you edit the JSON feed and publish it (for the GitHub setup, that's committing the file). Order in the array is the order banners appear. Removing an element removes the banner; the app shows however many the feed contains, and hides the whole strip if the feed is empty.

The one thing that will silently break everything is **invalid JSON**, and the most common cause is a literal line break inside a string — JSON does not allow real newlines inside a value, so paragraph breaks in a long `body` or `text` must be written as `\n` (and a blank line between paragraphs as `\n\n`). Because the app treats an unparseable feed the same as an empty one, a single malformed string makes *all* the banners vanish with no error. When in doubt, paste the feed into a JSON validator before publishing.

## Quick reference

A complete banner with one of each block type:

```json
{
  "id": "scifi-essentials",
  "tag": "Collection",
  "title": "Sci-Fi That Holds Up",
  "subtitle": "Four essentials for any shelf",
  "systemImage": "moon.stars.fill",
  "imageURL": "https://…/hero.jpg",
  "tintHex": "4B4FE0",
  "gradientHex": "7A2BD6",
  "content": [
    { "type": "text",   "text": "An intro with **bold** and a [link](https://example.com)." },
    { "type": "cover",  "url": "https://…/cover.jpg" },
    { "type": "image",  "url": "https://…/photo.jpg" },
    { "type": "button", "label": "See more", "url": "https://example.com" }
  ]
}
```

| Field                | Required | Notes                                              |
| -------------------- | -------- | -------------------------------------------------- |
| `id`                 | Yes      | Unique within the feed                             |
| `title` / `subtitle` | Yes      | Shown on card and article                          |
| `tintHex`            | Yes      | Hex, no `#`; drives the gradient                   |
| `tag`                | No       | Small eyebrow label                                |
| `systemImage`        | No       | SF Symbol; card watermark / hero fallback          |
| `gradientHex`        | No       | Second gradient colour                             |
| `imageURL`           | No       | Card artwork + article hero                        |
| `body`               | No       | Plain-text fallback if `content` is omitted        |
| `content`            | No       | Ordered blocks: `text`, `cover`, `image`, `button` |
