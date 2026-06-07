# Interactive Journey: 2D Map ↔ 3D Globe — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone `~/journey-globe/index.html` that morphs a 2D dotted world map into a 3D dotted globe, with #156BF3 location pins that reveal hover/tap story-cards, chronological journey arcs, and Apple-HIG-compliant motion.

**Architecture:** One self-contained HTML file. Three.js + OrbitControls via ESM importmap from a CDN (no build step). A single GPU `THREE.Points` field stores two positions per dot (`aFlat`, `aSphere`) and interpolates between them in a vertex shader driven by a `uMorph` uniform. Pins are 5 small CPU-updated meshes that lerp with the same eased morph value; raycasting drives an HTML popover card. Everything is organized into labeled comment sections inside one `<script type="module">`.

**Tech Stack:** HTML/CSS, vanilla JS (ESM), Three.js 0.169.0 (CDN), GLSL (inline shaders). Verification is manual browser QA (per the design spec §12), driven via the `/browse` skill.

> **Note on verification:** This is a visual WebGL artifact — there is no unit-test harness (the spec deliberately chose a single self-contained file with manual QA). Each task therefore ends with a concrete **browser verification** (a specific observable outcome) instead of a `pytest`-style assertion. The one place automated checking is feasible — the pure geometry helpers — uses inline `console.assert` dev checks (Task 3).
>
> **Note on commits:** The user's standing preference is "commit only when asked." Commit steps are written out per task for completeness; before the first commit, confirm with the user (or batch/skip commits if they prefer). Task 1 includes `git init` for the new folder.

**Reference for behavior:** https://designspells.com/spells/2d-map-morphs-into-3d-globe-in-vercel
**HIG:** https://developer.apple.com/design/human-interface-guidelines

---

## File Structure

| File | Responsibility |
|---|---|
| `~/journey-globe/index.html` | The entire app: markup (toggle + card), CSS, and one ESM module with labeled sections (`CONSTANTS`, `DATA`, `GEO HELPERS`, `SCENE`, `DOTFIELD`, `PINS`, `ARCS`, `INTERACTION`, `CARD`, `ANIMATE`, `BOOT`). |
| `~/journey-globe/docs/superpowers/specs/2026-06-06-map-globe-journey-design.md` | (Exists) The approved design spec. |
| `~/journey-globe/docs/superpowers/plans/2026-06-06-map-globe-journey.md` | (This file) The plan. |

All implementation tasks modify the single `index.html`. Later tasks insert code into the labeled comment sections created in Task 1, so placement is unambiguous without re-pasting the whole file.

**Serving (used by every verification step):**
```bash
cd ~/journey-globe && python3 -m http.server 8080
# then open http://localhost:8080/  (use the /browse skill)
```
A local server is required so the CDN modules and the cross-origin land texture load with correct CORS.

---

## Task 1: Project skeleton + dark canvas

**Files:**
- Create: `~/journey-globe/index.html`

- [ ] **Step 1: Create the HTML skeleton with importmap and labeled module sections**

Create `~/journey-globe/index.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
  <title>Lingkan — Journey</title>
  <style>
    :root { --accent: #156BF3; }
    * { box-sizing: border-box; }
    html, body { margin: 0; height: 100%; background: #05070d; overflow: hidden; }
    body { font-family: -apple-system, BlinkMacSystemFont, "SF Pro Text", "Segoe UI", system-ui, sans-serif;
           color: #e8edf6; -webkit-font-smoothing: antialiased; }
    #scene { position: fixed; inset: 0; display: block; width: 100%; height: 100%; }
  </style>
  <script type="importmap">
  {
    "imports": {
      "three": "https://unpkg.com/three@0.169.0/build/three.module.js",
      "three/addons/": "https://unpkg.com/three@0.169.0/examples/jsm/"
    }
  }
  </script>
</head>
<body>
  <canvas id="scene"></canvas>

  <script type="module">
  import * as THREE from 'three';
  import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

  // === CONSTANTS ===
  // === DATA ===
  // === GEO HELPERS ===
  // === SCENE ===
  // === DOTFIELD ===
  // === PINS ===
  // === ARCS ===
  // === INTERACTION ===
  // === CARD ===
  // === ANIMATE ===
  // === BOOT ===

  console.log('three', THREE.REVISION);
  </script>
</body>
</html>
```

- [ ] **Step 2: Initialize git for the new folder**

```bash
cd ~/journey-globe && git init -q && printf "node_modules\n.DS_Store\n" > .gitignore
```

- [ ] **Step 3: Verify it loads**

Run the server (see top of plan) and open `http://localhost:8080/` via `/browse`.
Expected: a solid near-black page, **no console errors**, and a console line like `three 169`.

