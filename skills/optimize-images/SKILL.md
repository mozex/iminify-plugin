---
name: optimize-images
description: Optimize the images in a project with Iminify and write the smaller files back in place. Compresses by default; converts to WebP, AVIF, JPG or PNG, or resizes, when asked. Use when the user asks to compress, optimize, shrink, convert or resize the images in a project, a folder or a set of files.
---

# Optimize images

Compress a project's images with Iminify and put the smaller files back where they were. Iminify does the encoding on the user's own account, through the `iminify` MCP server this plugin connects. This skill is the procedure around it: which files, which settings, and what gets replaced.

It changes files in the user's project, so it confirms before it starts and never overwrites anything that can't be restored.

## 0. Check the connection and the allowance

The Iminify tools are `get_usage`, `create_upload_link`, `compress_images` and `get_image`. If they're missing, or a call says the server needs authentication, stop and tell the user how to sign in:

- Claude Code: run `/mcp`, pick `plugin:iminify:iminify`, and press Approve on the Iminify page that opens.
- Cursor: open Cursor Settings, then MCP, and press Connect next to `iminify`.

Signing in needs an Iminify account with a verified email address. The free plan works.

Then call `get_usage` and note:

- `usage.images.remaining`: images left today (`null` means the plan has no daily cap);
- `limits.max_file_size`: the largest file the plan takes, in bytes;
- `limits.compressions_per_minute`: how many compressions a minute the plan allows.

## 1. Find the images

Work on exactly what the user named: a file, a folder, a pattern. When they said "the project" or "the site", look through the project. In a git repository, list tracked files with `git ls-files`, so ignored and generated files are left out. Otherwise, search the folder and skip dependency, build and cache folders: `node_modules`, `vendor`, `.git`, `dist`, `build`, `public/build`, `.next`, `.nuxt`, `.output`, `coverage`, `storage`, `tmp`.

Iminify reads `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`, `.tif` and `.tiff`. Leave everything else alone, SVG, ICO and AVIF included. Leave out files larger than `max_file_size` too, and name them in the report.

Show the user what you found before uploading anything: the number of files, their total size, the largest few, and the settings you'll use (step 3). Wait for a yes, unless the user already named the exact files and settings. When there are more files than `remaining` allows today, say so and offer to start with the largest ones.

## 2. Make sure the originals can come back

- In a git repository, a file can be restored after it's replaced only when git tracks it and it has no uncommitted changes: `git ls-files --error-unmatch <file>` succeeds and `git status --porcelain <file>` prints nothing. An ignored file prints nothing too, so the first check is what tells them apart. Name every file that fails either check (untracked, ignored, or changed) and ask before touching it.
- Outside git, say that the originals stay in the user's Iminify account (Iminify keeps every upload until the user deletes it, and `get_image` gives a fresh `original.download_url` at any time), and ask whether to go on.

## 3. Choose the settings

