# Lectern 3D Viewer — Design

## Goal
Static HTML page showing a 3D model of the black information lectern (sloped top panel + pedestal with brass "MW" anchor badge) from `~/Downloads/tata_2.jpg` (originally `Untitled.jpg`). Audience: potential investor. Visual polish matters; dimensions are approximate.

## Constraints
- Static files only, no build step, no server required (opens from disk or any static host).
- Three.js loaded from CDN (jsdelivr) via import map.
- Model freely movable on all axes: rotate (no locked up-axis), pan, zoom.

## Files
- `index.html` — page, styles, and scene script inline.
- `textures/panel.jpg` — top face artwork, perspective-corrected from the source image into a flat rectangle.
- `textures/pedestal.jpg` — pedestal front face including the brass badge, cropped from the source image.

## Geometry (meters, approximate)
- Pedestal: box 0.80 W × 0.90 H × 0.50 D, small corner bevel.
- Top panel: extruded rounded-rectangle slab 1.60 W × 0.60 D × 0.04 thick, tilted ~20° toward the viewer, with a cylindrical rolled lip along the front edge.
- Plinth: thin box under the pedestal, slightly larger footprint.

## Materials
- Body: `MeshStandardMaterial`, near-black, roughness ~0.4.
- Top face: panel texture (sRGB).
- Pedestal front: pedestal texture; slightly lower roughness so the brass badge reads as metallic.
- Environment map from `RoomEnvironment` + PMREM for reflections.

## Scene
- Soft gradient background, shadow-receiving ground plane, key/fill/rim lights, soft shadows.
- Slow idle auto-rotation; stops permanently on first user interaction.

## Controls
- `TrackballControls`: left-drag rotate (all axes), right-drag pan, wheel/pinch zoom.
- "Reset view" button restores initial camera position, target, and up vector.
- Canvas resizes with the window.

## Out of scope
Object-move gizmo, annotations/hotspots, AR, exact fabrication dimensions.

## Verification
Open in Chrome via DevTools MCP, screenshot from front / side / top / below, confirm zero console errors and that rotate/pan/zoom/reset work.
