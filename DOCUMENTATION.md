# Theodolite Panorama — Technical Documentation

A guided multi-frame panorama capture app for phones. It runs entirely
in the browser as an installable PWA: it uses the camera and motion
sensors, draws on-screen reticles to guide the user through a sequence
of aimed shots, and reprojects the captured frames into an
equirectangular (or fisheye) composite.

This document covers what the app is, how it works, the design
decisions behind it, and — most importantly — how to understand and
debug it. It is written for a developer picking the project up cold.

---

## 1. What the app is, and what it is not

**It is** a capture instrument. It tells the user exactly where to
point the phone, tracks orientation with the gyroscope/compass,
auto-captures when the aim is correct, and records each frame's pose
(azimuth, elevation, roll, field of view) into a `metadata.json`.

**It is not** a stitcher. The in-browser composite is a *pose-only
reprojection*: every frame is placed using its sensor pose and an
assumed perfect rectilinear lens. There is no feature matching, no lens
distortion solve, and no exposure blending. Seams and residual
misalignment are expected and normal. The real deliverable is the
**frame ZIP**: original JPEGs plus `metadata.json`, intended to be fed
into Hugin or PTGui on a desktop, which *do* solve distortion and align
on image content.

Keeping this boundary clear is the single most important thing for
anyone debugging the app. See §9 (Known limitations) — most "bugs"
people report against the composite are actually inherent limits of
pose-only reprojection.

---

## 2. File layout

```
theodolite-pwa/
├── index.html            The entire app — HTML, CSS, JS inline.
├── diagnostic.html        Standalone sensor/camera test bench.
├── manifest.webmanifest   PWA manifest.
├── sw.js                  Service worker (offline app-shell cache).
├── icons/                 App icons (PNG, normal + maskable, + SVG).
├── README.md              Short user-facing build/install notes.
└── DOCUMENTATION.md        This file.
```

The app is deliberately a single self-contained `index.html`. There is
no build step, no framework, no bundler. CSS and JS are inline. The
only external runtime dependencies are the JSZip CDN script (for the
download ZIP) and Google Fonts. Both are cached by the service worker
and both fail soft — if JSZip is unreachable the ZIP button reports an
error but the rest of the app still works.

---

## 3. The three screens and the capture flow

The UI is three absolutely-positioned `.screen` divs, toggled by
`showScreen(name)`. There is no router.

1. **Setup** (`#setup`) — pick the projection mode, detect and select a
   camera, set the field of view, the lock tolerance, and the motion
   tracking source. "Initialise Capture" calls `startCapture()`.

2. **Capture** (`#capture`) — live camera feed with an overlaid HUD
   canvas. The flow:
   - The user aims the phone and taps the central **shutter** button.
     `performInitialCapture()` runs: it sets the yaw reference (this
     direction becomes azimuth 0°), captures the first frame, then
     reveals the target reticles.
   - For every remaining target, the user aligns the reticle and holds
     steady. `checkAutoCapture()` fires `attemptCapture()` once the aim
     is within tolerance for ~380 ms.
   - When all targets are captured (or the user taps **Finish**),
     `finishCapture()` runs.

3. **Review** (`#review`) — shows the composite, capture stats, and
   download buttons (composite PNG, and the frame ZIP).

`finishCapture()` → `composeEquirectangular()` builds the 4096×2048
equirectangular master. For the fisheye mode it additionally runs
`renderFisheye()` to derive a circular fisheye from that master.

---

## 4. Coordinate systems and the orientation maths

This is the core of the app and the most common source of confusion.
Read this section before changing anything in `eulerToLook`,
`cameraBasis`, `projectTarget`, `placeCapture`, or `renderFisheye`.

### 4.1 The app world frame

All projection maths uses one consistent right-handed world frame:

