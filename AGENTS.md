# AGENTS.md

## What this app is

A single static page (`index.html`) — an HTML file viewer that runs entirely in the browser.
No build step, no dependencies, no backend, no environment variables, no external services.

## Run it

```sh
docker compose -f docker-compose.base44.yml up -d
curl -sS http://localhost:3000/
```

`nginx:alpine` serves `index.html` on host port 3000 and the page is bind-mounted read-only,
so edits are served on the next request (no rebuild, no rebuild-on-reload command). Only
`index.html` is mounted, not the repo root, so `.git` is never exposed.

Verify a change in the browser. The page has two tabs: `#tabUpload` (drop zone, `#dropZone`)
and `#tabPaste` (`#codeArea` textarea + `#pasteOpenBtn`). Tick "Preview inside this page
instead of a new window" so results stay in-page, then either dispatch a `drop` event with
a `DataTransfer` holding a `File` on `#dropZone`, or fill `#codeArea` and click
`#pasteOpenBtn`; `#previewWrap` gains `.show`, `#status` turns green and `#previewFrame`
gets a `srcdoc` containing the HTML.

The popup path itself (checkbox off) cannot be verified in the automated preview browser —
it blocks the window, so the app shows its "Popup blocked … Showing preview below."
fallback. Check that fallback and leave the real new-window behaviour to a manual click.

## Deploying (Cloudflare Workers static assets)

`wrangler.jsonc` publishes this repo as static assets (`assets.directory: "./"`, no Worker
script), and `.assetsignore` keeps everything except `index.html` out of the upload — verify
with `WRANGLER_LOG=debug npx wrangler deploy --dry-run` and look for `Ignoring asset:` lines.

On Cloudflare Workers Builds: build command can stay empty, deploy command `npx wrangler deploy`.
The Worker name in `wrangler.jsonc` must match the Workers project name. Deploys follow the
project's configured branch — `main` — so changes on a `base44/*` branch only reach the live
site once merged.

## Non-obvious quirk: no literal HTML tag names inside the inline script

The whole application script is inline in `index.html`. Some HTML post-processors
(the Base44 preview proxy is one) inject their own scripts right after each head tag they
find in the document source. A literal head-tag string inside the inline script therefore
gets the injector's markup spliced into the middle of the script, which closes the
`<script>` element early: everything after it renders as page text and every handler is
detached (`Uncaught SyntaxError`, checkbox and drop do nothing).

That is why loading-state markup is built from the `TAGS` object (`"<" + "head>`, etc.)
instead of a plain string. Keep it that way: never write a literal `<!DOCTYPE html>…<head>`
document string inside the inline script, and do not put tag names between angle brackets
in comments there either.