Send no settings unless the user asked for something. The defaults suit a website: level `smart` (the lowest quality that still looks like the original), the same format, and the metadata stripped (a photo's orientation is applied to its pixels first, so nothing comes back sideways).

| The user asks for | Send |
|---|---|
| no visible loss at all, "lossless" | `level: "lossless"` |
| "as small as possible", "aggressive" | `level: "ultra"` |
| WebP, AVIF, JPG or PNG | `format: "webp"` (or `"avif"`, `"jpg"`, `"png"`), and read step 6 first |
| "whatever format is smallest" | `format: "auto"`, and read step 6 first |
| no wider than N pixels | `width: N`, only for the images wider than N (below) |
| keep camera data, dates or location | `keep_metadata: true` |

A `width` or `height` is exact: Iminify enlarges an image that is smaller than it. For "no wider than 1600", read each image's width first and send `width` only in a call for the wider images. The rest go in a call without it. To read a width: `file` prints it for PNG and JPEG, as do ImageMagick's `identify` and, on macOS, `sips -g pixelWidth`. In Windows PowerShell, for JPG, PNG, GIF and TIFF (not WebP or HEIC): `Add-Type -AssemblyName System.Drawing; ([System.Drawing.Image]::FromFile((Resolve-Path 'path\to\hero.png'))).Width`. When a file's width can't be read, don't resize it: leave `width` out for it and say so in the report. Settings apply to every image in a call, so images that need different settings go in separate calls.

HEIC and TIFF always come back in another format, because Iminify doesn't write either. Treat them as conversions (step 6).

## 4. Upload, then compress

For each file, one at a time:

1. Call `create_upload_link` with the file's name. The answer has an `upload_id`, an `upload_url` and a ready `curl` command.
2. Send the file straight away, because the link expires within minutes and works once. Use the file's real path in the command:
   `curl --fail --upload-file 'path/to/hero.png' '<upload_url>'`
   In Windows PowerShell, type `curl.exe`: plain `curl` there is an alias for `Invoke-WebRequest`, which takes different arguments.
3. Keep the `upload_id` with the file's path.

An uploaded file keeps for about a day, so upload everything first, then call `compress_images` with `upload_ids` and the settings. One call takes a limited number of images (`compress_images`' own description gives the number, and a call over it is refused whole), so send the rest in further calls. The answer's `images` come back in the order of the `upload_ids` you sent, less the ones turned down, and each has an `id`: keep a map from that `id` to the local path. Anything turned down is in `refused`, with its `source` (the upload id), `code` and `message`. When nothing at all could be queued, the call comes back as an error that carries the same `refused` list. Handle each refusal by its `code`:

- `rate_limited`: the minute's allowance is spent. Wait `retry_after` seconds and send those upload ids again. An upload refused this way can still be used.
- `daily_limit_reached`: stop. Report what's done, what's left, and the message as it came.
- anything else: leave that file alone and give the reason in the report.

## 5. Wait, then write the results back

Poll `get_image` for each `id`, a few seconds apart, until its `status` is `completed`, `already-optimized`, `failed` or `cancelled`. Deal with each image as soon as it finishes, because download addresses expire after an hour. Ask `get_image` again for a fresh one if an address has expired.

- `completed`, the same format as the file, and `optimized.size` smaller than the file on disk: download `optimized.download_url` to a temporary file in the same folder (`curl --fail --location -o 'path/to/.hero.png.iminify' '<download_url>'`). Check that its size in bytes equals `optimized.size`, then move it over the original. If it doesn't, delete the temporary file, ask `get_image` for a fresh address and download again.
- `completed` in another format: it's a conversion. Follow step 6.
- `already-optimized`: Iminify couldn't make the image smaller with these settings. Leave the file as it is and say so plainly, with no saving reported.
- `completed` but not smaller: leave the file as it is.
- `failed` or `cancelled`: leave the file as it is, and pass on the `hint` if there is one.

## 6. Conversions change file names

A PNG converted to WebP becomes `hero.webp`, and everything that loads `hero.png` still points at the old file. Convert only when the user asked for it. Before writing anything:

1. Search the project for each file's name: code, templates, stylesheets, Markdown, config.
2. Tell the user what will change: the new files, the references you'll update, and that the old files stay unless they want them removed.
3. Check that nothing already sits at the new name. A site often ships `hero.webp` beside `hero.png` already: never overwrite it, and don't convert that original. Name those files in the report instead.
4. Save each result next to its original under the new extension, checking its size against `optimized.size` as in step 5, then update the references.

When a file's name appears nowhere in the project, the site may build its path at runtime or load it from a CMS. List those files for the user rather than guessing, and never delete their originals.

## 7. Report

Finish with a table: each file, its size before and after, and what it saved in bytes and percent, then a line for the total. Under it, list the files left alone and why: already optimized, too large, refused, failed, or skipped at the user's request. Call `get_usage` again and say how many images today's allowance has left.

Leave committing to the user. The uploads and their results also stay in the user's Iminify account, where the website lists them. Only delete them (`delete_image`, which can't be undone) when the user asks.