- **X** = right / east
- **Y** = up
- **Z** = forward / north (the user's chosen start direction)

A direction is described by **azimuth** and **elevation**:

- `azimuth` — 0° = forward (+Z), increasing **clockwise** viewed from
  above (turning right is +). Range normalised to [0, 360).
- `elevation` — 0° = horizon, +90° = straight up (zenith), −90° =
  straight down (nadir).

The world vector for an (az, el) pair is:

```
x = sin(az)·cos(el)
y = sin(el)
z = cos(az)·cos(el)
```

The equirectangular image uses the same convention: column → longitude
(== azimuth), row → latitude (== elevation).

### 4.2 Why we do NOT use raw Euler angles as yaw/pitch

`DeviceOrientationEvent` reports three Euler angles — `alpha`, `beta`,
`gamma`. An early version of the app used `alpha` directly as yaw and
`beta − 90` as pitch. **This is broken**, and the failure is specific
and instructive:

A panorama is shot with the phone held vertically, which means
`beta ≈ 90°`. That is a **gimbal-lock singularity** of the Euler
representation. Near it, a tiny real movement of the phone makes
`alpha` jump or flip by up to 180°, while `gamma` swings to compensate.
Fed into the projection, the overlay "springs in contradictory
directions". This was reported as a bug and is the reason the maths was
rewritten.

### 4.3 The look-vector / camera-basis approach (`eulerToLook`)

The fix: never use a single Euler angle as an axis. Instead build the
full device→world rotation matrix `R = Rz(α)·Rx(β)·Ry(γ)` and read the
camera's orientation off it as three orthonormal vectors:

- **forward** = the camera's optical axis (device −Z) in world coords
- **up** = the top of the screen (device +Y) in world coords
- **right** = device +X in world coords

`eulerToLook(alpha, beta, gamma)` computes the forward and up vectors
directly (the explicit matrix-column expressions are in the code) and
returns:

- `az` = `atan2(forward.x, forward.z)` — continuous through `beta=90`
- `el` = `asin(forward.y)` — continuous everywhere
- `roll` — see §4.4
- `nearPole` — true when `|el| > 85°`, where azimuth is ill-defined

Because the look vector is a *composite* of all three angles, it stays
continuous straight through the `beta=90` singularity. The singularity
still exists mathematically, but it now only affects the *decomposition
into an azimuth number* at the exact poles (looking straight up/down) —
which is why azimuth is frozen when `nearPole` is true.

### 4.4 Roll, and why `gamma` is not roll

Camera **roll** is rotation about the optical axis. It is tempting to
use `gamma` for this. **`gamma` is not roll.** At the near-vertical
pose used for panoramas, `gamma` actually behaves as a *yaw*. Using it
as roll silently corrupts the composite.

`eulerToLook` derives the true roll geometrically. Given the camera
forward and up vectors:

- `levelUp` = the component of world-up perpendicular to `forward`,
  normalised — i.e. where "up" would be if the camera were not rolled.
- `levelRight` = `forward × levelUp`.
- `roll` = `atan2(camUp·levelRight, camUp·levelUp)` — the signed angle
  by which the camera's actual up has rotated away from level, measured
  about the optical axis.

This is well-defined everywhere except looking exactly up/down, where
`levelUp` degenerates and roll is frozen at 0.

### 4.5 `cameraBasis(az, el, roll)` — the single projection primitive

Both the live reticle (`projectTarget`) and the composite assembly
(`placeCapture`) need to turn a world direction into camera-image
coordinates. They both use one shared helper, `cameraBasis`, which
rebuilds the orthonormal `{right, up, fwd}` basis from az/el/roll:

1. `fwd` from (az, el) as in §4.1.
2. `right0` = the horizontal zero-roll right vector `(cos az, 0, −sin az)`.
3. `up0` = `fwd × right0`.
4. Rotate `right0`/`up0` about `fwd` by `roll` to get the final
   `right`/`up`.

Projecting a world vector `w` is then just three dot products:
`cx = w·right`, `cy = w·up`, `cz = w·fwd`; then `px = cx/cz`,
`py = cy/cz`, with the pinhole scale `tan(FOV/2)`.

**Important invariant:** at `roll = 0`, `cameraBasis` reproduces the
older azimuth/elevation rotation *exactly* (verified numerically — the
two agree to ~1e-15). So the roll support is a strict superset of the
previous behaviour. If you change `cameraBasis`, re-check that
invariant.

### 4.6 The visible FOV problem (`updateVisibleFOV`)

The `<video>` element uses `object-fit: cover`, which **crops** the
camera image to fill the screen. The FOV visible on screen is therefore
*narrower* than the camera's native FOV. The reticles must be projected
with the visible FOV, not the native one, or they will not sit on the
real-world features they represent.

`updateVisibleFOV()` computes `visFovH` / `visFovV` from the
cover-crop maths (native video dimensions vs screen dimensions). The
reticles use these. `attemptCapture()` crops each saved frame to the
same visible area and stores the visible FOV with it, so the composite
reprojects exactly what the user framed.

---

## 5. Orientation event handling

Android browsers fire **both** `deviceorientation` (relative,
gyroscope-fused, smooth, slowly drifting) and
`deviceorientationabsolute` (magnetometer-referenced, true-north, but
jumps under magnetic interference). iOS typically fires only one.

If both streams are fed into one handler they fight each other — their
`alpha` values disagree, and the yaw smoother is yanked between them
every frame. So the app **locks onto a single source**:

- `onOrientationAbsolute` and `onOrientationRelative` are thin wrappers.
- The first to deliver a usable event sets `state.orientSource` and the
  other is ignored for the rest of the session.
- `state.orientPref` (`auto` / `relative` / `absolute`, chosen by the
  user in the Setup screen's Motion Tracking section) biases which one
  wins. `auto` prefers **relative** (gyro) because it is smoother and
  the user sets the zero direction with the shutter anyway, so absolute
  true-north is not needed.

### 5.1 Smoothing

Raw sensor values are noisy at 60 Hz+. `onOrientation` applies a
time-constant exponential moving average (`SMOOTH_TAU_MS = 80`):
`a = 1 − exp(−dt/τ)`. This is frame-rate independent because it uses
the real `dt` between sensor events. Yaw uses `smoothAngle`, which is
wrap-aware (averaging 359° and 1° gives 0°, not 180°). The first event
of a session initialises without smoothing so the overlay does not
animate in from zero.

---

## 6. Projection modes

Modes live in the `MODES` object. Each has a `compute()` returning an
array of `{az, el}` target directions, a label, a description, and an
SVG glyph.

| Key       | Name           | Pattern                                    |
|-----------|----------------|--------------------------------------------|
| `cyl`     | Cylindrical    | One horizontal ring, 360°.                 |
| `wide`    | Wide           | A 180° forward horizontal arc.             |
| `vert`    | Vertical       | A single floor-to-ceiling column.          |
| `sphere`  | Sphere         | Full 360×180 — rings + zenith + nadir.     |
| `planet`  | Little planet  | Dense lower-hemisphere capture.            |
| `fisheye` | Fisheye 180°   | Forward hemisphere; circular output.       |

### 6.1 The fisheye mode

The fisheye mode (`MODES.fisheye`, flagged `fisheye: true`) is the only
mode with a different **output projection**, not just a different
capture pattern.

- **Capture pattern** — targets covering the forward hemisphere
  (everything within 90° of the lens axis), laid out as concentric
  rings at 30°/58°/82° off the forward axis plus the centre point. A
  ring at angle θ with roll φ gives direction
  `(sinθ·cosφ, sinθ·sinφ, cosθ)`, converted to az/el.
- **Output** — the app still composites to the equirectangular master
  (the universal interchange format). `renderFisheye()` then reprojects
  that master into a circular **equidistant** 180° fisheye: a square
  image with the hemisphere inscribed as a circle, centre = lens axis,
  rim = 90° off-axis, radius linear in angle.

In fisheye mode the Review screen shows the circular result and the ZIP
contains both `fisheye_180.png` and `composite_equirect.jpg`.

---

## 7. The capture → composite pipeline in detail

1. **`attemptCapture(manual)`** — fires on auto-lock or the manual
   button. It crops the live video to the visible (cover-crop) area,
   draws it to an off-screen canvas, and pushes a capture record onto
   `state.captures`:
   ```
   { targetIdx, targetAz, targetEl,
     az, el, roll,            // measured pose at capture
     fov,                     // visible horizontal FOV
     fovStated,               // the user-set camera FOV
     w, h, canvas,            // the cropped frame
     manual, timestamp }
   ```

2. **`composeEquirectangular(progressCb)`** — creates the 4096×2048
   `#compositeCanvas`, draws a faint grid, then calls `placeCapture`
   for each frame.

3. **`placeCapture(ctx, W, H, cap)`** — the reprojection. For each
   equirectangular output pixel it computes the world direction, builds
   the camera basis with `cameraBasis(cap.az, cap.el, cap.roll)`,
   projects via three dot products, and samples the source frame. A
   bounding-box optimisation restricts the work to the strip the frame
   can possibly cover; that box is widened by the frame *diagonal* when
   roll is non-zero, because a rolled rectangle's footprint is larger.

4. **`renderFisheye(progressCb)`** (fisheye mode only) — reprojects the
   equirectangular master into the circular fisheye, in row chunks so
   the UI stays responsive.

5. **`downloadComposite()` / `downloadZip()`** — the ZIP contains the
   frames as JPEGs, `composite_equirect.jpg`, `metadata.json`, a
   `README.txt` with stitching instructions, and (fisheye mode)
   `fisheye_180.png`.

### 7.1 `metadata.json` schema

```jsonc
{
  "generator": "Theodolite Panorama Capture",
  "version": "2.1",
  "timestamp": "ISO-8601",
  "mode": "cyl|wide|vert|sphere|planet|fisheye",
  "mode_label": "...",
  "output_projection": "equirectangular|circular-fisheye-180-equidistant",
  "camera_fov_h_deg_stated": 78,        // user-set FOV
  "camera_lens": "main|ultrawide|tele|front|unknown|unspecified",
  "camera_label": "...",                // raw device label
  "camera_settings": { "width": 1920, "height": 1080, "facing": "environment" },
  "visible_fov_h_deg": 71.2,            // FOV after the cover-crop
  "visible_fov_v_deg": 110.5,
  "tolerance_deg": 4,
  "yaw_source": "compass-true-north|device-relative",
  "orientation_event": "absolute|relative|unknown",
  "frames": [
    {
      "file": "frame_000_az000_el+00.jpg",
      "index": 0,
      "target_az_deg": 0, "target_el_deg": 0,
      "actual_az_deg": 0.4, "actual_el_deg": -1.1, "actual_roll_deg": 2.3,
      "fov_h_deg": 71.2,                // FOV of the saved (cropped) frame
      "fov_h_deg_camera_full": 78,
      "width": 1080, "height": 1920,
      "manual_capture": false,
      "timestamp": 1700000000000
    }
  ]
}
```

`actual_az_deg` / `actual_el_deg` / `actual_roll_deg` and `fov_h_deg`
are the numbers a desktop stitcher needs as a starting pose for each
image.

---

## 8. Design decisions and their rationale

A chronological summary of why the app is shaped the way it is. Most of
these were made in response to a specific reported failure.

1. **Web app, not native.** The build environment cannot produce signed
   iOS/Android binaries. A PWA gives camera, motion sensors, install-to-
   home-screen and offline use with one codebase.

2. **Single-file app, multi-file PWA bundle.** The app logic is one
   `index.html` for simplicity. It is delivered as a bundle (manifest,
   service worker, icons) because a single file cannot be a proper
   installable, offline PWA.

3. **Shutter-first capture flow.** Originally the yaw reference was set
   on the first sensor event — i.e. wherever the phone happened to be
   when sensors woke up. Now the user deliberately aims and taps a
   shutter; that frame defines azimuth 0°. This is also what makes
   alignment debuggable: the origin is a known, user-chosen moment.

4. **Look-vector orientation maths.** See §4.2–4.3. Replaced raw-Euler
   yaw/pitch after the overlay was reported "springing in contradictory
   directions" — the classic gimbal-lock signature at `beta ≈ 90°`.

5. **Single-source orientation locking.** See §5. Added after the app
   worked on iPhone but not on a multi-sensor Android phone, because
   both orientation event streams were being fed into one handler and
   disagreeing.

6. **Explicit camera picker.** Multi-lens phones expose several rear
   cameras; `facingMode: environment` lets the browser pick, and Chrome
   and Firefox picked *different* lenses (ultrawide vs main) with
   different FOVs — which broke the projection. The Setup screen now
   detects and lists cameras and requests one by exact `deviceId`, with
   FOV presets tied to lens type.

7. **Gyro as the default tracking source.** `auto` prefers the relative
   (gyroscope) event: it is smoother and immune to magnetometer
   interference. True-north is unnecessary because the shutter sets the
   zero. Compass remains selectable.

8. **True geometric roll.** `placeCapture` originally ignored roll
   entirely, and the value it stored as "roll" was `gamma` (which is
   not roll — see §4.4). Roll is now derived geometrically and applied
   in both the reticle and the composite.

9. **Honest "preview, not stitch" framing.** The in-browser composite
   is pose-only reprojection and cannot be seamless. Rather than
   pretend otherwise, the Review screen says so and the app's real
   output is the frame ZIP for Hugin/PTGui.

---

## 9. Known limitations

These are inherent, not bugs. They are the reason the composite has
residual misalignment and the reason the desktop-stitch path exists.

- **Lens distortion.** Phone lenses (especially ultrawide) are not
  perfectly rectilinear; straight lines bow. The reprojection assumes a
  perfect pinhole lens, so frame edges will not perfectly meet. Cannot
  be solved from sensor data alone.
- **FOV estimate error.** The lens FOV is a user-set estimate. If it is
  a few percent off, every frame is reprojected at slightly the wrong
  angular size and the error accumulates around the panorama.
- **Parallax.** Rotating around the body rather than the lens's nodal
  point shifts near objects between frames.
- **Sensor noise / drift.** Any error in the measured pose translates a
  whole frame bodily; there is no feature matching to correct it. The
  relative (gyro) source also drifts slowly over minutes.
- **No blending.** There is no exposure matching or feathering. Note
  that blending only smooths *correctly-placed* overlaps — it cannot
  fix geometric misalignment, and would only be worth adding after a
  feature-based align.
- **Pole awkwardness.** Sphere/planet/fisheye modes include near-zenith
  and near-nadir targets that are physically awkward to aim at.

---

## 10. Debugging guide

A symptom-first reference. "Composite" means the in-app preview;
"reticle/overlay" means the live on-screen guide during capture.

### Overlay does not appear, or is clipped to half the screen
- Canvas sizing race / containing-block issue. The canvas is sized from
  `window.innerWidth/innerHeight` every frame in `ensureCanvasSized()`;
  if you reintroduce `clientWidth`-based sizing or body padding around
  the fixed-position layout this regresses.
- Check `ensureCanvasSized` runs each frame and that `hudCssW` is reset
  to 0 on resize / orientation change / `visualViewport` resize.

### Overlay springs / jumps in contradictory directions as you pan
- Gimbal lock. Confirm `eulerToLook` is in use and that nothing feeds a
  raw Euler angle straight into yaw/pitch. Run the **Sensor Stability**
  test in `diagnostic.html` — it explicitly detects the signature
  (raw `alpha` noisy while the look-vector azimuth stays calm).
- If the look-vector azimuth *itself* is noisy, it is a genuine sensor
  problem — try Gyro mode and recalibrate the magnetometer (figure-8).

### Overlay tracks smoothly but reticles sit off the real-world features
- FOV mismatch. The visible FOV (`visFovH`) must match the actual lens.
  Check the camera picker selected the intended lens and the FOV preset
  matches it. Nudge FOV ±2° until a known feature lines up.
- Confirm `updateVisibleFOV` is being called (it runs inside
  `ensureCanvasSized`) and that the video has non-zero
  `videoWidth/videoHeight` by the time it does.

### Composite frames are geometrically offset / doubled
- This is assembly geometry, never blending. Work through §9.
- First suspect: FOV. Second: roll — confirm `cap.roll` is populated
  and `placeCapture` builds the basis with it. Third: lens distortion /
  parallax — inherent; use the desktop stitch.

### Composite frame is rotated relative to its neighbours
- Roll. Check `eulerToLook` returns a sensible `roll`, that
  `attemptCapture` stores `state.roll`, and that `placeCapture` passes
  `cap.roll` to `cameraBasis`. The level bubble reflects the same roll;
  if the bubble looks wrong the roll maths is wrong.

### No motion events at all (often Android)
- Run `diagnostic.html`. Check: HTTPS (sensors are blocked on
  non-secure origins), Chrome site-settings → Motion sensors allowed,
  and on iOS that the permission prompt was accepted.

### Camera shows the wrong lens, or Chrome/Firefox differ
- Multi-lens phone. Use the Setup → Camera → Detect picker to lock one
  lens by `deviceId`. `diagnostic.html` lists every lens with its
  probed resolution and a guessed FOV.

### Auto-capture never fires
- `checkAutoCapture`: the aim must be within `state.tol` for >380 ms.
  Near the poles azimuth tolerance is relaxed/ignored (azimuth is
  meaningless there). Raise the tolerance in Setup if the user cannot
  hold steady enough.

### PWA will not install
- PWAs require HTTPS. `file://` and plain HTTP will not work. Service
  worker registration also requires HTTPS. See README.md for hosts.

### Stale app after an update
- Bump `CACHE` in `sw.js` (currently `theodolite-v6`). The service
  worker is cache-first for assets; a new cache name forces a refresh.

### General technique
- `diagnostic.html` is the first stop for any sensor or camera issue —
  it is standalone, needs no rebuild, and produces a copyable log.
- The maths primitives (`eulerToLook`, `cameraBasis`) are pure
  functions and can be unit-tested in isolation with Node. The key
  invariant to assert: `cameraBasis` at `roll=0` matches the plain
  az/el rotation, and `eulerToLook` → `cameraBasis` round-trips a
  rotation matrix to machine precision away from the poles.

---

## 11. Build, deploy, and the diagnostic page

There is no build step. To deploy, host the folder on any static HTTPS
host (GitHub Pages, Cloudflare Pages, Netlify, surge). HTTPS is
mandatory — the camera, the motion sensors and the service worker all
require a secure context. `localhost` is a secure context for desktop
development but the phone needs a real HTTPS URL it can reach.

`diagnostic.html` is a standalone page (no dependencies, no service
worker interaction) for testing a specific device. It checks the secure
context, the orientation events and their rate, runs a 7-second sensor
**stability** test (raw vs look-vector jitter, drift, gimbal-lock
detection), a **compass-vs-gyro** comparison that recommends a tracking
source, and probes every camera lens individually. "Run everything &
diagnose" runs all checks and prints a plain-language verdict.

---

## 12. Code map

Everything is in `index.html` inside one IIFE. Rough order:

| Area                  | Functions |
|-----------------------|-----------|
| PWA / install         | service worker registration, `beforeinstallprompt` |
| State                 | the `state` object |
| Modes                 | `MODES`, `renderModes` |
| Setup UI              | `setFov`, camera: `classifyCamera` / `detectCameras` / `renderCameraList` / `selectCamera` |
| Session start         | `startCapture`, `stopStreams`, `abortCapture` |
| Orientation maths     | `smoothScalar`, `smoothAngle`, `eulerToLook`, `cameraBasis` |
| Orientation events    | `onOrientationAbsolute`, `onOrientationRelative`, `onOrientation` |
| FOV / canvas          | `updateVisibleFOV`, `ensureCanvasSized` |
| Render loop / HUD     | `loop`, `updateReadout`, `angDiff`, `projectTarget`, `drawHud`, `drawLevelBar` |
| Capture               | `checkAutoCapture`, `performInitialCapture`, `attemptCapture`, `updateProgress` |
| Finish / compositing  | `finishCapture`, `composeEquirectangular`, `placeCapture`, `renderFisheye` |
| Output                | `timestamp`, `downloadComposite`, `downloadZip`, `readmeText` |

When changing projection maths, the functions that must stay mutually
consistent are `eulerToLook`, `cameraBasis`, `projectTarget`,
`placeCapture` and `renderFisheye` — they all share the §4.1 world
frame and the §4.5 projection primitive.
