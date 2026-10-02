# J.A.R.V.I.S. Holographic Dashboard

A polished, browser-only J.A.R.V.I.S.-inspired interface with a biometric access screen and an interactive Three.js command sphere. The project combines client-side face recognition, webcam hand tracking, pointer/touch interaction, animated telemetry, and a cinematic HUD without a backend or build pipeline.

> **Important:** the biometric screen is a demonstration feature, not a production authentication system. It has no liveness or anti-spoofing protection, and stored face descriptors are not encrypted.

## ✨ Highlights

### 🔐 Biometric access screen

- Client-side face detection and recognition with `face-api.js`.
- First-run enrollment with five independent face descriptors per operator.
- Returning operators are compared against their stored enrollment samples using Euclidean descriptor distance.
- Three consecutive matching frames are required before access is granted.
- Legacy profiles from the earlier single-descriptor format are still accepted.
- Duplicate operator names update the existing profile instead of silently creating repeated entries.
- Profile data is validated before use and capped at eight local profiles.
- Camera/model failures fall back cleanly to guest access.
- Detection is serialized; a slow inference cannot overlap a later detection pass.
- Operator-controlled text is rendered with `textContent`, avoiding the stored-HTML injection issue present in the earlier diagnostic log implementation.
- Camera streams and timers are cleaned up when the page is left.

### 🌐 3D holographic dashboard

- Three.js/WebGL wire sphere with 42 interactive system nodes.
- Randomized great-circle wiring, accent rings, circuit flecks, reactor glow, and animated data pulses.
- Corrected staggered power-up sequence for nodes and glows.
- Frame-rate-independent auto-rotation, smoothing, and accent-ring animation.
- Pointer Events provide one input path for mouse, pen, and touch.
- Drag empty space to rotate the sphere.
- Drag a node to reposition it while its connector geometry updates live.
- Mouse wheel/trackpad scroll controls camera zoom.
- Duplicate nearest-neighbor edges are removed from the node network.
- Tooltips are constrained to the viewport instead of overflowing near screen edges.
- WebGL/library failures produce an explicit fallback message instead of failing silently.
- Detail panel supports `Escape` to close and exposes basic ARIA metadata.

### ✋ Webcam gesture controls

The dashboard can optionally use MediaPipe Hands to track one hand and classify four interaction states:

| Gesture | Detection rule | Action |
|---|---|---|
| Open palm | Index, middle, ring, and pinky extended | Rotate the sphere |
| Pinch claw | Middle/ring/pinky curled; thumb-index distance below threshold | Zoom continuously |
| Two-finger aim | Index + middle extended; ring + pinky curled | Aim at nodes and dwell to select |
| Aim → pinch | Pinch shortly after aiming at a node | Grab and reposition that node |
| Fist | Fingers curled | Close the detail panel |

Gesture signals use normalized hand measurements, debouncing, and exponential smoothing. Camera startup is guarded against duplicate requests, MediaPipe assets are version-pinned, and webcam/model resources are released when gesture mode is disabled or the page is left.

## 📁 Project structure

```text
.
├── index.html       # Biometric enrollment / recognition entry point
├── dashboard.html   # Interactive 3D dashboard and hand gestures
├── README.md        # Project documentation
└── LICENSE          # MIT license
```

There is intentionally no bundler, package manager, or build step. The project is suitable for any static web host.

## 🚀 Run locally

### Requirements

- A modern Chromium-, Firefox-, or WebKit-based browser with WebGL support.
- JavaScript enabled.
- Internet access on first load for CDN-hosted libraries, model files, and web fonts.
- A webcam for face recognition or gesture controls.
- HTTPS or `localhost` for camera APIs.

### Start a local server

Do **not** rely on double-clicking the HTML files with a `file://` URL. Browser camera APIs require a secure context.

From the project directory:

```bash
python -m http.server 8000
```

Then open:

```text
http://localhost:8000/index.html
```

Other static servers are also fine, for example:

```bash
npx serve .
```

or VS Code Live Server, GitHub Pages, Netlify, Vercel, Cloudflare Pages, and similar HTTPS hosts.

## 🧭 Usage

### First visit

1. Open `index.html` through `localhost` or HTTPS.
2. Allow camera access when prompted.
3. Wait for the face detector, landmark model, and recognition model to load.
4. If there are no profiles, center your face and enter an operator designation.
5. Select **BEGIN ENROLLMENT SCAN** and hold still while five samples are captured.
6. After enrollment, the dashboard opens automatically.

### Returning visit

1. Open `index.html`.
2. Look at the camera.
3. The system compares the current descriptor against each stored operator profile.
4. Three consecutive matches below the configured distance threshold grant access.

