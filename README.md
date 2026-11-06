# Square My Art

Make a square image from a screenshot. The empty space gets a blurred copy of the screenshot, a solid color, or the image is cropped to fill the square.

Made for Podhome chapter art. All work stays in the browser. Images are not sent to a server.

## Podhome chapter art rules

- JPEG or PNG (GIF is experimental)
- Square
- 1400 × 1400 minimum, 3000 × 3000 maximum
- sRGB color space
- Files of 500 KB or less load faster

## Defaults

- Fill: Blur (strength 6, darken 0 %)
- Size: 3000 × 3000
- Format: PNG (lossless). For JPEG, quality is 95. "Keep JPEG at 500 KB or less" is off.

The app remembers your last settings in the browser.

## Input

Drop a file on the page, paste an image (Cmd+V / Ctrl+V), or click the drop box to choose a file.

## Files

| File | Purpose |
|---|---|
| `index.html` | The app (HTML, CSS and JavaScript in one file) |
| `manifest.webmanifest` | Web app manifest, for "Add to Home Screen" and "Add to Dock" |
| `sw.js` | Service worker, so the app opens offline. Change `VERSION` when you change cached files. |
| `icon.svg`, `icon-*.png`, `apple-touch-icon.png` | App icons |

## Deploy

Static site. No build step. On Vercel, use the "Other" framework preset with no build command and the repo root as the output directory.
