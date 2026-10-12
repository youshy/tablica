# Lectern 3D Viewer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A static, double-clickable HTML page showing a polished, freely rotatable/pannable/zoomable 3D model of the black lectern with the MW brass badge.

**Architecture:** One `index.html` with an inline ES-module script (Three.js from jsdelivr via import map). Geometry is built procedurally; the original drawing is embedded once as a base64 data URI in `texture.js` (classic script, so it loads under `file://`) and mapped onto the top face and pedestal front with computed UVs — no image-cropping toolchain needed.

**Tech Stack:** Three.js 0.170.0 (`three`, `three/addons`: TrackballControls, RoundedBoxGeometry, RoomEnvironment), plain HTML/CSS, Chrome DevTools MCP for verification.

**Spec:** `docs/superpowers/specs/2026-10-04-lectern-3d-viewer-design.md`

**Deviation from spec (deliberate):** spec lists `textures/panel.jpg` + `textures/pedestal.jpg`. Python has no PIL here, and Chrome blocks WebGL textures from `file://` images. Instead: `texture.js` holds the whole source JPG as a data URI and UVs select the regions. Same visual result, works on double-click.

## Global Constraints

- Static files only, no build step, no server required (must work opened via `file://`).
- Three.js 0.170.0 from `https://cdn.jsdelivr.net/npm/three@0.170.0/`.
- Controls: `TrackballControls` — rotate on all axes (no locked up-axis), pan, zoom.
- "Reset view" button restores initial camera position, target, up vector, and model rotation.
- Slow idle auto-rotation that stops permanently on first user interaction.
- Approximate dimensions (meters): pedestal 0.80 W × 0.90 H × 0.50 D; top slab 1.60 W × 0.60 D × 0.04 thick, tilted 20°; plinth under pedestal.
- No git in this project (user's instruction) — no commit steps.

## Source image facts (949 × 960 px, `~/Downloads/Untitled.jpg`)

- Top face trapezoid (pixel coords, y down): TL (192,88), TR (741,88), BL (82,326), BR (843,326).
- Pedestal front: x 269–656, y 371–806 (inset 2 px from the measured 267–658 / 369–808 to avoid white edge).
- UV conversion: `u = px / 949`, `v = 1 - py / 960`.

## Review Focus

1. Opened by double-click (`file://`) — textures must render, not black/tainted; verified in Task 3 by loading the `file://` URL.
2. Viewer loses orientation after free rotation — Reset must restore camera position, up vector, target, and model rotation exactly; verified in Task 3.
3. Window resized / phone-portrait viewport — canvas fills window, no scrollbars, trackball still maps drags correctly (`handleResize`); verified in Task 3.
4. Zooming into the model or out to nothing — clamp with `minDistance`/`maxDistance`; verified in Task 3.
5. No internet / CDN blocked — page shows a readable message instead of a blank screen; verified in Task 3.

---

### Task 1: Embed the source image

**Files:**
- Create: `texture.js`

**Interfaces:**
- Produces: global `window.LECTERN_TEX` — string, `data:image/jpeg;base64,...` of the source drawing.

- [ ] **Step 1: Generate `texture.js`**

```bash
cd /Users/arturkondas/Desktop/git/tata/tablica
printf 'window.LECTERN_TEX="data:image/jpeg;base64,%s";\n' "$(base64 -i ~/Downloads/Untitled.jpg | tr -d '\n')" > texture.js
```

- [ ] **Step 2: Verify**

Run: `head -c 60 texture.js; echo; wc -c texture.js`
Expected: starts with `window.LECTERN_TEX="data:image/jpeg;base64,/9j/`, size ≈ 104 KB.

---

### Task 2: Page, scene, and model

**Files:**
- Create: `index.html`

**Interfaces:**
- Consumes: `window.LECTERN_TEX` (Task 1), loaded by `<script src="texture.js">` before the module script.
- Produces: `window.lectern = { camera, controls, model, reset }` for verification; `#reset` button; `#msg` element (hidden once the scene starts).

- [ ] **Step 1: Write `index.html`**

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Lectern 3D</title>
<style>
  :root { --bg-top: #f4f2ee; --bg-bottom: #d9d5cd; --ink: #1d1d1f; --btn: rgba(255,255,255,.85); }
  @media (prefers-color-scheme: dark) {
    :root { --bg-top: #2a2a2e; --bg-bottom: #0f0f11; --ink: #f2f2f2; --btn: rgba(40,40,44,.85); }
  }
  html, body { margin: 0; height: 100%; overflow: hidden; font: 14px/1.4 system-ui, sans-serif; color: var(--ink);
    background: radial-gradient(ellipse at 50% 35%, var(--bg-top), var(--bg-bottom)); }
  canvas { display: block; touch-action: none; }
  #ui { position: fixed; left: 16px; right: 16px; bottom: 16px; display: flex; gap: 12px; align-items: center;
    justify-content: space-between; pointer-events: none; }
  #hint { opacity: .7; }
  #reset { pointer-events: auto; border: 0; border-radius: 999px; padding: 10px 18px; font: inherit; color: inherit;
    background: var(--btn); box-shadow: 0 2px 10px rgba(0,0,0,.15); cursor: pointer; }
  #msg { position: fixed; inset: 0; display: grid; place-items: center; text-align: center; padding: 16px; }
</style>
</head>
<body>
<div id="msg">Loading 3D model…<br><small>Requires an internet connection for the 3D library.</small></div>
<div id="ui">
  <span id="hint">Drag to rotate · Right-drag to pan · Scroll / pinch to zoom</span>
  <button id="reset" type="button">Reset view</button>
</div>

<script src="texture.js"></script>
<script type="importmap">
{ "imports": {
  "three": "https://cdn.jsdelivr.net/npm/three@0.170.0/build/three.module.js",
  "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.170.0/examples/jsm/"
} }
</script>
<script type="module">
import * as THREE from 'three';
import { TrackballControls } from 'three/addons/controls/TrackballControls.js';
import { RoundedBoxGeometry } from 'three/addons/geometries/RoundedBoxGeometry.js';
import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js';

const IMG_W = 949, IMG_H = 960;
const TILT = THREE.MathUtils.degToRad(20);

// --- renderer / scene / camera ---
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.setSize(innerWidth, innerHeight);
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.prepend(renderer.domElement);

const scene = new THREE.Scene();
const pmrem = new THREE.PMREMGenerator(renderer);
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;

const camera = new THREE.PerspectiveCamera(35, innerWidth / innerHeight, 0.05, 50);

// --- lights ---
const key = new THREE.DirectionalLight(0xffffff, 2.2);
key.position.set(2.5, 4, 3);
key.castShadow = true;
key.shadow.mapSize.set(2048, 2048);
key.shadow.camera.left = key.shadow.camera.bottom = -1.5;
key.shadow.camera.right = key.shadow.camera.top = 1.5;
key.shadow.radius = 6;
scene.add(key);
const rim = new THREE.DirectionalLight(0xfff1e0, 1.2);
rim.position.set(-3, 2.5, -3);
scene.add(rim);
scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 0.4));