### Guest mode

Use **ENTER AS GUEST →** if you do not want to enroll a face or if camera/model loading is unavailable.

### Dashboard controls

| Input | Action |
|---|---|
| Drag empty space | Rotate sphere |
| Mouse wheel / trackpad scroll | Zoom |
| Click/tap node | Open node detail panel |
| Drag node | Reposition node |
| `Escape` | Close node detail panel |
| Enable Gesture Control | Start webcam hand tracking |
| Sign Out | Clear active session name and return to access screen |

## 🧠 Recognition implementation

### Face pipeline

The access page loads three `face-api.js` networks:

- `tinyFaceDetector`
- `faceLandmark68TinyNet`
- `faceRecognitionNet`

A scan produces a 128-value face descriptor. During enrollment, five descriptors are stored for the operator. During recognition, the current descriptor is compared with every descriptor in each profile. The three smallest distances for that profile are averaged (or all available distances for legacy profiles with fewer samples), and the profile with the lowest aggregate distance is considered the candidate.

Current defaults:

```js
matchThreshold: 0.55,
matchConfirmFrames: 3,
noMatchGraceMs: 3200,
enrollSamples: 5,
detectIntervalMs: 380,
maxProfiles: 8
```

The UI reports **match distance**, not a percentage "confidence" value. Euclidean embedding distance is not a calibrated probability and should not be presented as one.

### Stored profile format

New profiles are stored in `localStorage` under `jarvis_profiles`:

```json
{
  "name": "OPERATOR",
  "descriptors": [
    [0.01, -0.02, 0.03],
    [0.02, -0.01, 0.04]
  ],
  "enrolledAt": "2026-10-03T00:00:00.000Z"
}
```

The example above truncates each descriptor for readability; real descriptors contain 128 values.

The loader also accepts the previous schema:

```json
{
  "name": "OPERATOR",
  "descriptor": [0.01, -0.02, 0.03],
  "enrolledAt": "..."
}
```

When that operator is re-enrolled, the profile is saved using the new multi-sample format.

## ✋ Gesture implementation

MediaPipe Hands provides 21 normalized landmarks for the detected hand. Gesture classification is based on hand-relative ratios rather than raw pixel distances:

- Finger extension compares fingertip distance from the wrist with the corresponding PIP-joint distance.
- Pinch distance is normalized by wrist-to-index-knuckle span.
- A gesture must persist for multiple frames before becoming active.
- Palm, pinch, and aiming coordinates use exponential moving averages.
- A two-finger aim and a pinch claw are structurally distinct, reducing accidental mode switches.
- A short aim-to-pinch grace window allows a node to be grabbed after targeting it.

Key tuning values are near the gesture section in `dashboard.html`:

```js
const EXTEND_MARGIN      = 1.15;
const PINCH_SPLIT        = 0.62;
const DEBOUNCE_FRAMES    = 3;
const EMA_PALM           = 0.5;
const EMA_PINCH          = 0.45;
const EMA_CURSOR         = 0.35;
const ZOOM_SENSITIVITY   = 460;
const ROTATE_SENSITIVITY = 3.0;
const DWELL_MS           = 600;
const GRAB_AIM_GRACE_MS  = 550;
```

## ⚙️ Customization

### Dashboard nodes

Edit the `NAMES` array in `dashboard.html`:

```js
const NAMES = [
  ["ARC REACTOR CORE", "POWER GENERATION"],
  ["SECONDARY POWER CELL", "POWER GENERATION"]
];
```

Node IDs, placement, status values, and category-based descriptions are generated automatically.

### Face-recognition strictness

Adjust the `CONFIG` object in `index.html`:

```js
matchThreshold: 0.55,
matchConfirmFrames: 3,
enrollSamples: 5
```

Lower match thresholds are stricter. Threshold selection is dataset-, model-, camera-, and environment-dependent; validate it against your own conditions instead of treating `0.55` as a universal security boundary.

### Dashboard motion

Useful motion constants include:

```js
const ROT_RESPONSE  = 9.0;
const ZOOM_RESPONSE = 10.5;
const INTRO_MS = 1500;
const PULSE_COUNT = 16;
```

Rotation and zoom smoothing now use delta-time-aware exponential response, so behavior is substantially more consistent across 60 Hz, 120 Hz, and 144 Hz displays.

### Theme

Both pages centralize the visual palette in CSS custom properties:

```css
--amber: #ffaa00;
--amber-bright: #ffe0a0;
--amber-hot: #fff2cf;
--amber-dim: #6b4400;
--red: #ff3b30;
--text-dim: #8a6a2a;
```

## 🔒 Privacy and security

### What stays local

