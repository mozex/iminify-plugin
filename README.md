<p align="center">
  <img src="assets/logo.svg" alt="Iminify" width="96">
</p>

# Iminify for Claude Code, Claude Cowork and Cursor

[Iminify](https://www.iminify.com) compresses, converts and resizes images, and scans web pages for every image they load. This plugin connects Claude Code, Claude Cowork or Cursor to Iminify's MCP server and adds a skill that optimizes the images in a project, or any folder of them, and writes the smaller files back in place.

It reads JPG, PNG, WebP, GIF, HEIC and TIFF, and writes JPG, PNG, WebP and AVIF.

## What's inside

- **The `iminify` MCP server** at `https://www.iminify.com/mcp`. It signs in with OAuth, so there's no key to paste.
- **The `optimize-images` skill.** It finds the images in a project, uploads them, compresses them and puts the results back. It shows you the list and waits for a yes before it starts. It won't overwrite a file git can't restore, and it only renames files, updating their references, when you ask for a new format.

The server's tools:

| Tool | What it does |
|---|---|
| `compress_images` | Compress, convert or resize images from their addresses, from upload links, or sent inline |
| `create_upload_link` | A one-time address to send a local file to |
| `get_image` | One image's status, sizes, savings and download links |
| `list_images` | Your images, newest first, filtered by status, source or scan |
| `create_zip` | One download link for several finished images, or all of a scan's |
| `scan_page` | Scan a web page for every image it loads and compress them all |
| `get_scan` | A scan's progress, its images and what it saved |
| `rename_image` | Change the name a result downloads under |
| `share_image` | Give a result a public page to send to someone |
| `delete_image` | Delete an image and both its files, for good |
| `get_usage` | Your plan, its limits and what today has used |

## Install

You need an Iminify account with a verified email address. [The free plan](https://www.iminify.com/pricing) works.

### Claude on the web and desktop

No plugin needed: Iminify is in [Claude's connectors directory](https://claude.ai/directory/connectors/iminify). Open Customize, then Connectors, search for Iminify, press Connect and approve it on the Iminify page that opens. Claude and Cowork can then compress, convert and resize any image with a web address. The plugin below adds the skill that works through folders on your computer.

### Claude Code

```text
/plugin marketplace add mozex/iminify-plugin
/plugin install iminify@iminify
```

Then run `/mcp`, pick `plugin:iminify:iminify`, and press Approve on the Iminify page that opens.

### Claude Cowork

In the Claude desktop app, open Customize, then Plugins. Under Personal plugins, press +, choose Add marketplace, then Add from a repository, and enter `https://github.com/mozex/iminify-plugin`. Install Iminify from it and sign in the first time Cowork asks.

Then give Cowork a folder and say "optimize the images in this folder". It lists what it found, waits for your yes, and writes the smaller files back. Cowork also uses the connectors on your Claude account, so with Iminify connected from the directory, a task can compress images from their web addresses even without the plugin.

### Cursor

The plugin is on its way to the Cursor Marketplace. Until it's listed there, add the server with this button:

[![Add to Cursor](https://img.shields.io/badge/Add_to-Cursor-000000?style=for-the-badge)](https://cursor.com/en/install-mcp?name=iminify&config=eyJ1cmwiOiJodHRwczovL3d3dy5pbWluaWZ5LmNvbS9tY3AifQ==)

Cursor shows a Connect button next to `iminify` in its MCP settings. Press it to sign in.

### Anything else

Any app that speaks MCP over HTTP can use `https://www.iminify.com/mcp`. The [setup guide](https://www.iminify.com/docs/mcp) covers Claude, ChatGPT, VS Code and connecting with an API key.

## Things to ask

- "Optimize the images in `public/images`."
- "Convert the PNGs in `resources/img` to WebP and update the references."
- "Make every photo in `content/` at most 1600 pixels wide."
- "Scan https://example.com and tell me which images are heaviest."
- "Compress https://example.com/hero.jpg to AVIF and give me the link."

## Limits

Every image the agent compresses counts against your plan's daily images, the same count the website and the API use. `get_usage` shows what's left, and the [pricing page](https://www.iminify.com/pricing) has the numbers for each plan. When a limit is reached, the agent passes on Iminify's message and how long to wait.

## Your data

Iminify keeps each upload and its optimized copy in your account until you delete them. Results come back as download links that work for an hour. [Settings > Connected apps](https://www.iminify.com/settings/connected-apps) shows every app you've approved and disconnects any of them. See the [privacy policy](https://www.iminify.com/privacy-policy) and the [terms of service](https://www.iminify.com/terms-of-service).

Questions or problems: support@iminify.com, or open an issue here.

## License

The plugin files are [MIT licensed](LICENSE). The Iminify service they connect to is covered by its [terms of service](https://www.iminify.com/terms-of-service).