const ground = new THREE.Mesh(new THREE.PlaneGeometry(10, 10), new THREE.ShadowMaterial({ opacity: 0.25 }));
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);

// --- texture ---
const tex = new THREE.TextureLoader().load(window.LECTERN_TEX);
tex.colorSpace = THREE.SRGBColorSpace;
tex.anisotropy = renderer.capabilities.getMaxAnisotropy();

const toUV = (px, py) => [px / IMG_W, 1 - py / IMG_H];

// Plane whose UVs sample the trapezoid (tl, tr, bl, br) of the source image.
// ponytail: bilinear, exact for this flat drawing; use a homography if the source becomes a real photo.
function texturedPlane(w, h, tl, tr, bl, br, segX = 32, segY = 8) {
  const g = new THREE.PlaneGeometry(w, h, segX, segY);
  const uv = g.attributes.uv;
  for (let i = 0; i < uv.count; i++) {
    const u = uv.getX(i), v = uv.getY(i);
    const topX = tl[0] + (tr[0] - tl[0]) * u, botX = bl[0] + (br[0] - bl[0]) * u;
    const topY = tl[1] + (tr[1] - tl[1]) * u, botY = bl[1] + (br[1] - bl[1]) * u;
    uv.setXY(i, ...toUV(botX + (topX - botX) * v, botY + (topY - botY) * v));
  }
  return g;
}

