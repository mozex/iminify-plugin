---
name: image-seo
description: Give a project's images alt text and descriptive file names with Iminify's AI, and put them into the markup that shows them, with the user's approval. Use when the user asks for alt text, image descriptions, image accessibility, or SEO-friendly image file names, for a project, a folder, some files or a live page. Not for making images smaller (compressing, converting and resizing are the optimize-images skill) or for how a page loads its images (LCP, layout shift, lazy loading are the fix-core-web-vitals skill).
---

# Image SEO

Search engines and screen readers know an image by its alt text and its file name. This skill finds the images in a project that lack one worth reading, has Iminify's AI write both, and puts them in place once the user has read them. Iminify does the writing on the user's own account, through the `iminify` MCP server this plugin connects.

The text is written by AI, so the user approves every alt text before it goes in, and nothing a person wrote is ever replaced.

## 0. Check the connection and the allowance

The Iminify tools are `get_usage`, `create_upload_link`, `compress_images`, `get_image` and `write_alt_text`, plus `scan_page` and `get_scan` for a live page. If they're missing, or a call says the server needs authentication, stop and tell the user how to sign in:

- Claude Code: run `/mcp`, pick `plugin:iminify:iminify`, and press Approve on the Iminify page that opens.
- Cursor: open Cursor Settings, then MCP, and press Connect next to `iminify`.

Signing in needs an Iminify account with a verified email address. The free plan works.

Then call `get_usage` and note:

- `usage.alt_texts.remaining`: AI alt texts left today (`null` means the plan has no daily cap);
- `usage.images.remaining`: images left today, since each local file is sent as an image (`null` means no daily cap);
- `usage.scans.remaining`: page scans left today, when the user named a live page;
- `limits.max_file_size`: the largest file the plan takes, in bytes.

When the user named a page's address rather than files, go to "A live page" below.

## 1. Find where the project shows its images

Work on exactly what the user named: a file, a folder, a page template. When they said "the project" or "the site", look through the project, skipping dependency, build and cache folders (`node_modules`, `vendor`, `.git`, `dist`, `build`, `public/build`, `.next`, `.nuxt`, `.output`, `coverage`, `storage`, `tmp`).

Look for every place an image is shown: `<img>` tags, Markdown images (`![...](...)`), and components that take an `alt` prop (a framework's own image component, or the project's). CSS background images have no alt text, so leave them out. For each place, note the image file it shows and what its alt says:

- **No alt attribute at all**, or a Markdown image with nothing between the brackets: this needs one.
- **`alt=""`**: someone marked the image as decoration. Leave it, unless the user says otherwise.
- **Alt text a person wrote**: leave it. Only when the user asks to review existing alt text, show the AI's next to it and let them pick.
- **Alt set from data** (`alt={post.title}`, `:alt="$image->caption"`): leave the code alone and tell the user where the text comes from.

When the user also wants better file names, note the images whose names say nothing about them: camera and phone names (`IMG_4032.jpg`, `DSC_0001.JPG`, `PXL_20260412_...`), `screenshot-...`, `image1.png`, `untitled`, `download (3).png`, hashes.

Iminify reads `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`, `.tif` and `.tiff`. An SVG, ICO or AVIF can't be sent, and neither can a file larger than `max_file_size`: name those in the report.

**Choose the language.** Alt text belongs in the page's own language. Read it from `<html lang>`, the site's locale config or the content, and pass `alt_text_language` (a language tag such as `de` or `ja`) when it isn't English. `compress_images` lists the languages it takes; for any other, tell the user rather than writing English into a page in another language.

Show the user what you found before anything is sent: the places that need alt text, the distinct files behind them, the files you'd rename if they asked for names, the language, and what it uses (one of today's images and one AI alt text per file). Wait for a yes. When there are more files than `remaining` allows today, say so and offer to start with the pages that matter most.

## 2. Ask Iminify for the alt text

Send each distinct file that needs alt text, or a new name, once, even when the project shows it in several places.