- [ ] **Step 4: Commit** *(confirm with user first — see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "chore: scaffold journey-globe (importmap + dark canvas)"
```

---

## Task 2: Scene, camera, lights, controls, render loop

**Files:**
- Modify: `~/journey-globe/index.html` (`SCENE`, `ANIMATE`, `BOOT` sections)

- [ ] **Step 1: Fill the `SCENE` section**

Replace the `// === SCENE ===` line with:

```js
  // === SCENE ===
  const canvas = document.getElementById('scene');
  const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setClearColor(0x05070d, 1);

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 1, 4000);
  camera.position.set(0, 0, 280);

  scene.add(new THREE.AmbientLight(0xffffff, 0.9));
  const key = new THREE.DirectionalLight(0xffffff, 0.6);
  key.position.set(1, 1, 1);
  scene.add(key);

  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.08;
  controls.enablePan = false;
  controls.enableZoom = false;          // zoom is off in v1 (spec §7)
  controls.rotateSpeed = 0.5;
  controls.autoRotate = true;
  controls.autoRotateSpeed = 0.6;       // ~10°/s

  function onResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }
  window.addEventListener('resize', onResize);
```

- [ ] **Step 2: Fill the `ANIMATE` and `BOOT` sections**

Replace `// === ANIMATE ===` and `// === BOOT ===` with:

```js
  // === ANIMATE ===
  function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
  }

  // === BOOT ===
  animate();
```

Delete the temporary `console.log('three', THREE.REVISION);` line.

- [ ] **Step 3: Verify**

Reload `http://localhost:8080/`.
Expected: dark scene, no errors, window resizes without distortion. (Nothing is drawn yet — that's expected.)

- [ ] **Step 4: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: scene, camera, lights, orbit controls, render loop"
```

---

## Task 3: Data + geometry helpers (with inline dev assertions)

**Files:**
- Modify: `~/journey-globe/index.html` (`CONSTANTS`, `DATA`, `GEO HELPERS` sections)

- [ ] **Step 1: Fill `CONSTANTS`**

Replace `// === CONSTANTS ===` with:

```js
  // === CONSTANTS ===
  const ACCENT = '#156BF3';
  const ACCENT_HEX = 0x156BF3;
  const FIELD_COLOR = 0x8fa8d4;   // cool off-white land dots
  const R = 100;                  // globe radius
  const DEG = Math.PI / 180;
  const reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
  const isTouch = window.matchMedia('(hover: none), (pointer: coarse)').matches;
```

- [ ] **Step 2: Fill `DATA` with the locations + journey order**

Replace `// === DATA ===` with:

```js
  // === DATA ===  (the only block to edit for content)
  const LOCATIONS = [
    { id: 'yangzhou', city: 'Yangzhou, Jiangsu', lat: 32.39, lng: 119.41,
      experiences: [
        { org: 'Born here', role: '', dates: '2001.08', blurb: 'Where the story starts.' },
      ] },
    { id: 'beijing', city: 'Beijing', lat: 39.90, lng: 116.40,
      experiences: [
        { org: '北京化工大学', role: 'Undergraduate', dates: '2019.08–2023.06', blurb: 'Undergraduate — where design began.' },
        { org: '清华大学未来实验室', role: 'Research', dates: '2023.08–2024.03', blurb: 'Research at the Future Lab, Tsinghua.' },
        { org: '快手 Kuaishou', role: 'Product Designer Intern', dates: '2024.05–2024.08', blurb: 'Product design for creators at scale.' },
      ] },
    { id: 'suzhou', city: 'Suzhou', lat: 31.30, lng: 120.58,
      experiences: [
        { org: 'Ecovacs Robotics', role: 'Product Designer Intern', dates: '2024.03–2024.05', blurb: 'Product design for home robotics.' },
      ] },
    { id: 'pittsburgh', city: 'Pittsburgh, USA', lat: 40.44, lng: -79.99,
      experiences: [
        { org: 'Carnegie Mellon · HCI Institute', role: "Master's, HCI", dates: '2024.08–2025.12', blurb: 'Studying human-computer interaction.' },
      ] },
    { id: 'sanjose', city: 'San Jose, USA', lat: 37.34, lng: -121.89,
      experiences: [
        { org: 'Cyan Studio', role: 'Product Designer', dates: '2025.12–present', blurb: 'Designing products and teaching.' },
      ] },
  ];

  // Chronological journey (Beijing visited twice → appears twice).
  const JOURNEY = [
    { lat: 32.39, lng: 119.41 }, // Yangzhou (born)
    { lat: 39.90, lng: 116.40 }, // Beijing (化工大学 / 清华)
    { lat: 31.30, lng: 120.58 }, // Suzhou (Ecovacs)
    { lat: 39.90, lng: 116.40 }, // Beijing (快手)
    { lat: 40.44, lng: -79.99 }, // Pittsburgh (CMU)
    { lat: 37.34, lng: -121.89 }, // San Jose (Cyan)
  ];
```

- [ ] **Step 3: Fill `GEO HELPERS` and add dev assertions**

Replace `// === GEO HELPERS ===` with:

```js
  // === GEO HELPERS ===
  // Sphere position for a lat/lng (standard equirectangular-texture mapping).
  function latLngToSphere(lat, lng, r = R) {
    const phi = (90 - lat) * DEG;
    const theta = (lng + 180) * DEG;
    return new THREE.Vector3(
      -r * Math.sin(phi) * Math.cos(theta),
       r * Math.cos(phi),
       r * Math.sin(phi) * Math.sin(theta)
    );
  }
  // Flat (equirectangular plane) position; scaled so width = sphere circumference.
  function latLngToFlat(lat, lng) {
    return new THREE.Vector3(lng * DEG * R, lat * DEG * R, 0);
  }
  // Smoothstep ease (used for the morph).
  function ease(t) { return t * t * (3 - 2 * t); }

  // --- dev assertions (visible in console; harmless in production) ---
  (function devChecks() {
    const eq = (a, b, eps = 1e-6) => Math.abs(a - b) < eps;
    const n = latLngToSphere(90, 0);            // north pole → +Y, radius R
    console.assert(eq(n.y, R) && eq(n.x, 0, 1e-4) && eq(n.z, 0, 1e-4), 'north pole maps to +Y');
    const len = latLngToSphere(12.3, 45.6).length();
    console.assert(eq(len, R, 1e-4), 'sphere points have radius R');
    const f = latLngToFlat(0, 180);
    console.assert(eq(f.x, Math.PI * R, 1e-4) && eq(f.y, 0), 'flat right edge at +πR');
    console.assert(eq(ease(0), 0) && eq(ease(1), 1) && eq(ease(0.5), 0.5), 'ease endpoints/mid');
    console.log('%cgeo dev checks passed', 'color:#156BF3');
  })();
```

- [ ] **Step 4: Verify**

Reload. Expected console output: **`geo dev checks passed`** and **no `Assertion failed` messages**.

- [ ] **Step 5: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: locations data + geo helpers with dev assertions"
```

---

## Task 4: Dot field — sample land texture, render the flat 2D map

**Files:**
- Modify: `~/journey-globe/index.html` (`DOTFIELD` section, `BOOT` section)

- [ ] **Step 1: Fill `DOTFIELD` with sampling + a flat-rendered point field**

Replace `// === DOTFIELD ===` with:

```js
  // === DOTFIELD ===
  let field = null;
  const fieldData = { flats: [], spheres: [] };

  function loadImage(src) {
    return new Promise((res, rej) => {
      const im = new Image();
      im.crossOrigin = 'anonymous';
      im.onload = () => res(im);
      im.onerror = rej;
      im.src = src;
    });
  }

  async function buildDotField() {
    const img = await loadImage('https://unpkg.com/three-globe/example/img/earth-blue-marble.jpg');
    const cw = 400, ch = 200;                 // sample resolution (2:1 equirectangular)
    const cnv = document.createElement('canvas');
    cnv.width = cw; cnv.height = ch;
    const ctx = cnv.getContext('2d', { willReadFrequently: true });
    ctx.drawImage(img, 0, 0, cw, ch);
    const data = ctx.getImageData(0, 0, cw, ch).data;

    const step = 2;                           // every Nth pixel → dot density
    const flats = [], spheres = [];
    for (let y = 0; y < ch; y += step) {
      for (let x = 0; x < cw; x += step) {
        const i = (y * cw + x) * 4;
        const r = data[i], g = data[i + 1], b = data[i + 2];
        // earth-blue-marble: ocean is blue-dominant; land is warmer & not too dark.
        const isLand = (r + g) > b * 1.6 && (r + g + b) > 60;   // QA-tunable threshold
        if (!isLand) continue;
        const lng = (x / cw) * 360 - 180;
        const lat = 90 - (y / ch) * 180;
        const f = latLngToFlat(lat, lng);
        const s = latLngToSphere(lat, lng);
        flats.push(f.x, f.y, f.z);
        spheres.push(s.x, s.y, s.z);
      }
    }
    fieldData.flats = flats;
    fieldData.spheres = spheres;
    console.log('dots:', flats.length / 3);

    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.Float32BufferAttribute(flats, 3)); // FLAT for now
    const mat = new THREE.PointsMaterial({ color: FIELD_COLOR, size: 1.6, sizeAttenuation: true });
    field = new THREE.Points(geo, mat);
    field.frustumCulled = false;
    scene.add(field);
  }
```

- [ ] **Step 2: Boot the field and frame the flat map**

Replace the `BOOT` section body (`animate();`) with:

```js
  // === BOOT ===
  controls.autoRotate = false;            // temporary: view the flat map straight-on
  camera.position.set(0, 0, 760);         // frame the flat plane
  buildDotField();
  animate();
```

- [ ] **Step 3: Verify**

Reload. Expected: a **flat dotted world map** with recognizable continents, console logs a dot count (~4–9k). If continents look wrong/empty, adjust the `isLand` threshold or `step` and reload (QA tuning, expected).

- [ ] **Step 4: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: dot field sampled from land texture (flat map)"
```

---

## Task 5: Morph shader + default globe

**Files:**
- Modify: `~/journey-globe/index.html` (`DOTFIELD`, `INTERACTION`, `ANIMATE`, `BOOT`)

- [ ] **Step 1: Add a shared morph state object in `INTERACTION`**

Replace `// === INTERACTION ===` with:

```js
  // === INTERACTION ===
  const state = {
    mode: 'globe',         // 'globe' | 'map'
    morph: 1,              // 0 = flat map, 1 = globe (current, animated)
    morphAnim: null,       // {from, to, start, dur}
    camZ: 280,             // current camera distance (animated)
    camZTarget: 280,
  };
```

- [ ] **Step 2: Replace the field material with the morph ShaderMaterial**

In `buildDotField()`, replace the geometry/material/Points block (everything from `const geo = new THREE.BufferGeometry();` to `scene.add(field);`) with:

```js
    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.Float32BufferAttribute(spheres, 3)); // bounds only
    geo.setAttribute('aFlat', new THREE.Float32BufferAttribute(flats, 3));
    geo.setAttribute('aSphere', new THREE.Float32BufferAttribute(spheres, 3));

    const mat = new THREE.ShaderMaterial({
      transparent: true,
      uniforms: {
        uMorph: { value: 1 },
        uColor: { value: new THREE.Color(FIELD_COLOR) },
        uSize:  { value: 2.4 },
        uScale: { value: window.innerHeight / 2 },
      },
      vertexShader: `
        attribute vec3 aFlat;
        attribute vec3 aSphere;
        uniform float uMorph;
        uniform float uSize;
        uniform float uScale;
        float ease(float t){ return t*t*(3.0-2.0*t); }
        void main(){
          vec3 p = mix(aFlat, aSphere, ease(uMorph));
          vec4 mv = modelViewMatrix * vec4(p, 1.0);
          gl_PointSize = uSize * (uScale / -mv.z);
          gl_Position = projectionMatrix * mv;
        }
      `,
      fragmentShader: `
        uniform vec3 uColor;
        void main(){
          vec2 c = gl_PointCoord - vec2(0.5);
          if (dot(c, c) > 0.25) discard;     // round dots
          gl_FragColor = vec4(uColor, 0.92);
        }
      `,
    });
    field = new THREE.Points(geo, mat);
    field.frustumCulled = false;
    scene.add(field);
```

- [ ] **Step 3: Drive the uniform from `state.morph` each frame**

In `animate()`, add — right after `controls.update();`:

```js
    if (field) field.material.uniforms.uMorph.value = state.morph;
```

- [ ] **Step 4: Restore globe framing in `BOOT`**

Replace the `BOOT` section body with:

```js
  // === BOOT ===
  camera.position.set(0, 0, 280);
  controls.autoRotate = !reducedMotion;
  buildDotField();
  animate();
```

- [ ] **Step 5: Verify**

Reload. Expected: a **3D dotted globe** (not flat), gently auto-rotating (unless reduced-motion), and **drag rotates** it. No console errors. Dots stay round and attenuate with distance.

- [ ] **Step 6: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: vertex-shader morph; default 3D globe"
```

---

## Task 6: Toggle control + animated map↔globe switch

**Files:**
- Modify: `~/journey-globe/index.html` (markup, CSS, `INTERACTION`, `ANIMATE`)

- [ ] **Step 1: Add the segmented control markup**

Immediately after `<canvas id="scene"></canvas>`, add:

```html
  <div id="ui" role="group" aria-label="View">
    <div class="segmented">
      <button id="seg-map" type="button" aria-pressed="false">Map</button>
      <button id="seg-globe" type="button" aria-pressed="true">Globe</button>
    </div>
  </div>
```

- [ ] **Step 2: Add CSS for the control**

Inside `<style>`, before the closing `</style>`, add:

```css
    #ui { position: fixed; top: max(16px, env(safe-area-inset-top)); left: 50%;
          transform: translateX(-50%); z-index: 10; }
    .segmented { display: inline-flex; padding: 3px; gap: 2px; border-radius: 999px;
                 background: rgba(255,255,255,0.08); backdrop-filter: blur(12px);
                 border: 1px solid rgba(255,255,255,0.12); }
    .segmented button { appearance: none; border: 0; cursor: pointer; color: #cdd6e8;
                        font: inherit; font-size: 14px; line-height: 1; padding: 9px 18px;
                        border-radius: 999px; background: transparent;
                        transition: background .25s ease, color .25s ease; }
    .segmented button[aria-pressed="true"] { background: var(--accent); color: #fff; }
    .segmented button:focus-visible { outline: 2px solid #fff; outline-offset: 2px; }
    @media (prefers-reduced-motion: reduce) { .segmented button { transition: none; } }
```

- [ ] **Step 3: Add `setMode()` + the camera/morph tween logic in `INTERACTION`**

Append to the `INTERACTION` section:

```js
  const segMap = document.getElementById('seg-map');
  const segGlobe = document.getElementById('seg-globe');

  function setMode(mode) {
    if (mode === state.mode) return;
    state.mode = mode;
    const to = mode === 'globe' ? 1 : 0;
    state.morphAnim = { from: state.morph, to, start: performance.now(), dur: reducedMotion ? 0 : 1200 };
    state.camZTarget = mode === 'globe' ? 280 : 760;
    controls.enableRotate = mode === 'globe';
    controls.autoRotate = mode === 'globe' && !reducedMotion;
    segGlobe.setAttribute('aria-pressed', String(mode === 'globe'));
    segMap.setAttribute('aria-pressed', String(mode === 'map'));
  }
  segMap.addEventListener('click', () => setMode('map'));
  segGlobe.addEventListener('click', () => setMode('globe'));
```

- [ ] **Step 4: Advance the tweens each frame in `animate()`**

In `animate()`, replace the line `if (field) field.material.uniforms.uMorph.value = state.morph;` with:

```js
    if (state.morphAnim) {
      const a = state.morphAnim;
      const t = a.dur ? Math.min(1, (performance.now() - a.start) / a.dur) : 1;
      state.morph = a.from + (a.to - a.from) * ease(t);
      if (t >= 1) state.morphAnim = null;
    }
    state.camZ += (state.camZTarget - state.camZ) * (reducedMotion ? 1 : 0.08);
    camera.position.setLength(state.camZ);     // keep direction, change distance
    if (field) field.material.uniforms.uMorph.value = state.morph;
```

- [ ] **Step 5: Verify**

Reload. Expected: tapping **Map** smoothly flattens the globe into the 2D map (~1.2s) and the map is straight-on & non-rotating; tapping **Globe** morphs back and resumes auto-rotation. The pressed segment is highlighted in #156BF3. Under OS reduced-motion, the switch is near-instant. No errors.

- [ ] **Step 6: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: Map/Globe segmented toggle with morph + camera tween"
```

---

## Task 7: Location pins (#156BF3, glow, idle pulse, morphing)

**Files:**
- Modify: `~/journey-globe/index.html` (`PINS`, `ANIMATE`, `BOOT`)

- [ ] **Step 1: Fill the `PINS` section**

Replace `// === PINS ===` with:

```js
  // === PINS ===
  const pins = [];   // { mesh, glow, flat, sphere, loc, phase }

  function makeGlowTexture() {
    const s = 64;
    const c = document.createElement('canvas'); c.width = c.height = s;
    const g = c.getContext('2d');
    const grd = g.createRadialGradient(s/2, s/2, 0, s/2, s/2, s/2);
    grd.addColorStop(0, 'rgba(255,255,255,0.9)');
    grd.addColorStop(0.25, 'rgba(21,107,243,0.8)');
    grd.addColorStop(1, 'rgba(21,107,243,0)');
    g.fillStyle = grd; g.fillRect(0, 0, s, s);
    return new THREE.CanvasTexture(c);
  }

  function buildPins() {
    const glowTex = makeGlowTexture();
    const pinGeo = new THREE.SphereGeometry(1.9, 16, 16);
    const pinMat = new THREE.MeshBasicMaterial({ color: ACCENT_HEX });
    LOCATIONS.forEach((loc, idx) => {
      const mesh = new THREE.Mesh(pinGeo, pinMat.clone());
      mesh.userData.locId = loc.id;
      const glow = new THREE.Sprite(new THREE.SpriteMaterial({
        map: glowTex, color: ACCENT_HEX, transparent: true, opacity: 0.55, depthWrite: false,
      }));
      glow.scale.setScalar(11);
      mesh.add(glow);
      scene.add(mesh);
      pins.push({
        mesh, glow, loc, phase: idx * 1.7,
        flat: latLngToFlat(loc.lat, loc.lng),
        sphere: latLngToSphere(loc.lat, loc.lng),
      });
    });
  }

  function updatePins(now) {
    const e = ease(state.morph);
    for (const p of pins) {
      p.mesh.position.copy(p.flat).lerp(p.sphere, e);
      const pulse = reducedMotion ? 1 : 1 + Math.sin(now * 0.004 + p.phase) * 0.14;
      p.mesh.scale.setScalar(pulse);
    }
  }

  // True when the pin is on the camera-facing side of the globe.
  function isPinFrontFacing(p) {
    if (state.morph < 0.5) return true;                 // flat map: all face camera
    const normal = p.sphere.clone().normalize();
    const toCam = camera.position.clone().sub(p.mesh.position).normalize();
    return normal.dot(toCam) > 0.0;
  }
```

- [ ] **Step 2: Build pins on boot, update each frame**

In `BOOT`, add `buildPins();` right after `buildDotField();`.

In `animate()`, add right before `renderer.render(scene, camera);`:

```js
    updatePins(performance.now());
```

- [ ] **Step 3: Verify**

Reload. Expected: **5 glowing #156BF3 pins** at Yangzhou, Beijing, Suzhou, Pittsburgh, San Jose, gently pulsing (steady under reduced-motion). They sit on the globe and **travel correctly** to the flat map when you toggle. Spot-check positions against the cities. No errors.

- [ ] **Step 4: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: #156BF3 location pins with glow, pulse, morph, facing test"
```

---

## Task 8: Raycast hover detection (mouse) + cursor feedback

**Files:**
- Modify: `~/journey-globe/index.html` (`INTERACTION`, CSS)

- [ ] **Step 1: Add raycasting + hover plumbing in `INTERACTION`**

Append to the `INTERACTION` section:

```js
  const raycaster = new THREE.Raycaster();
  const ndc = new THREE.Vector2();
  let hovered = null;     // current pin object or null

  function pickPin(clientX, clientY) {
    ndc.x = (clientX / window.innerWidth) * 2 - 1;
    ndc.y = -(clientY / window.innerHeight) * 2 + 1;
    raycaster.setFromCamera(ndc, camera);
    const hits = raycaster.intersectObjects(pins.map(p => p.mesh), false);
    for (const h of hits) {
      const p = pins.find(pp => pp.mesh === h.object);
      if (p && isPinFrontFacing(p)) return p;
    }
    return null;
  }

  // Mouse path only; touch handled in Task 10.
  if (!isTouch) {
    renderer.domElement.addEventListener('pointermove', (ev) => {
      if (ev.pointerType === 'touch') return;
      const p = pickPin(ev.clientX, ev.clientY);
      renderer.domElement.style.cursor = p ? 'pointer' : '';
      if (p !== hovered) {
        hovered = p;
        if (p) showCard(p);            // hover mode (not locked)
        else hideCard();
      }
    });
    renderer.domElement.addEventListener('pointerleave', () => {
      hovered = null; renderer.domElement.style.cursor = ''; hideCard();
    });
  }
```

- [ ] **Step 2: Add temporary stubs so the page runs before Task 9**

Append to `INTERACTION` (these are **replaced** in Task 9 — they only keep the page error-free now):

```js
  // TEMP stubs — replaced by the real CARD section in Task 9.
  function showCard(p) { console.log('hover pin:', p.loc.id); }
  function hideCard() {}
```

- [ ] **Step 3: Verify**

Reload (desktop). Expected: moving the mouse over a pin shows a **pointer cursor** and logs e.g. `hover pin: beijing`; pins on the **far side of the globe do not trigger** the log; moving off clears it. No errors.

- [ ] **Step 4: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: raycast hover detection with front-facing cull (mouse)"
```

---

## Task 9: Story-card popover (HTML), with Beijing stacked

**Files:**
- Modify: `~/journey-globe/index.html` (markup, CSS, `CARD`, `INTERACTION`, `ANIMATE`)

- [ ] **Step 1: Add the card markup**

After the `#ui` block, add:

```html
  <div id="card" role="dialog" aria-label="Place details" hidden>
    <button id="card-close" type="button" aria-label="Close" hidden>×</button>
    <div id="card-photo" aria-hidden="true"></div>
    <div id="card-body"></div>
  </div>
```

- [ ] **Step 2: Add card CSS (HIG-style popover + spring)**

Inside `<style>`, add:

```css
    #card { position: fixed; z-index: 20; width: 268px; max-width: calc(100vw - 24px);
            background: rgba(20,24,33,0.86); backdrop-filter: blur(20px) saturate(140%);
            border: 1px solid rgba(255,255,255,0.12); border-radius: 16px; overflow: hidden;
            box-shadow: 0 12px 40px rgba(0,0,0,0.5);
            opacity: 0; transform: scale(0.96); transform-origin: var(--ox,50%) var(--oy,100%);
            transition: opacity .22s ease, transform .26s cubic-bezier(.2,.9,.3,1.2);
            pointer-events: none; }
    #card.show { opacity: 1; transform: scale(1); }
    #card.touch { pointer-events: auto; }
    #card-photo { aspect-ratio: 16 / 10; width: 100%;
                  background: linear-gradient(135deg, rgba(21,107,243,0.35), rgba(21,107,243,0.05));
                  border-bottom: 1px solid rgba(255,255,255,0.08);
                  display: grid; place-items: center; color: rgba(255,255,255,0.4); font-size: 12px; }
    #card-photo::after { content: 'photo'; }
    #card-body { padding: 14px 16px 16px; }
    .card-city { font-size: 12px; letter-spacing: .04em; text-transform: uppercase;
                 color: var(--accent); margin: 0 0 8px; font-weight: 600; }
    .card-exp { padding: 8px 0; border-top: 1px solid rgba(255,255,255,0.07); }
    .card-exp:first-of-type { border-top: 0; padding-top: 0; }
    .card-org { font-size: 15px; font-weight: 600; margin: 0; color: #f2f5fb; }
    .card-role { font-size: 13px; margin: 2px 0 0; color: #aeb9cd; }
    .card-dates { font-size: 12px; margin: 2px 0 0; color: #7f8aa0; }
    .card-blurb { font-size: 13px; line-height: 1.45; margin: 6px 0 0; color: #cdd6e8; }
    #card-close { position: absolute; top: 6px; right: 6px; width: 30px; height: 30px;
                  border: 0; border-radius: 50%; background: rgba(0,0,0,0.35); color: #fff;
                  font-size: 20px; line-height: 1; cursor: pointer; z-index: 1; }
    @media (prefers-reduced-motion: reduce) { #card { transition: opacity .12s ease; transform: none; } #card.show { transform: none; } }
```

- [ ] **Step 3: Replace the TEMP stubs (Task 8 Step 2) with the real `CARD` section**

First **delete** the two TEMP stub functions added in Task 8 Step 2. Then replace `// === CARD ===` with:

```js
  // === CARD ===
  const cardEl = document.getElementById('card');
  const cardBody = document.getElementById('card-body');
  const cardClose = document.getElementById('card-close');
  let cardPin = null;        // pin currently shown (for per-frame repositioning)
  let cardLocked = false;    // touch: stays open until dismissed

  function renderCardBody(loc) {
    const exps = loc.experiences.map(e => `
      <div class="card-exp">
        <p class="card-org">${e.org}</p>
        ${e.role ? `<p class="card-role">${e.role}</p>` : ''}
        <p class="card-dates">${e.dates}</p>
        <p class="card-blurb">${e.blurb}</p>
      </div>`).join('');
    cardBody.innerHTML = `<p class="card-city">${loc.city}</p>${exps}`;
  }

  function projectToScreen(v3) {
    const v = v3.clone().project(camera);
    return { x: (v.x * 0.5 + 0.5) * window.innerWidth, y: (-v.y * 0.5 + 0.5) * window.innerHeight };
  }

  function positionCard() {
    if (!cardPin) return;
    const s = projectToScreen(cardPin.mesh.position);
    const w = cardEl.offsetWidth, h = cardEl.offsetHeight, gap = 14, pad = 12;
    // Prefer above the pin; flip below if it would clip the top.
    let top = s.y - h - gap, oy = '100%';
    if (top < pad) { top = s.y + gap; oy = '0%'; }
    let left = s.x - w / 2;
    left = Math.max(pad, Math.min(left, window.innerWidth - w - pad));
    const ox = `${Math.max(0, Math.min(100, ((s.x - left) / w) * 100))}%`;
    cardEl.style.left = `${left}px`;
    cardEl.style.top = `${Math.max(pad, top)}px`;
    cardEl.style.setProperty('--ox', ox);
    cardEl.style.setProperty('--oy', oy);
  }

  function showCard(pin, locked = false) {
    cardPin = pin;
    cardLocked = locked;
    renderCardBody(pin.loc);
    cardEl.hidden = false;
    cardClose.hidden = !locked;
    cardEl.classList.toggle('touch', locked);
    controls.autoRotate = false;                 // pause rotation while open
    positionCard();
    requestAnimationFrame(() => cardEl.classList.add('show'));
  }

  function hideCard() {
    if (!cardPin) return;
    cardPin = null; cardLocked = false;
    cardEl.classList.remove('show');
    cardClose.hidden = true;
    if (state.mode === 'globe' && !reducedMotion) controls.autoRotate = true;
    setTimeout(() => { if (!cardPin) cardEl.hidden = true; }, 240);
  }

  cardClose.addEventListener('click', hideCard);
```

- [ ] **Step 4: Reposition the card every frame (so it tracks the pin)**

In `animate()`, add right after `updatePins(performance.now());`:

```js
    if (cardPin) positionCard();
```

- [ ] **Step 5: Verify**

Reload (desktop). Expected: hovering a pin shows a card that **fades + springs up** anchored to the pin; the **Beijing** card lists all three experiences (化工大学 → 清华未来实验室 → 快手) in order; a placeholder "photo" block sits on top; the card flips below the pin near the top edge and never clips off-screen; auto-rotation **pauses** while a card is open and resumes after. Reduced-motion shows a quick fade with no scale. No errors.

- [ ] **Step 6: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: HIG story-card popover; Beijing stacked; pin-tracked positioning"
```

---

## Task 10: Touch / tap path

**Files:**
- Modify: `~/journey-globe/index.html` (`INTERACTION`)

- [ ] **Step 1: Add tap-to-open / tap-outside-to-dismiss for touch**

Append to the `INTERACTION` section:

```js
  if (isTouch) {
    renderer.domElement.addEventListener('pointerdown', (ev) => {
      if (ev.pointerType === 'mouse') return;
      const p = pickPin(ev.clientX, ev.clientY);
      if (p) {
        if (cardPin === p) hideCard();
        else showCard(p, true);          // locked = touch mode (shows ×, stays open)
      } else if (cardPin && cardLocked) {
        hideCard();                       // tap empty space dismisses
      }
    }, { passive: true });
  }
```

- [ ] **Step 2: Verify**

In `/browse`, enable touch emulation (or test on a touch device). Expected: **tapping a pin opens** its card with a visible **× button**; tapping the same pin again or tapping empty space **dismisses** it; the card is interactive (× tappable). Mouse behavior on desktop is unchanged. No errors.

- [ ] **Step 3: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: touch tap-to-open / tap-outside-to-dismiss"
```

---

## Task 11: Journey arcs (chronological, globe-only)

**Files:**
- Modify: `~/journey-globe/index.html` (`ARCS`, `ANIMATE`, `BOOT`)

- [ ] **Step 1: Fill the `ARCS` section**

Replace `// === ARCS ===` with:

```js
  // === ARCS ===
  const arcs = [];   // { line, total, drawn }

  function greatCirclePoints(aLatLng, bLatLng, steps = 80) {
    const a = latLngToSphere(aLatLng.lat, aLatLng.lng, 1).normalize();
    const b = latLngToSphere(bLatLng.lat, bLatLng.lng, 1).normalize();
    const omega = Math.acos(THREE.MathUtils.clamp(a.dot(b), -1, 1));
    const sinO = Math.sin(omega) || 1e-6;
    const pts = [];
    for (let i = 0; i <= steps; i++) {
      const t = i / steps;
      const k0 = Math.sin((1 - t) * omega) / sinO;
      const k1 = Math.sin(t * omega) / sinO;
      const dir = a.clone().multiplyScalar(k0).add(b.clone().multiplyScalar(k1)).normalize();
      const lift = 1 + 0.18 * Math.sin(Math.PI * t);   // bow outward over the surface
      pts.push(dir.multiplyScalar(R * lift));
    }
    return pts;
  }

  function buildArcs() {
    for (let i = 0; i < JOURNEY.length - 1; i++) {
      const pts = greatCirclePoints(JOURNEY[i], JOURNEY[i + 1]);
      const geo = new THREE.BufferGeometry().setFromPoints(pts);
      const mat = new THREE.LineBasicMaterial({ color: ACCENT_HEX, transparent: true, opacity: 0.85 });
      const line = new THREE.Line(geo, mat);
      line.frustumCulled = false;
      geo.setDrawRange(0, reducedMotion ? pts.length : 0);   // animate draw-on
      scene.add(line);
      arcs.push({ line, total: pts.length, drawn: reducedMotion ? pts.length : 0 });
    }
  }

  function updateArcs() {
    const onGlobe = state.morph;                 // 1 on globe, 0 on map
    for (const a of arcs) {
      a.line.material.opacity = 0.85 * onGlobe;  // fade out in map mode
      a.line.visible = onGlobe > 0.02;
      if (!reducedMotion && a.drawn < a.total && state.mode === 'globe') {
        a.drawn = Math.min(a.total, a.drawn + 1.2);
        a.line.geometry.setDrawRange(0, Math.floor(a.drawn));
      }
    }
  }
```

- [ ] **Step 2: Build + update arcs**

In `BOOT`, add `buildArcs();` right after `buildPins();`.
In `animate()`, add right after the `if (cardPin) positionCard();` line:

```js
    updateArcs();
```

- [ ] **Step 3: Verify**

Reload. Expected: thin **#156BF3 arcs** draw on across the globe connecting Yangzhou→Beijing→Suzhou→Beijing→Pittsburgh→San Jose, bowing above the surface; switching to **Map fades the arcs out**, switching back shows them. Reduced-motion shows them fully drawn immediately. No errors.

- [ ] **Step 4: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: chronological great-circle journey arcs (globe-only)"
```

---

## Task 12: Edge cases — WebGL fallback, resize robustness

**Files:**
- Modify: `~/journey-globe/index.html` (markup, CSS, `BOOT`, `SCENE`)

- [ ] **Step 1: Add a fallback element**

After the `#card` block, add:

```html
  <div id="fallback" hidden>
    <p>This experience needs WebGL.</p>
    <p>Please open it in a modern browser with hardware acceleration enabled.</p>
  </div>
```

- [ ] **Step 2: Add fallback CSS**

Inside `<style>`, add:

```css
    #fallback { position: fixed; inset: 0; z-index: 30; display: grid; place-content: center;
                text-align: center; gap: 6px; padding: 24px; background: #05070d; color: #aeb9cd; }
    #fallback p { margin: 0; }
```

- [ ] **Step 3: Guard boot with a WebGL check**

Replace the entire `BOOT` section with:

```js
  // === BOOT ===
  function webglAvailable() {
    try {
      const c = document.createElement('canvas');
      return !!(window.WebGLRenderingContext && (c.getContext('webgl') || c.getContext('experimental-webgl')));
    } catch (e) { return false; }
  }

  if (!webglAvailable()) {
    document.getElementById('fallback').hidden = false;
    document.getElementById('ui').style.display = 'none';
  } else {
    camera.position.set(0, 0, 280);
    controls.autoRotate = !reducedMotion;
    buildDotField();
    buildPins();
    buildArcs();
    animate();
  }
```

- [ ] **Step 4: Keep the shader point-size uniform correct on resize**

In `onResize()` (in the `SCENE` section), add as the last line inside the function:

```js
    if (field) field.material.uniforms.uScale.value = window.innerHeight / 2;
```

- [ ] **Step 5: Verify**

Reload normally — everything still works. Then in `/browse` devtools, disable WebGL (or set the renderer creation to throw) and reload: expected the **fallback message** shows and the toggle is hidden. Re-enable, resize the window aggressively: dots/pins/card stay correctly framed, point sizes don't jump. No errors.

- [ ] **Step 6: Commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "feat: WebGL fallback + resize robustness"
```

---

## Task 13: Final QA pass (spec §12 checklist)

**Files:**
- Possibly modify: `~/journey-globe/index.html` (tuning only)

- [ ] **Step 1: Run the full QA checklist via `/browse`**

Serve and open the page. Confirm each item; fix any tuning issues inline (threshold, sizes, timings) and note what changed:

1. Morph plays smoothly **both directions** via the toggle (~1.2s, eased).
2. **Hover** each pin (mouse) shows the correct card; **Beijing** shows the stacked card with all 3.
3. **Tap** path works under touch emulation; dismiss via outside-tap and ×.
4. **Back-facing** pins on the globe are not hoverable.
5. `prefers-reduced-motion` → near-instant morph, no idle pulse, no auto-rotate, arcs pre-drawn.
6. **Resize** re-fits without breaking layout or card anchoring.
7. **WebGL-disabled** fallback renders the message.
8. Continents are recognizable; pin positions match their cities; arcs follow the journey order.
9. Accent color is exactly **#156BF3** on pins, arcs, and the active toggle segment.

- [ ] **Step 2: Final commit** *(see header note)*

```bash
cd ~/journey-globe && git add -A && git commit -q -m "polish: final QA tuning pass"
```

---

## Self-Review (completed by plan author)

**Spec coverage:** §1 summary → all tasks; §3 tech (CDN importmap) → Task 1; §4 data model → Task 3; §5 modules → labeled sections across tasks; §6 morph → Tasks 4–6; §7 view states/toggle/zoom-off → Tasks 2, 6; §8 pins/arcs/card incl. Beijing stack, back-face cull, touch → Tasks 7–11; §9 HIG/reduced-motion → Tasks 6, 7, 9 + throughout; §10 look & feel → Tasks 1, 3, 7; §11 edge cases (WebGL, resize, perf cap) → Tasks 2, 12; §12 verification checklist → Task 13. No gaps.

**Placeholder scan:** No TBD/TODO. The Task 8 "TEMP stubs" are explicitly created and then explicitly deleted/replaced in Task 9 Step 3 (intentional, not a placeholder). Land-threshold/dot-density/timings are real values flagged as QA-tunable, not placeholders.

**Type/name consistency:** Verified shared names across tasks — `state` (`.mode`/`.morph`/`.morphAnim`/`.camZ`/`.camZTarget`), `field`, `fieldData`, attributes `aFlat`/`aSphere`, uniform `uMorph`/`uColor`/`uSize`/`uScale`, `ease()`, `latLngToSphere()`/`latLngToFlat()`, `pins`/`updatePins()`/`isPinFrontFacing()`, `showCard(pin, locked)`/`hideCard()`/`positionCard()`/`cardPin`/`cardLocked`, `pickPin()`, `arcs`/`buildArcs()`/`updateArcs()`. `showCard` signature: Task 9 defines `(pin, locked=false)`. A bug was caught here — Task 8's mouse hover originally called `showCard(p, ev.clientX, ev.clientY)`, which would pass `clientX` (truthy) as `locked` and wrongly lock the hover card. **Fixed inline:** Task 8 Step 1 now calls `showCard(p)`.
</content>