// --- materials ---
const black = new THREE.MeshStandardMaterial({ color: 0x0b0b0c, roughness: 0.38, metalness: 0.1 });
const topMat = new THREE.MeshStandardMaterial({ map: tex, roughness: 0.45, metalness: 0.05 });
const frontMat = new THREE.MeshStandardMaterial({ map: tex, roughness: 0.32, metalness: 0.35 });

const shadowed = (m) => { m.castShadow = m.receiveShadow = true; return m; };

// --- model ---
const model = new THREE.Group();
scene.add(model);

const PLINTH_H = 0.04, PED_W = 0.80, PED_H = 0.90, PED_D = 0.50;
const PED_TOP = PLINTH_H + PED_H;

const plinth = shadowed(new THREE.Mesh(new RoundedBoxGeometry(PED_W + 0.06, PLINTH_H, PED_D + 0.06, 2, 0.008), black));
plinth.position.y = PLINTH_H / 2;
model.add(plinth);

const pedestal = shadowed(new THREE.Mesh(new RoundedBoxGeometry(PED_W, PED_H, PED_D, 3, 0.012), black));
pedestal.position.y = PLINTH_H + PED_H / 2;
model.add(pedestal);

const pedFront = new THREE.Mesh(
  texturedPlane(PED_W - 0.02, PED_H - 0.02, [269, 371], [656, 371], [269, 806], [656, 806], 1, 1), frontMat);
pedFront.position.set(0, pedestal.position.y, PED_D / 2 + 0.001);
model.add(pedFront);

// Slab: centre height chosen so its underside clears the pedestal's front-top edge.
const SLAB_W = 1.60, SLAB_D = 0.60, SLAB_T = 0.04;
const SLAB_Y = PED_TOP + 0.02 + SLAB_T / 2 + (PED_D / 2) * Math.tan(TILT);

// Wedge filling the gap between pedestal top and the tilted slab underside.
// Shape coords: (s, y) with s = -worldZ (rotateY(+90°) maps shape x -> -z).
const under = (z) => SLAB_Y - SLAB_T / 2 - z * Math.tan(TILT) + 0.01;
const wedgeShape = new THREE.Shape([
  new THREE.Vector2(-PED_D / 2 + 0.01, PED_TOP - 0.01),
  new THREE.Vector2(PED_D / 2 - 0.01, PED_TOP - 0.01),
  new THREE.Vector2(PED_D / 2 - 0.01, under(-(PED_D / 2 - 0.01))),
  new THREE.Vector2(-PED_D / 2 + 0.01, under(PED_D / 2 - 0.01)),
]);
const wedgeGeo = new THREE.ExtrudeGeometry(wedgeShape, { depth: PED_W - 0.02, bevelEnabled: false });
wedgeGeo.rotateY(Math.PI / 2);
wedgeGeo.translate(-(PED_W - 0.02) / 2, 0, 0);
model.add(shadowed(new THREE.Mesh(wedgeGeo, black)));

const top = new THREE.Group();
top.position.y = SLAB_Y;
top.rotation.x = TILT; // front (+z) dips toward the viewer
model.add(top);

top.add(shadowed(new THREE.Mesh(new RoundedBoxGeometry(SLAB_W, SLAB_T, SLAB_D, 3, 0.01), black)));

const face = new THREE.Mesh(
  texturedPlane(SLAB_W - 0.02, SLAB_D - 0.02, [192, 88], [741, 88], [82, 326], [843, 326]), topMat);
face.geometry.rotateX(-Math.PI / 2);
face.position.y = SLAB_T / 2 + 0.001;
top.add(face);

const LIP_R = 0.035;
const lipGeo = new THREE.CapsuleGeometry(LIP_R, SLAB_W + 0.02 - 2 * LIP_R, 8, 24);
lipGeo.rotateZ(Math.PI / 2);
const lip = shadowed(new THREE.Mesh(lipGeo, black));
lip.position.set(0, -0.005, SLAB_D / 2);
top.add(lip);

document.getElementById('msg').remove();

