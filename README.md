# J.A.R.V.I.S. Holographic Dashboard

A browser-based, Iron-Man-style holographic interface: a face-recognition
login gate followed by a rotating 3D "wire sphere" command dashboard that
you can drive entirely hands-free with webcam hand-gesture tracking.

Everything runs **100% client-side** — no backend, no server, no API keys.
All computer vision (face recognition and hand tracking) happens locally
in your browser using TensorFlow.js / MediaPipe models. Nothing is
uploaded anywhere.

![status](https://img.shields.io/badge/status-experimental-orange)
![stack](https://img.shields.io/badge/stack-vanilla%20JS%20%2B%20WebGL-blue)
![license](https://img.shields.io/badge/license-MIT-green)

---

## ✨ Features

### 🔐 Biometric login (`index.html`)
- Real, client-side face recognition using [`face-api.js`](https://github.com/justadudewhohacks/face-api.js) (TensorFlow.js under the hood).
- **Automatic sign-up**: if no operator is registered yet, the page detects
  your face, asks for a name once, captures five samples, and saves an
  averaged 128-dimension face descriptor to `localStorage`.
- **Automatic sign-in**: on return visits, your face is matched against
  every stored profile (Euclidean distance on the face descriptor) and you're
  signed in with no typing at all.
- Multi-operator support — anyone can enroll; JARVIS recognizes each person
  by name (à la Tony / Pepper / Rhodey each getting their own greeting).
- Cinematic HUD presentation: animated scan reticle locked to your detected
  face position, a live diagnostic boot log, and a full-screen "ACCESS
  GRANTED" flash transition into the dashboard.
- Graceful fallbacks: camera/model failures show a clear reason and a
  **guest bypass** so the dashboard is always reachable. A **reset profiles**
  control is available for testing/demos.

### 🌐 Holographic dashboard (`dashboard.html`)
- A tangled wire-frame amber sphere (Three.js / WebGL) modeled after the
  Iron Man hologram — 58 randomized great-circle rings, additive-blended
  glow, a hot reactor-style core light, and ~250 scattered "circuit" flecks.
- A cinematic **power-up intro**: on load, every node materializes with a
  staggered pop-in, the wire rings unfurl outward, and the reactor core
  ramps up from dark — instead of just appearing fully formed.
- **Traveling data-pulse particles** continuously ride the node network's
  connector lines, like packets moving across a live circuit.
- Two slim counter-rotating accent rings orbit the sphere for extra depth,
  and the camera drifts very slightly on its own for a less static feel.
- Expanding "ping" ripples flash wherever you select or drop a node.
- 42 clickable system nodes (Arc Reactor, Flight Stabilizers, Threat
  Analysis, AI Core, Cybersecurity Firewall, etc.), each opening a detail
  panel with live status, integrity/throughput/signal stats, and an
  activity log.
- **Drag-and-drop**: pick up and reposition any node, either by clicking
  and holding it with a mouse, or with the aim-then-pinch hand gesture
  described below — connector lines and glow follow the node live.
- Mouse/touch controls: drag empty space to rotate, scroll/pinch to zoom,
  click a node to select it, click-drag a node to move it.
- Animated telemetry meters, a live clock, and full HUD chrome (corner
  brackets, status strip, operator badge with sign-out).

### ✋ Hand-gesture control (webcam, no mouse required)
A second real-time model ([MediaPipe Hands](https://developers.google.com/mediapipe))
tracks 21 hand landmarks and classifies your hand into one of four shapes
every frame:

| Gesture | Shape | Action |
|---|---|---|
| **Open palm**, move hand | all 4 fingers extended | Rotate the sphere |
| **Pinch claw**, spread/close thumb + index | middle/ring/pinky curled | Zoom in/out (continuous, like a phone pinch) |
| **Two-finger aim**, point index + middle together | "peace sign", ring/pinky curled | Aim a reticle; hold it on a node ~0.6s to open it |
| **Aim, then pinch** | two-finger aim → pinch claw on the same node | **Grab and drag that node** through 3D space; release the pinch to drop it |
| **Fist** | everything curled | Close the detail panel |

Dragging works with a mouse too — click and hold directly on any node to
pick it up, move it anywhere on the sphere, and let go to drop it.
Connector lines and the reactive glow follow the node in real time either
way.

The two-finger aim gesture is deliberately distinct from the pinch claw
(which requires the middle finger curled) so a full pinch-to-zoom motion —
fingers spread wide apart — can never be misread as an aim/select
gesture partway through.

Every gesture signal is smoothed (exponential moving average) and
debounced (a shape must win several consecutive frames before it's acted
on), and all motion (mouse, scroll, and gesture alike) is routed through a
single "target + damping" system so rotation and zoom always ease smoothly
regardless of which input drove them.

A live "optical sensor" panel shows your hand skeleton, current gesture,
and a dwell-progress ring so you can see exactly what's about to trigger
before it does.

---

## 📁 Project structure

```
.
├── index.html       # Biometric login / enrollment gate
├── dashboard.html    # The 3D holographic dashboard
└── README.md
```

Open `index.html` first — on success (or as a guest) it redirects to
`dashboard.html?user=<name>`. You can also open `dashboard.html` directly
to skip the login screen entirely; it will just show `OPERATOR: GUEST`.

---

## 🚀 Getting started

### Requirements
- A modern browser (Chrome, Edge, or Firefox recommended) with WebGL and
  `getUserMedia` support.
- A webcam, if you want face login and/or hand-gesture control. Everything
  else (mouse/touch control of the dashboard) works without a camera.

### Important: run it from a local server, not `file://`
Camera access (`getUserMedia`) requires a **secure context**. Most
browsers do *not* treat a double-clicked `file://` page as secure, so the
webcam will silently fail to prompt for permission. Serve the folder
instead:

```bash
# from the project folder
python3 -m http.server 8000
# then open:
# http://localhost:8000/index.html
```

Any other static server works too (`npx serve`, VS Code's Live Server,
GitHub Pages, Netlify, Vercel, etc.) — `localhost` and `https://` both
count as secure contexts.

### First run
1. Open `index.html`. It boots the recognition models (a few seconds — the
   face recognition network is the largest download, ~6 MB) and requests
   camera access.
2. With no operators registered yet, show your face — you'll be prompted
   for a name and the system captures five samples to build your profile.
3. You're signed in automatically and redirected to the dashboard.
4. Reload the page any time afterward and you'll be recognized and signed
   in automatically, no typing required.
5. On the dashboard, click **ENABLE GESTURE CONTROL** (bottom-left panel)
   to turn on hand tracking, or just use your mouse/trackpad.

---

## 🧠 How the recognition/tracking pipelines work

### Face recognition
1. [`face-api.js`](https://github.com/justadudewhohacks/face-api.js) loads
   three models from a public CDN mirror of the project's weights:
   `tinyFaceDetector` (face localization), `faceLandmark68TinyNet`
   (landmark alignment), and `faceRecognitionNet` (128-d descriptor
   extraction).
2. Every ~380ms, one frame from the webcam is run through the pipeline to
   get a descriptor.
3. The descriptor is compared (Euclidean distance) against every stored
   profile in `localStorage`. A distance below `0.55` counts as a match,
   and two consecutive matching frames are required before access is
   granted (avoids one-off false positives).
4. Enrollment averages five descriptors captured a third of a second apart
   into one reference vector, which is more robust to a single bad frame
   (blink, motion blur, etc.) than a single sample would be.

### Hand gestures
1. [MediaPipe Hands](https://developers.google.com/mediapipe/solutions/vision/hand_landmarker)
   returns 21 (x, y, z) landmarks per hand, normalized to the video frame.
2. Finger "extended" state is computed as a **ratio of distances from the
   wrist** (tip vs. its pip joint), not raw position — this makes the
   classifier orientation-independent, so a sideways or rotated hand still
   classifies correctly.
3. Pinch distance is normalized by the wrist-to-knuckle span (the hand's
   own scale), so zoom sensitivity is consistent whether your hand is
   close to or far from the camera.
4. A 3-frame debounce and exponential smoothing on every continuous signal
   (pinch ratio, palm position, aim-cursor position) keep the motion
   stable despite frame-to-frame tracking jitter.

---

## 🔒 Privacy

- Face descriptors and hand-tracking data **never leave your browser** —
  there is no backend, no analytics, and no network calls other than
  fetching the static model weight files on first load (which are cached
  by the browser afterward).
- Enrolled face profiles are stored only in this browser's `localStorage`,
  scoped to whatever origin you serve the page from. Clearing site data
  (or the **RESET ALL PROFILES** button on the login page) removes them
  permanently.
- Nothing is stored in cookies, and no third party ever receives your
  camera feed.

---

## 🛠️ Customization

**Add or edit dashboard nodes** — edit the `NAMES` array near the top of
the script in `dashboard.html`; each `[name, category]` pair automatically
gets placed on the sphere and wired into the detail panel with a
category-based description.

**Tune gesture sensitivity** — in `dashboard.html`, look for the constants
block near the gesture code:
```js
const PINCH_SPLIT     = 0.62; // lower = requires a tighter pinch to trigger zoom mode
const DEBOUNCE_FRAMES = 3;    // higher = more stable but slightly more latency
const ZOOM_SENSITIVITY   = 460;
const ROTATE_SENSITIVITY = 3.0;
const DWELL_MS = 600;         // how long to hold the two-finger aim on a node before it opens
const GRAB_AIM_GRACE_MS = 550; // how recently you must have aimed at a node for a pinch to grab it
```

**Tune the intro / animation feel** — near the render loop:
```js
const INTRO_MS = 1500;   // total boot/assembly animation length
const PULSE_COUNT = 16;  // how many traveling data-pulse particles ride the network
```

**Tune face-match strictness** — in `index.html`:
```js
const MATCH_THRESHOLD = 0.55;       // lower = stricter (fewer false accepts, more false rejects)
const MATCH_CONFIRM_FRAMES = 2;
const ENROLL_SAMPLES = 5;
```

**Re-theme the colors** — both files use CSS custom properties at the top
of the `<style>` block (`--amber`, `--amber-bright`, `--red`, etc.) — swap
the palette to restyle the whole HUD in one place.

---

## ⚠️ Known limitations

- **Sandboxed previews** (e.g. viewing this inside a chat product's inline
  artifact preview) typically block camera access entirely for security
  reasons. Download the files and serve them yourself as described above
  to use face login or gesture control. Mouse/touch control of the
  dashboard doesn't require a camera and works everywhere.
- Face recognition here is a lightweight, browser-only demo — good for a
  personal project or portfolio piece, but it has not been hardened
  against spoofing (e.g. a photo held up to the camera) and shouldn't be
  used to protect anything sensitive.
- Model downloads (~6–7 MB total, dominated by the face recognition
  network) mean the first login on a slow connection can take a few
  seconds; the browser caches them after that.
- Gesture recognition works best with clear, deliberate hand shapes and
  reasonable, even lighting. Very fast motion or a hand partially out of
  frame can momentarily drop tracking — this is inherent to the underlying
  model, not the gesture-classification logic.

---

## 🧰 Tech stack

| Piece | Library | Purpose |
|---|---|---|
| 3D rendering | [Three.js](https://threejs.org/) (r128) | Wire sphere, nodes, camera, lighting |
| Hand tracking | [MediaPipe Hands](https://developers.google.com/mediapipe) | 21-point hand landmark detection |
| Face recognition | [face-api.js](https://github.com/justadudewhohacks/face-api.js) | Face detection, landmarks, 128-d descriptors |
| Fonts | [Orbitron](https://fonts.google.com/specimen/Orbitron), [Share Tech Mono](https://fonts.google.com/specimen/Share+Tech+Mono) | HUD typography |

No build step, no bundler, no `npm install` — every dependency loads from
a CDN via a `<script>` tag.

---

## 📄 License

MIT — do whatever you'd like with this, a credit back to the repo is
appreciated but not required.

---

## 🙏 Credits

Visually inspired by the JARVIS interface from the *Iron Man* films
(Marvel Studios). This is an independent fan-made technical project, not
affiliated with or endorsed by Marvel or Disney.
