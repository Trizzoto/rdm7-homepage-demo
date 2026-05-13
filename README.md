# rdm7-homepage-demo

Single-page WASM demo of the RDM-7 Dash, designed to be embedded as an
`<iframe>` on the Realtime Data Monitoring Shopify homepage.

Boots straight to the **dashboard scene** (synthetic drive cycle on the
default layout), renders inside a styled bezel at 800×480 native pixels,
scales down responsively for mobile, and forwards browser pointer events
into the firmware so visitors can tap on widgets just like they would on
the real product.

## Repo contents

- `index.html` — the demo page; bezel + canvas + WASM bootstrap
- `rdm7-sandbox.js` — Emscripten module loader (from `@rdm7/sandbox`)
- `rdm7-sandbox.wasm` — compiled LVGL + firmware (~1.9 MB)
- `RDM.rdmimg` — splash-screen logo (loaded into the WASM virtual FS)
- `_headers` — Cloudflare Pages config (WASM MIME, cache, no X-Frame-Options)
- `vercel.json` — same config in Vercel's format

## Local preview

Any static-file server will do. From the repo root:

```bash
# python
python -m http.server 5173
# node
npx http-server -p 5173
```

Open <http://localhost:5173>. The WASM should boot in 1–3 seconds and
the dashboard appears with synthetic RPM/speed/temperature ticking.

## Deploy

Pick one — both produce a public URL you'll embed in Shopify.

### Option A: Cloudflare Pages (recommended)

1. `git init && git add . && git commit -m "Initial homepage demo"`
2. Create an empty repo at <https://github.com/Trizzoto/rdm7-homepage-demo>
3. `git remote add origin git@github.com:Trizzoto/rdm7-homepage-demo.git && git push -u origin master`
4. In Cloudflare dashboard → **Workers & Pages** → **Create application**
   → **Pages** → **Connect to Git** → select the repo
5. Build settings: leave **build command** empty (static site), set
   **build output directory** to `/` (or leave blank)
6. Deploy. You get a URL like `rdm7-homepage-demo.pages.dev`.
7. Optional: add a custom domain (`demo.realtimedatamonitoring.com.au`)
   under **Custom domains** → **Set up a custom domain** → CNAME the
   subdomain to the Pages URL.

The `_headers` file is picked up automatically — `.wasm` files get the
correct `application/wasm` MIME type and cache headers.

### Option B: Vercel

1. Same git push as above
2. <https://vercel.com/new> → import the GitHub repo
3. Framework preset: **Other** (static)
4. Deploy. You get `rdm7-homepage-demo.vercel.app`.
5. Add custom domain in **Settings** → **Domains**.

`vercel.json` handles MIME types + cache headers.

### Option C: GitHub Pages (simplest, no extra account)

1. Push the repo to GitHub as above
2. Repo **Settings** → **Pages** → source: `master` branch, `/` root
3. Wait 30 seconds. URL: `https://trizzoto.github.io/rdm7-homepage-demo/`

GitHub Pages serves `.wasm` with the right MIME type by default.
Downside: no custom headers, no CDN tuning. Fine for a basic embed.

## Embedding on Shopify

Once you've got a deploy URL, open the Shopify storefront editor:

1. Pick the section on the homepage where you want the demo (or add a
   new **Custom Liquid** section)
2. Paste this snippet:

```html
<div style="max-width:880px; margin:0 auto; aspect-ratio:860/540;">
  <iframe
    src="https://demo.realtimedatamonitoring.com.au"
    loading="lazy"
    width="100%" height="100%"
    style="display:block; border:0; background:transparent;"
    sandbox="allow-scripts allow-same-origin"
    allow="autoplay"
    title="RDM-7 Dash live demo"></iframe>
</div>
```

Notes on the snippet:

- `loading="lazy"` defers the 2 MB WASM download until the user scrolls
  near the iframe — homepage paint stays fast.
- `aspect-ratio` keeps the iframe the right shape on every screen.
- `sandbox` is set conservatively — scripts can run, same-origin
  cookies/storage allowed for the dash's own state, nothing else.

### Putting it behind a "tap to start" button on mobile

If you'd rather not stream 2 MB of WASM to every mobile visitor who
scrolls past, append `?gate=1` to the iframe URL:

```html
<iframe src="https://demo.realtimedatamonitoring.com.au?gate=1" ...>
```

A "Tap to start" overlay shows; nothing loads until the user clicks.
Recommended for the mobile breakpoint of your Shopify theme.

## Updating the WASM

When you ship a new firmware build that you also want reflected in the
demo:

```bash
# from the rdm7-sandbox repo, after running scripts/build.ps1:
cp public/rdm7-sandbox.js   ../rdm7-homepage-demo/
cp public/rdm7-sandbox.wasm ../rdm7-homepage-demo/

# then in rdm7-homepage-demo:
git add rdm7-sandbox.js rdm7-sandbox.wasm
git commit -m "Sync WASM to latest firmware build"
git push
```

Cloudflare/Vercel auto-deploy on push. Browsers pick up the new bytes
within seconds (the HTML revalidates on every visit; the WASM is
content-hashed via ETag).

## How it works

- `rdm7-sandbox.wasm` is the firmware's LVGL UI compiled to WebAssembly
  via Emscripten with the SDL2 backend. The whole signal/widget/layout
  pipeline runs in-browser; the WASM exports `_sandbox_step`,
  `_sandbox_set_pointer`, and `_sandbox_set_scene` for the host page.
- `index.html` calls `_sandbox_set_scene(6)` (SCENE_DASHBOARD) right
  after the WASM module finishes booting, so the demo opens on the
  product's main dashboard rather than the first-run wizard.
- Synthetic signal injection (`_dash_tick` inside the WASM) drives RPM /
  speed / temperatures up and down on a continuous cycle so the demo
  always looks alive.
- Pointer events (mouse + touch + pen) are translated to native canvas
  pixel coordinates and forwarded via `_sandbox_set_pointer`.
- `requestAnimationFrame` is paused when the iframe goes hidden
  (`document.visibilitychange`) so background tabs don't burn CPU.

## Licence

Same as the firmware — see the source repo for details.