Camera frames are passed directly to browser-side face/hand models. This project does not contain application code that uploads frames or descriptors to a project backend.

Stored operator profiles live in browser `localStorage` for the current origin. **RESET ALL PROFILES** removes the stored profiles and active-user value for that origin.

### What still uses the network

The project loads third-party static resources from CDNs:

- Google Fonts
- Three.js
- `face-api.js`
- face-api model weights
- MediaPipe Hands and its model/WASM assets

Your browser therefore makes normal HTTP requests to those CDN providers when resources are not already cached.

### Not production authentication

This project must not be treated as strong biometric access control:

- No liveness detection.
- No anti-spoofing check against photos, displays, or replayed video.
- Face descriptors are stored in `localStorage`, not encrypted secure storage.
- A client-side user can modify local storage and page code.
- There is no server-side identity, authorization, audit log, or session enforcement.

Use it as a UI/ML demonstration, portfolio project, kiosk prototype, or interaction experiment—not to protect sensitive data or physical access.

## ♿ Accessibility and resilience

The improved version includes:

- visible keyboard focus styles;
- `aria-live` status regions for major camera/gesture state changes;
- `Escape` support for the detail panel;
- `aria-pressed` state on the gesture toggle;
- reduced-motion handling through `prefers-reduced-motion`;
- responsive sizing for short/small screens;
- explicit messages for missing Three.js, WebGL, MediaPipe, camera APIs, model downloads, and blocked permissions;
- safe text rendering for names and logs.

The 3D node field itself remains a primarily visual interaction surface and is not a complete screen-reader equivalent. If this project needs production-grade accessibility, add a synchronized semantic node list with keyboard navigation and equivalent actions.

## 📦 External dependencies

| Component | Version | Purpose |
|---|---:|---|
| Three.js | r128 | WebGL scene, nodes, camera, lighting |
| face-api.js | 0.22.2 | Face detector, landmarks, descriptors |
| face-api.js model weights | 0.22.2 repository tag | Recognition model assets |
| MediaPipe Hands | 0.4.1675469240 | 21-point hand tracking |
| Orbitron / Share Tech Mono | hosted fonts | HUD typography |

The JavaScript library versions are explicitly pinned so a future CDN release cannot silently change runtime behavior.

## 🧪 Validation performed on this revision

The revised source was checked for:

- JavaScript syntax errors with `node --check` on both inline application scripts;
- duplicate HTML IDs;
- JavaScript `getElementById(...)` references pointing to missing elements;
- unsafe dynamic `innerHTML` usage in operator-controlled logging;
- lifecycle cleanup for camera streams and detection/gesture loops;
- pointer/mouse/touch interaction conflicts;
- frame-rate-dependent animation behavior;
- the node-intro milliseconds/seconds timing defect.

Full camera recognition and MediaPipe inference still require testing in a real secure browser context with a webcam because those capabilities depend on browser permission, hardware, lighting, and network-loaded model assets.

## 🧯 Troubleshooting

### Camera permission never appears

- Confirm the address is `https://...` or `http://localhost:...`.
- Check the browser's site-level camera permission.
- Make sure another application is not exclusively using the camera.
- Avoid sandboxed inline previews that do not delegate camera permission.

### Models fail to load

- Confirm internet connectivity.
- Check whether a content blocker, enterprise proxy, or firewall is blocking jsDelivr/CDNJS/Google Fonts.
- Open DevTools → Network and inspect failed model/script requests.

### Face is detected but never matches

- Re-enroll in similar lighting and camera placement.
- Keep the face large enough in frame.
- Adjust `matchThreshold` only after collecting representative genuine/impostor examples.

### Gesture control is unstable

- Use even lighting and a simple background.
- Keep one hand fully in frame.
- Make deliberate gestures rather than switching shapes rapidly.
- Increase `DEBOUNCE_FRAMES` for more stability at the cost of latency.
- Reduce `EMA_*` values for heavier smoothing.

### WebGL fallback appears

Enable hardware acceleration, update the browser/GPU driver, or test another browser/device with WebGL enabled.

## 🛠️ Deployment notes

Because this is a static project, deployment is simply the four files in this repository. No environment variables or API keys are required.

For a more controlled production-like deployment, consider self-hosting all third-party JavaScript/model/font assets and adding a strict Content Security Policy instead of depending on public CDNs.

## 📄 License

Released under the MIT License. See [`LICENSE`](LICENSE).

## 🙏 Attribution

The visual direction is inspired by fictional J.A.R.V.I.S.-style interfaces from the *Iron Man* films. This is an independent fan-made technical project and is not affiliated with or endorsed by Marvel Studios, Marvel, or Disney.
