---
name: fix-core-web-vitals
description: Fix the image side of a page's Core Web Vitals with Iminify. Scans the live page, reads its image report (the largest contentful paint, oversized images, missing width and height, lazy loading in the wrong place), finds each image in the project and fixes the markup and the files, with the user's approval. Use when the user asks to fix or improve LCP, CLS, layout shift, Core Web Vitals, PageSpeed or Lighthouse results caused by images, or slow-loading images on a page. Not for compressing a folder of images (the optimize-images skill) or writing alt text (the image-seo skill).
---

# Fix image-related Core Web Vitals

Images are the usual cause of a slow largest contentful paint (LCP) and of a page that jumps while it loads (layout shift, CLS). This skill has Iminify open the page in a real browser, read what it reports about each image, then finds those images in the user's project and fixes them: the attributes on the tags, and the files that are far larger than the space they fill. Iminify works on the user's own account, through the `iminify` MCP server this plugin connects.

It changes the user's code and files, so it shows the plan and waits for a yes before it changes anything.

## 0. Check the connection and the allowance

The Iminify tools are `get_usage`, `scan_page` and `get_scan` (and on Pro `crawl_site` and `get_crawl` for several pages), plus `create_upload_link`, `compress_images` and `get_image` for resizing files. If they're missing, or a call says the server needs authentication, stop and tell the user how to sign in:

- Claude Code: run `/mcp`, pick `plugin:iminify:iminify`, and press Approve on the Iminify page that opens.
- Cursor: open Cursor Settings, then MCP, and press Connect next to `iminify`.

Signing in needs an Iminify account with a verified email address. The free plan works.

Then call `get_usage` and note:

- `usage.scans.remaining`: page scans left today (`null` means the plan has no daily cap);
- `usage.crawls.remaining` and `limits.pages_per_crawl`: site crawls left today and how many pages one takes, 0 on a plan without them (Pro has them);
- `limits.images_per_scan`: how many of a page's images a scan downloads and compresses;
- `usage.images.remaining`: images left today, which resizing a file spends;
- `limits.max_file_size`: the largest file the plan takes, in bytes;
- `plan`: on `pro`, an image shown at very different widths can get a set of responsive sizes (step 6).

## 1. Get the page's address

The scan runs on Iminify's servers, so it needs the page as it's published: the live site or a preview deployment. It can't reach `localhost` or any other address on a private network, and a page behind a login usually comes back with no images. Ask the user for the address when they haven't given one.

Each page is one scan. For several pages, take the ones the user names, or the home page and one page of each kind (a product page, an article). On a plan with site crawls left today, crawl them in one go (step 2) and say it uses one of today's crawls; otherwise scan them one by one and say how many of today's scans that uses.

## 2. Scan the page

Call `scan_page` with the `url`, `format: "auto"` and `alt_text: false`:

- `format: "auto"` makes each image's report say what it would weigh in every format that can hold it (`format_sizes`), which is how you know whether a conversion is worth suggesting.
- `alt_text: false` leaves out AI alt text, which isn't a speed fix and would spend the user's allowance. When the user wants alt text too, leave the argument out instead, and afterwards continue with the `image-seo` skill's "A live page" from its item 3, with this scan's id, so the page isn't scanned twice.

The images a scan finds don't count against the user's daily images, and the compressed copies stay in their account.

Poll `get_scan` a few seconds apart until its `status` is no longer `queued` or `scanning` and `images_in_progress` is 0 (`alt_texts_in_progress` too, when alt text is on). A status of `failed`, `cancelled` or `no-image-found` ends it: report the status, and stop for that page.

For several pages of one site on a plan with site crawls, call `crawl_site` instead, once, with the addresses in `pages` (all on one site, up to `limits.pages_per_crawl`) and the same `format` and `alt_text`. An image the pages share is then compressed once. Poll `get_crawl` until its `status` is `completed` or `failed` and both `pages_in_progress` and `images_in_progress` are 0; a failed crawl's `note` says why. Each entry in its `pages` is a page scan: carry on from step 3 with each page's `id`, and treat a page whose `status` is `failed` or `no-image-found` as above. `audit` on the crawl sums up the site, and `images_to_fix` on each page says where the work is.

## 3. Read the report

Call `get_scan` once more with `audit_images: true`. The report is in `audit`:

