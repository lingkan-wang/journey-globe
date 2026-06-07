# Interactive Journey: 2D Map ↔ 3D Globe

**Date:** 2026-06-06
**Status:** Approved design — ready for implementation plan
**Deliverable:** A single self-contained `index.html` in `~/journey-globe/`

---

## 1. Summary

Recreate Vercel's "2D dotted map morphs into a 3D globe" interaction
(reference: https://designspells.com/spells/2d-map-morphs-into-3d-globe-in-vercel),
then extend it: each of the author's life/career locations is a highlighted
**#156BF3** pin, and hovering (or tapping) a pin reveals a story-card with a
photo placeholder and a short blurb. All motion follows Apple's Human Interface
Guidelines (https://developer.apple.com/design/human-interface-guidelines).

The headline effect is a **real geometric morph** (not a crossfade): one GPU
point cloud where every dot interpolates between a flat equirectangular position
and a position on a sphere.

## 2. Goals & non-goals

**Goals**
- Faithful, smooth map↔globe morph driven by a single shader uniform.
- 5 location pins in #156BF3 with hover/tap story-cards.
- HIG-compliant motion, affordances, reduced-motion + touch handling.
- Zero build step: one HTML file, Three.js via CDN importmap.

**Non-goals (v1)**
- Free zoom / deep map exploration (zoom is off).
- Real photos (placeholders only).
- Full keyboard navigation of individual pins (the toggle is keyboard-accessible;
  per-pin keyboard focus is a possible later enhancement).
- Integration into the Next.js portfolio (build standalone first; port later).

## 3. Tech approach

- **Three.js** + **OrbitControls**, loaded via ESM `importmap` from a CDN.
- A single `<script type="module">` organized into the modules in §5.
- No bundler, no `package.json`. The file can be opened directly or served by
  any static server (a local server is recommended so the embedded assets load
  cleanly).

## 4. Data model

```js
// Each location has lat/lng and one-or-more experiences (Beijing has 3).
const LOCATIONS = [
  { id: 'yangzhou',   city: 'Yangzhou, Jiangsu', lat: 32.39, lng: 119.41,
    experiences: [
      { org: 'Born here', role: '', dates: '2001.08',
        blurb: 'Where the story starts.', photo: 'placeholder' },
    ]},
  { id: 'beijing',    city: 'Beijing', lat: 39.90, lng: 116.40,
    experiences: [
      { org: '北京化工大学',          role: 'Undergraduate',            dates: '2019.08–2023.06', blurb: 'Undergraduate — where design began.', photo: 'placeholder' },
      { org: '清华大学未来实验室',     role: 'Research',                 dates: '2023.08–2024.03', blurb: 'Research at the Future Lab, Tsinghua.', photo: 'placeholder' },
      { org: '快手 Kuaishou',         role: 'Product Designer Intern',  dates: '2024.05–2024.08', blurb: 'Product design for creators at scale.', photo: 'placeholder' },
    ]},
  { id: 'suzhou',     city: 'Suzhou', lat: 31.30, lng: 120.58,
    experiences: [
      { org: 'Ecovacs Robotics', role: 'Product Designer Intern', dates: '2024.03–2024.05', blurb: 'Product design for home robotics.', photo: 'placeholder' },
    ]},
  { id: 'pittsburgh', city: 'Pittsburgh, USA', lat: 40.44, lng: -79.99,
    experiences: [
      { org: 'Carnegie Mellon · HCI Institute', role: "Master's, HCI", dates: '2024.08–2025.12', blurb: 'Studying human-computer interaction.', photo: 'placeholder' },
    ]},
  { id: 'sanjose',    city: 'San Jose, USA', lat: 37.34, lng: -121.89,
    experiences: [
      { org: 'Cyan Studio', role: 'Product Designer', dates: '2025.12–present', blurb: 'Designing products and teaching.', photo: 'placeholder' },
    ]},
];
```

Chronological arc order (for §8): yangzhou → beijing → suzhou → beijing →
pittsburgh → sanjose. (Beijing is visited twice; the two arc segments to/from
Suzhou are both drawn.)

Accent color constant: `ACCENT = '#156BF3'`.

## 5. Modules (single file)

| Module | Responsibility |
|---|---|
| `data` | The `LOCATIONS` array + `ACCENT`. The only thing the author edits for content. |
| `scene` | Renderer, perspective camera, lights, resize handling, render loop. |
| `dotField` | Generate land dots from an embedded land-mask; build `THREE.Points` with `aFlat`/`aSphere` attributes + morph shader. |
| `pins` | Build the 5 highlighted pin objects (glow + idle pulse), positioned by lat/lng, morphing with the field. |
| `arcs` | Build chronological great-circle arcs between cities; animate draw-on; globe-only visibility. |
| `interaction` | Toggle handling (morph + camera tween), auto-rotation, drag, raycast hover/tap, back-face culling, card anchoring. |
| `card` | The HTML story popover: build, fill, position, show/hide, touch dismiss. |

## 6. The morph (core)

- **Dot seeding:** sample a compact embedded equirectangular land-mask bitmap on
  a grid; keep a dot where the pixel is land. Target ~6,000–8,000 dots, tuned for
  60fps on a laptop.
- **Two positions per dot:** `aFlat` (x,y on the equirectangular plane, z≈0) and
  `aSphere` (lat/lng → unit-sphere xyz × radius).
- **Shader:** a `uMorph` uniform (0 = flat, 1 = globe) lerps `mix(aFlat, aSphere, eased)`
  in the vertex shader. Point size attenuated by depth on the globe.
- **Easing:** `uMorph` is tweened by JS over **~1.2s, ease-in-out**; reduced-motion
  jumps near-instant.

## 7. View states & toggle

- **Default resting state: globe.** (Trivially flipped to start-as-map.)
- **Segmented control "Map | Globe"** (HIG styling) in a corner; it is a real,
  keyboard-focusable control.
- **Globe state:** auto-rotation ~10°/s; drag-to-spin via OrbitControls (rotate
  only). **Map state:** front-on flat view, rotation locked.
- Toggling tweens `uMorph` and transitions the camera between the two framings.
- **Zoom disabled** in v1.

## 8. Pins, arcs, and hover card

**Pins**
- Larger than field dots, **#156BF3**, soft glow + gentle idle pulse (pulse
  disabled under reduced-motion).
- Pointer cursor + slight scale-up on hover as feedback.
- On the globe, **back-facing pins are non-interactive** (dot(normal, viewDir)
  test) until rotated into view.

**Arcs**
- Thin **#156BF3** great-circle arcs in chronological order; animated draw-on.
- Visible on the globe, faded out on the map. Included by default.

**Hover/tap card (the added interaction)**
- HIG-style **popover** anchored beside the pin: placeholder photo (16:10), then
  place, role/org, dates, one-line blurb.
- **Beijing** renders a **stacked card** with all three experiences in time order.
- **Motion:** fade + scale-up from the pin, soft spring ~0.25s; reverse on leave.
  Auto-reposition to avoid clipping off-screen.
- **Auto-rotation pauses** while a card is open.
- **Touch:** tap a pin to open; tap outside or an × button to dismiss
  (pointer-type detection — hover path for mouse, tap path for touch).

## 9. Apple HIG adherence

- Purposeful, eased, interruptible motion (clarity / deference / depth).
- `prefers-reduced-motion`: near-instant morph, no idle pulse, no auto-rotate.
- Generous touch targets; obvious affordances on toggle + pins; immediate
  feedback on every interaction; legible card contrast.

## 10. Look & feel

- **Deep near-black / navy background**, off-white field dots, **#156BF3**
  highlights / arcs / accents. Adjustable to a light theme later.

## 11. Edge cases & errors

- **No WebGL:** graceful fallback — show a static map image + short message.
- **Resize:** renderer + camera re-fit; card repositions or closes.
- **Performance:** dot count tuned; pixel ratio capped (e.g. ≤2).
- **Reduced motion / touch:** handled per §8–§9.

## 12. Verification (manual QA)

No unit tests (visual standalone). QA in a browser (driven via `/browse`):
1. Morph plays smoothly both directions via the toggle.
2. Hover each pin (mouse) shows the correct card; Beijing shows the stacked card.
3. Tap path works under touch emulation; dismiss via outside-tap and ×.
4. Back-facing pins on the globe are not hoverable.
5. `prefers-reduced-motion` removes/reduces motion as specified.
6. Window resize re-fits without breaking layout or card anchoring.
7. WebGL-disabled fallback renders the static message.

## 13. Open items / future

- Real photos replace placeholders.
- Optional light theme.
- Optional per-pin keyboard navigation + a visually-hidden location list for a11y.
- Possible port into `~/lingkan-portfolio` (`/about` or `/playground`) as a
  react-three-fiber component.
