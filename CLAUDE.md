# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A one-page static web app: it turns a screenshot into a square image for Podhome chapter and episode art. `README.md` is the user-facing doc.

- **Images never leave the browser.** The README promises it. Do not add an upload, analytics or a third-party script. The only outside request is the Google Fonts stylesheet and its fonts.
- **The target is Podhome's chapter art rules:** JPEG or PNG, square, 1400–3000 px, sRGB, and 500 KB or less loads faster. The size buttons (1400 / 2048 / 3000), the `colorSpace: 'srgb'` canvas context and the "Keep JPEG at 500 KB or less" option exist for these rules.
- **The target is a desktop browser.** Chad uses the app only on the desktop (2026-11-07). The iOS home-screen tags and the iPad hint in the fallback text exist, but iPad and iPhone are not targets, so do not ask for a device test. Headless Chrome covers Chrome and the other Chromium browsers. Desktop Safari is stricter about a download or a clipboard write that comes some time after the click, so a change to Save or Copy timing needs one Safari test if Safari is the browser in use.
- **Defaults live in the `state` literal** at the top of the script. The README's "Defaults" section repeats them, so change both together.

## Commands

There is no build, no package.json, no dependency, no test suite and no linter. Serve the folder and open it:

```bash
python3 -m http.server 8000 -d .    # then http://localhost:8000/
```

Use `localhost` or https: the service worker does not register from `file://`.

## Deploy

Vercel project `square-my-art` (team `chadfs-projects`), linked to this GitHub repo. A push to `main` deploys production at https://square-my-art.vercel.app. The project uses the "Other" framework preset, no build command, and the repo root as output.

A branch push makes a preview deployment, but Vercel deployment protection covers every domain except `square-my-art.vercel.app`. A preview or deployment URL answers 302 to a Vercel login, so curl and headless Chrome cannot open it. Test locally.

## Architecture

All app code is in `index.html`: the CSS, the markup, then one IIFE.

- **One state object, one update path.** Every control handler sets a field of `state` and calls `changed()`. That runs `syncUI()` (which buttons are pressed, slider values, which fields show for the current mode), `savePrefs()`, and `render()` on the next animation frame.
- **`render()`** redraws the `#out` canvas at `state.size` from `state.img`. That is an `HTMLImageElement`, or at boot the canvas from `makeSample()` (`state.sample` marks it): a made-up news-article page in a browser window, drawn in code, chosen by Chad on 2026-11-08 because its white page stands out from the blurred bands. The `crop` mode covers the square and uses the `pos` slider. The `blur` and `color` modes paint a background, then draw the image at the size the `fit` slider (−100 to 100, Fit at 0) sets, with an optional shadow. Left of 0 is a margin of `-fit / 5` %, up to 20 %. Right of 0 is a zoom that ends at 100 ("Fill"), where the image covers the square. A zoom cuts the long sides, and `pos` (the Crop position slider) picks the part that stays. It shows in every mode, also when nothing is cut (Chad's choice, 2026-11-08), so the panel does not jump. `pos` moves in steps of 5; its label is "Center" only at 50, otherwise the direction and distance ("Left 20 %", or "Top"/"Bottom" for a tall image). Both sliders have a `.tick` mark under their middle. Settings saved before the zoom have `pad` (a 0–20 % margin), which loads as `fit = -pad × 5`.
- **The output is always opaque.** In `blur` and `crop`, `render()` first paints a base in `edgeOf(src)`: the mean color along the edge of the opaque content. A transparent source, such as a macOS window screenshot (⌘⇧4, then Space), otherwise gave a transparent PNG and a black JPEG. The shadow is drawn from the image itself, so it follows a transparent margin. "Use image edge color" uses the same `edgeOf()`.
- **`blurredCover()` blurs by downscale, then stepwise 2× upscale.** This is on purpose: Safari has no `ctx.filter`. Do not replace it with `ctx.filter = 'blur(...)'`.
- **`encode()` runs after each render.** It keeps the promise of the newest encode in `encoding` and the finished blob in `encoded`. `encToken` discards the result of an encode that a newer one replaced. For JPEG with the 500 KB option, it tries quality 0.92 down to 0.45 and keeps the first blob of 500 KB or less, or the 0.45 blob if none fits (the chip then says "Over 500 KB"). The meta line's "File … KB" item shows `…` until the encode finishes.
- **Save** gets its file from `currentFile()`. That calls `flush()`, which draws a change that still waits for its animation frame, then waits for the newest `encoding`. So the file always matches the screen and the 500 KB option. Do not give Save its own encode again: that is how it once saved a 1.7 MB JPEG while the app said "500 KB or less". The file name is `<name>-square-<size>.<jpg|png>`.
- **Copy** always writes a PNG through `ClipboardItem` with a promise, which is the form Safari needs. Only a browser with no image clipboard disables the button. A refused copy (for example, the window lost focus) leaves it enabled.
- **Input:** the file picker, a drop anywhere on the page and a paste all go through `loadFile()`.
- **Settings persist** in localStorage key `square-my-art-prefs`: every `state` field except `img`, `name` and `sample`. A new persisted setting must also go into the key list of the restore block at the top of the script.

## Service worker and paths

- `sw.js` is network first, then cache. The cache name is `VERSION` and the precache list is `FILES`. Change `VERSION` when you change a cached file, and add a file to `FILES` if the app needs it offline.
- Every URL is root-absolute: `/sw.js`, `/manifest.webmanifest`, the icons, and the manifest's `scope` and `start_url`. The app works only at the root of an origin. A subpath host, such as a GitHub Pages project site, breaks the service worker, the manifest and the icons.

## Testing in a browser

There are no automated tests, so drive the app in headless Chrome over CDP. These methods worked in the 2026-11-06 review:

- **Load an image without the OS file picker:** build a `File`, add it to a `DataTransfer`, set `#file.files`, and dispatch `change`. A `DragEvent('drop', { dataTransfer })` on the page and a `ClipboardEvent('paste', { clipboardData })` on `document` also work.
- **Wait for an encode:** poll the `#fsize` item until it shows `NNN KB`. A change renders on the next animation frame, so `#fsize` shows the old size for a moment after a change.
- **A transparent source:** draw a light rectangle with `shadowBlur` inside a transparent margin on a canvas, and export it as PNG. It stands in for a macOS window screenshot.
- **Save** worked with a real mouse event (`Input.dispatchMouseEvent`) and `Browser.setDownloadBehavior` to catch the file. Chrome can also write a temporary `downloads.html` into that folder, so match the saved file by its `-square-<size>.<ext>` name. **Copy** worked with `Browser.grantPermissions` for the clipboard and `Emulation.setFocusEmulationEnabled`.
- **Phone layout:** use `Emulation.setDeviceMetricsOverride` with `mobile: true`, never `--window-size`.
- **CDP harness:** `~/Vibe/boostmebitch/scripts/cdp.mjs` is dependency-free and can be imported from a script outside this repo.