- `rendered`: when it's `false`, only the page's HTML could be read, so nothing about how the images are shown is known. Say so and stop for that page.
- `largest_paint`: the biggest thing in the first screen. When its `kind` is `image`, that's the LCP image: its `url`, and its `element` tells an `img` from a CSS background (`div`, `section` and the like).
- `findings`: how many images have each finding. `oversized_bytes` is about how much the oversized images would shed, and `bytes` what a browser downloaded for them all.
- `images`: every image address on the page, in the page's order, each with its `findings`, its file's size (`natural_width`, `natural_height`), the size it's shown at (`rendered_width`, `rendered_height`), `above_fold`, `wasted_bytes`, and `format_sizes`.

The scan measures the page at a desktop width, before anything scrolls, so "the first screen" in the report is a desktop's. On a phone, a different image can be the largest paint, and images can move above or below the first screen. Fix what the report found and tell the user it describes the desktop layout.

The findings and what fixes each:

| Finding | What it means | The fix (step 6) |
|---|---|---|
| `lcp_not_prioritized` | The LCP image is an `img` fetched without priority (a CSS background never gets this finding: see step 4) | `fetchpriority="high"` on it |
| `lazy_above_fold` | An image in the first screen has `loading="lazy"`, which delays it | Remove the lazy loading |
| `oversized` | The file is more than twice as wide and twice as tall as its space | Resize the file, or fix `sizes` |
| `without_dimensions` | No `width` and `height` attributes, so the page moves when it arrives | Add both |
| `eager_below_fold` | An image below the first screen loads straight away | `loading="lazy"` |
| `missing_alt` | No alt text (not a speed problem) | Offer the `image-seo` skill |

## 4. Find each image in the project

For every image with a finding, and for the LCP image whatever its findings (the one `largest_paint` names when its `kind` is `image`), find the file and the tag behind its address:

1. Take the file name from the address's path and search the project for it: the folders the site serves files from (`public`, `static`, `assets` and the like) for the file, and the templates, components, stylesheets and content for the place that shows it. A build tool may add a hash to the name (`hero.4f2a9c1b.jpg` comes from `hero.jpg`): match the name without it. A query string such as `?v=76796a38` only refreshes caches, so leave it out of the search.
2. An address with sizing in it (`?w=1200`, `/_next/image?url=...`, an image CDN's transformation) is made at request time. Its size comes from the component's props or the address's parameters, not from the file, so fix those.
3. An image from another host that the project doesn't hold (a CMS, a third-party widget, an ad) can't be fixed here. List it for the user.

When an address matches several files, or none, list it for the user rather than guessing.

## 5. Show the plan and wait for a yes

List the changes, most important first: the LCP image (a CSS background one always gets a preload, step 6), then the lazy loading in the first screen, then the oversized files by `wasted_bytes`, then the missing dimensions, then the lazy loading lower down. For each, name the file, the place, and the exact change. Say which images you couldn't place and why, and how many of today's images the resizes will use. Change only what the user approves.

## 6. Make the fixes

**The LCP image** (`lcp_not_prioritized`). Add `fetchpriority="high"` to its `img` tag, and make sure it has no `loading="lazy"`. Give it to that one image only: priority on several images makes none of them stand out. When `largest_paint.element` isn't `img`, the image is a CSS background (or a video's poster), which no attribute reaches and no finding names: unless the page's `<head>` already preloads it, add `<link rel="preload" as="image" href="..." fetchpriority="high">` there, with the address the page uses, and the same `media` condition as the stylesheet rule when the background changes with the screen size. When a component renders the tag, set this through the component's own prop for it: read the component, or its documentation, rather than guessing the name.

**Lazy loading in the first screen** (`lazy_above_fold`). Remove `loading="lazy"`. A lazy-loading script that swaps `data-src` into `src` also delays the image: give the tag a real `src` and leave the script out of it.

**Oversized files** (`oversized`).

