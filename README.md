# Theodolite Panorama (PWA)

Guided multi-frame panorama capture for phones, with on-screen reticles,
gyroscope tracking, and equirectangular compositing.

## What's new in v2

- **Fixed overlay alignment.** The pitch sign was inverted in v1 (looking up
  was being treated as looking down), and the yaw reference was being
  set on the first sensor reading rather than on a deliberate user action,
  so the targets never lined up with where you were actually pointing.
- **New capture flow.** A large amber shutter button is shown initially.
  Aim the phone where you want the panorama to start, tap shutter — that
  direction becomes 0°, the first frame is captured, and target reticles
  appear for the rest. Auto-capture (align reticle, hold ~½ sec) handles
  the rest, exactly like before.
- **Visible-FOV projection.** Reticles are now projected using the
  *visible* FOV (accounting for the `object-fit: cover` crop on the video
  element), not the camera's full FOV, so they sit where the world
  features they represent actually appear on screen.
- **Captures are cropped to what you saw.** What you framed on screen is
  what gets stored, with the visible FOV recorded per-frame in metadata.
- **Level bubble** at top of screen to help you keep the phone vertical
  (roll/gamma is recorded in metadata but not yet compensated in the
  reticle projection — hold the phone vertical for best alignment).
- **Off-screen arrow** points to the next target when it's outside the
  current frame.

## Hosting

A PWA requires HTTPS to install. Drop these files on any static host:

- **GitHub Pages** — push to a `gh-pages` branch or repo with Pages enabled.
- **Cloudflare Pages** — drag and drop the folder at pages.cloudflare.com.
- **Netlify** — drag and drop at app.netlify.com/drop.
- **Surge** — `npm i -g surge && cd theodolite-pwa && surge`.

You **cannot** install a PWA from a `file://` URL or from plain HTTP. The
service worker needs HTTPS (localhost is fine for desktop dev, but the
phone needs a real HTTPS URL it can reach).

Once hosted, open the URL on your phone:

- **iOS Safari:** Share → "Add to Home Screen".
- **Android Chrome:** menu → "Install app" (or the install button on the
  setup screen will trigger the native prompt automatically).

After installation the app runs fullscreen, offline, and shows up like a
native app on the home screen.

## Files

```
theodolite-pwa/
├── index.html              ← the app
├── manifest.webmanifest    ← PWA manifest
├── sw.js                   ← service worker (offline cache)
└── icons/                  ← app icons (PNG + SVG, normal + maskable)
```

## Permissions the app will ask for

- **Camera** — for the live viewfinder and the captured frames.
- **Motion & Orientation** (iOS calls this "Motion Access") — for the
  gyroscope/compass that drives the reticle tracking. On iOS this
  permission must be granted on every fresh session via a user gesture;
  the app requests it when you tap "Initialise Capture".

## Output

After capture you get two downloads:

- `theodolite-*.png` — 4096×2048 equirectangular preview composite
  (per-pixel reprojection — seams visible, no blending).
- `theodolite-*.zip` — original frames as JPEGs + `metadata.json` listing
  yaw/pitch/roll/FOV per frame, plus stitching instructions.

For a clean stitch, feed the zip into **Hugin** (free) or **PTGui**
(paid) on desktop. The metadata lets them skip feature matching if you
want; otherwise let them auto-align as normal.

## Known limitations

- No roll (gamma) compensation in the reticle projection yet — the level
  bubble warns you when the phone isn't vertical.
- In-browser composite is per-pixel reprojection only; seams are visible
  and there's no exposure blending. Use desktop stitchers for finished work.
- Modes "Sphere" and "Little Planet" include near-zenith/nadir targets
  that are physically awkward to point at — expect them to be the
  trickiest captures of the set.