- **A file in the project:** upload it and send it to `compress_images` exactly as step 4 of the `optimize-images` skill in this plugin says (load that skill if it isn't loaded): upload links, one batch at a time, and what to do with each refusal code. Leave `alt_text` out, since it's on by default, and add `alt_text_language` when the language isn't English. Nothing is written back to the file: the compressed copy stays in the user's Iminify account.
- **An image the project loads from a web address** (a CDN or a CMS): pass the address in `compress_images`' `urls` instead of uploading it.
- **Images compressed earlier in this session with alt text on** (by the `optimize-images` skill, say): their ids already carry it. Go straight to step 3 with them.

## 3. Wait for the alt text

Poll `get_image` for each `id`, a few seconds apart. The image finishes first (`completed` or `already-optimized`), then its alt text a few seconds later, so keep going until `alt_text.status` is no longer `pending`. A `failed` or `cancelled` image keeps a `pending` alt text until it's retried, so stop waiting for it and name it in the report.

- `written`: `alt_text.text` is one sentence for the `alt` attribute, and `alt_text.suggested_name` a descriptive file name without an extension.
- `failed`: call `write_alt_text` once with the image's `id` (and `alt_text_language` if you sent one), then poll again.
- `skipped`: today's AI alt texts were used up when the image was queued. Say so: `write_alt_text` can ask again after `usage.alt_texts.resets_at`.

## 4. Put the alt text in place

Show the user every place and the text you'd put there, grouped by file. Read each place before you propose its text:

- An image that is the only content of a link or a button needs alt text that says where the link goes or what the button does ("Iminify's pricing page"), not what the picture shows. Rewrite the AI's text for those and say you did.
- The same picture can need different words in different places. Adjust the text to the surrounding content when the AI's sentence doesn't fit it, and mark what you changed.

Change only what the user approves. In HTML and templates, add an `alt` attribute and escape what the file's syntax needs (`"` becomes `&quot;` inside an HTML attribute). In Markdown, the text goes between the brackets, with `[` and `]` escaped. A component takes it in its `alt` prop.

## 5. File names, when asked

Rename only when the user asked for file names, and only the files they approve. A name and an alt text are separate approvals.

1. The new name is `alt_text.suggested_name` plus the file's own extension, in lower case. Never overwrite a file that's already there: when the name is taken, ask.
2. Search the project for every reference to the old name: code, templates, stylesheets, Markdown, config, sitemaps. When the name appears nowhere, the site may build the path at runtime or load it from a CMS: list that file for the user and leave it as it is.
3. Rename with `git mv` when git tracks the file, so its history follows. Otherwise move it. Then update every reference.

A published image has links to it from search results, image search and other sites. Renaming it breaks them unless the old address redirects to the new one, so for a live site say that before renaming, and suggest leaving images alone that already rank.

## A live page

When the user names a page's address ("which images on example.com have no alt text?"), a scan finds the images it shows with no alt attribute and writes alt text for those alone.

When the page was already scanned in this session (the `fix-core-web-vitals` skill does that), use that scan's id and start at item 3 below: no new scan is needed.

1. Call `scan_page` with the `url`, and `alt_text_language` when the page isn't in English. It counts one against today's page scans and one AI alt text for each image without alt; the images it finds don't count against today's images. It compresses them too, as every scan does, and they stay in the user's account. The page has to be reachable from the internet: Iminify refuses addresses on private networks, such as `localhost`, and a page behind a login usually comes back with no images.
2. Poll `get_scan` a few seconds apart until its `status` is no longer `queued` or `scanning` and both `images_in_progress` and `alt_texts_in_progress` are 0. A status of `failed`, `cancelled` or `no-image-found` ends it: report the status and stop.
3. Call `get_scan` once more with `audit_images: true`. When `audit.rendered` is `false`, only the page's HTML could be read: say the alt text couldn't be checked, and stop. Otherwise `audit.images` lists every image address on the page. The ones with `alt: "missing"` need alt text; `empty` is decoration, and `present` already has text. For each missing one:
   - `suggested_alt` has text: that's the alt text.
   - `suggested_alt` is `null` and the row has an `image_id` (its alt text failed or was skipped, it shares a copy with another address on the page, or the scan ran with alt text off): call `write_alt_text` with that `image_id`, and `alt_text_language` if needed, then poll `get_image` as section 3 above says. It costs one AI alt text and no image.
   - `outcome` is `left_out` (past the plan's images per scan): send the address to `compress_images` in `urls`, as section 2 above does. It counts as one of today's images.
   - `outcome` is `skipped` with no `image_id`: list it with its `skip_reason` (a format Iminify can't read, too large, unreachable). Sending it again would be refused the same way.
4. When the user's project is the site's source, find the file and the place behind each address, then carry on from section 4 above. Otherwise give the user a table of each image address and its suggested alt text.

## 6. Report

Finish with a table: each place, the file it shows, and the alt text it now has. List renames as old name, new name and the references updated. Under it, list what was left alone and why: alt text a person wrote, decoration, alt set from data, a file Iminify can't read or that's too large, a rename with no reference found, an alt text that failed or was skipped. Call `get_usage` again and say how many AI alt texts today has left.

Remind the user the text was written by AI and is worth one more read before they publish. Leave committing to them. The uploads stay in their Iminify account, where the website lists them; delete them (`delete_image`, which can't be undone) only when they ask.