function render() {
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(render);

window.lectern = { camera, model };
</script>
</body>
</html>
```

- [ ] **Step 2: Open and check**

Use DevTools MCP: `new_page` with `file:///Users/arturkondas/Desktop/git/tata/tablica/index.html`; camera is unpositioned in this task, so run `evaluate_script` `() => { const {camera}=window.lectern; camera.position.set(1.6,1.4,2.6); camera.lookAt(0,0.7,0); return true; }` then `take_screenshot`.
Expected: lectern visible — black pedestal with brass MW badge on front, tilted top with header strip, text lines, QR and photo; rounded lip on front edge; shadow on ground. `list_console_messages` shows no errors. If the top artwork appears upside-down or mirrored, flip by swapping the `tl/tr` and `bl/br` arguments to `texturedPlane`.

---

### Task 3: Controls, reset, idle rotation, resize, offline message

**Files:**
- Modify: `index.html` (module script, after `top.add(lip);` through the end of the script)

**Interfaces:**
- Consumes: `camera`, `model`, `renderer` from Task 2.
- Produces: `window.lectern = { camera, controls, model, reset }`.

- [ ] **Step 1: Replace the tail of the module script**

Replace everything from `document.getElementById('msg').remove();` to the end of the module script with:

```js
// --- controls ---
const HOME = { pos: new THREE.Vector3(1.6, 1.4, 2.6), target: new THREE.Vector3(0, 0.7, 0), up: new THREE.Vector3(0, 1, 0) };
const controls = new TrackballControls(camera, renderer.domElement);
controls.rotateSpeed = 3;
controls.zoomSpeed = 1.2;
controls.panSpeed = 0.8;
controls.dynamicDampingFactor = 0.12;
controls.minDistance = 0.8;
controls.maxDistance = 8;

let idle = true;
controls.addEventListener('start', () => { idle = false; });

function reset() {
  camera.position.copy(HOME.pos);
  camera.up.copy(HOME.up);
  controls.target.copy(HOME.target);
  model.rotation.set(0, 0, 0);
  camera.lookAt(HOME.target);
  controls.update();
}
reset();
document.getElementById('reset').addEventListener('click', reset);

addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
  controls.handleResize();
});

document.getElementById('msg').remove();

const clock = new THREE.Clock();
renderer.setAnimationLoop(() => {
  const dt = clock.getDelta();
  if (idle) model.rotation.y += dt * 0.25;
  controls.update();
  renderer.render(scene, camera);
});

window.lectern = { camera, controls, model, reset };
```

- [ ] **Step 2: Verify from `file://` (Review Focus 1)**

`navigate_page` to `file:///Users/arturkondas/Desktop/git/tata/tablica/index.html`, wait 2 s, `take_screenshot`.
Expected: textured lectern, slowly rotating; `#msg` gone; `list_console_messages` has no errors (in particular no "tainted canvas" / CORS errors).

- [ ] **Step 3: Verify free rotation, pan, zoom, and reset (Review Focus 2, 4)**

`evaluate_script`:
```js
() => {
  const { camera, controls, reset } = window.lectern;
  const r = (v) => v.toArray().map((n) => +n.toFixed(3)).join(',');
  camera.position.set(0, -2, 0.01); camera.up.set(1, 0, 0); controls.target.set(0.3, 0.2, 0); controls.update();
  const moved = r(camera.position);
  camera.position.set(0, 0.7, 0.01); controls.update(); // try to zoom inside model
  const minOk = camera.position.distanceTo(controls.target) >= 0.79;
  reset();
  return { moved, minOk, pos: r(camera.position), up: r(camera.up), target: r(controls.target) };
}
```
Expected: `minOk: true`, `pos: "1.6,1.4,2.6"`, `up: "0,1,0"`, `target: "0,0.7,0"`.
Then manually: `drag` on the canvas → screenshot shows the model viewed from a new angle (including from below/upside-down); `click` `#reset` → screenshot matches the initial framing.

- [ ] **Step 4: Verify resize (Review Focus 3)**

`resize_page` to 390×844, `take_screenshot`; then `evaluate_script` `() => [document.documentElement.scrollWidth <= innerWidth, document.querySelector('canvas').width > 0]`.
Expected: `[true, true]`; model visible, hint and button not overlapping off-screen.

- [ ] **Step 5: Verify offline message (Review Focus 5)**

`evaluate_script` `() => !!document.getElementById('msg')` after load → Expected: `false` (removed only after the scene is built, so if the CDN fails the "Loading… requires internet" message stays visible).

- [ ] **Step 6: Final screenshots**

Screenshots from front, side, top, and below (via `drag`), confirm no console errors. Report them to the user.