- `rendered_width` is the desktop layout's. Read the stylesheet before choosing a size: a grid that turns into one column on a phone shows the image wider there, and the image has to fit the widest layout it's shown in.
- When the tag has a `srcset`, the browser picked a candidate too large for the space. The usual cause is a `sizes` attribute that's missing or too wide: without one, the browser assumes the image fills the whole window. Write `sizes` with a width for each layout (`(max-width: 640px) 100vw, 300px`), and check that `srcset` has a candidate close to twice the desktop slot: when its smallest is far larger, the fix also needs a smaller file in `srcset`.
- Otherwise, resize the file so both its sides stay at least twice the space it fills, which still looks sharp on high-density screens. The target width is the larger of `rendered_width` × 2 and `rendered_height` × 2 × `natural_width` ÷ `natural_height`: the second one matters when the space crops the image (`object-fit: cover`), so a 1600 x 900 photo in a 300 x 300 square needs 1067 pixels, not 600. Only shrink, never enlarge: skip any file whose `natural_width` is already at or below the target. Skip one that would barely change too: when `wasted_bytes` is a small share of the image's `bytes` (under a tenth, say), the file is only just past the line, so name it in the report and leave it. First search the project for every place that shows the same file and size it for the widest one. An image that stretches with the window, such as a full-width banner, is shown wider on large monitors than the scan measured: keep it larger, or ask.
- On Pro, an `img` the layouts show at very different widths (a banner that fills a phone screen and a desktop window, a grid that turns into one column) does better with a set of responsive sizes than with one file sized for the widest: offer it in the plan. Send the file through the same upload as a resize, and call `compress_images` with `responsive_widths` instead of `width`: twice each layout's slot width, narrowest first (a 360 pixel phone slot and a 600 pixel desktop slot ask for `[720, 1200]`), at most eight, and `alt_text: false`. A width wider than the file comes back at the file's own width. Poll `get_image` until its `status` is `failed` or `cancelled` (leave the file, and report it with the `hint`), or until `responsive.status` is no longer `pending`. When that's `ready`, download each file in `responsive.files` from its `download_url` and save it beside the original under its `name`, then give the tag a `srcset` naming each file with its width (`hero-720w.jpg 720w, hero-1200w.jpg 1200w`), with the same folder path the tag's `src` uses, and a `sizes` with each layout's slot; `responsive.html` shows the shape. Keep the original as the `src`, or write the optimized copy back over it with the `optimize-images` skill's step 5. The sizes come in the file's own format, so the `srcset` matches the `src`. When the status is `failed` or `skipped` (an animation), resize the file as above instead. A set counts as one of today's images, like a resize.
- Iminify reads `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`, `.tif` and `.tiff` up to `max_file_size`. List any other file (an AVIF, an SVG) instead of resizing it. When the image is a `<picture>`, each `<source>` holds the same picture in another file: resize each one Iminify can read, and list the rest.
- Resize through Iminify with the `optimize-images` skill in this plugin (load it if it isn't loaded): its step 2 to make sure the original can come back, its step 4 to upload the file and call `compress_images` with `width` (alone, so the height follows) and `alt_text: false`, and its step 5 to write the result back over the file. Each resize counts as one of today's images. The `width` and `height` attributes a tag already has keep their ratio, so leave them.

**Missing dimensions** (`without_dimensions`). Add `width` and `height` attributes with the file's own size in pixels (`natural_width` and `natural_height`, or the new size after a resize). The browser uses their ratio to hold the space before the image arrives. When a stylesheet sets the image's width (`width: 100%`, `max-width: 100%`) but not its height, the attribute's height wins and stretches the image: make sure the rule also says `height: auto`.

**Lazy loading lower down** (`eager_below_fold`). Add `loading="lazy"`, never to the LCP image.

**Formats.** When `format_sizes` shows another format much smaller than what the page serves, mention it with the numbers. Converting gives the file a new name, so do it only when the user asks, with the `optimize-images` skill's step 6.

## 7. Check it after the release

The fixes reach the page only once the user publishes them. When they have, scan the page again (one more of today's scans, or crawl the pages again) and compare the `findings`, and each fixed image's `bytes`, with the first scan's: a `sizes` change can clear `oversized` while the browser still downloads as much as before. Core Web Vitals scores themselves come from real visitors' browsers over the following weeks (Search Console, PageSpeed Insights' field data), so don't promise a score: say which image problems the page no longer has.

## 8. Report

Finish with a table per page: each change, its file and place, for a resized file its size before and after, and for a responsive set the files it added. Under it, list what was left alone and why: an image on another host, an address you couldn't place, a file already small enough, a change the user declined. Give the scan's `oversized_bytes`, and its `saved_bytes` and `saved_percent` from `get_scan`: what compressing the page's images would save with each one in its smallest format. Offer the `optimize-images` skill for the files the project holds, and say it keeps each file's format unless the user asks for another, so it saves less than that figure when the smallest format was a different one. When `missing_alt` was found, offer the `image-seo` skill, which starts from this scan's id (its "A live page", item 3) without scanning again. Call `get_usage` again and say how many scans and images today has left.

Leave committing to the user. The scan and its images stay in their Iminify account, where the website lists them; delete images (`delete_image`, which can't be undone) only when they ask.
