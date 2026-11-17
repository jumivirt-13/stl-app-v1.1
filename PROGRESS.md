# Progress and next agenda

Updated: 2026-11-17

## Current state (v1.1)

- Widget: `client/your-extensions/widgets/stl-export-1.1` (source mirrored in
  `src/`, `manifest.json`, `config.json`, `icon.svg`).
- Known-good backup: `C:\arcgis-experience-builder\_backups\stl-export-1.1_known-good_20261117-0759`.
- Repo: https://github.com/jumivirt-13/stl-app-v1.1 (`main`)
- Live app: https://jumivirt-13.github.io/stl-app-v1.1/
- Deployed build: Experience Builder static export under `cdn/3/`;
  `index.html` uses `<base href="./cdn/3/"/>`, `isOutOfExb: true`,
  `appFolderName: "."`, `mountPath: "/"`. GitHub Pages serves `main` root.
- Last commits: `0cf593b` (full README), `955d5e8` (demo URL), `0ae16ad`
  (static build `cdn/3`), `b2ac56e` (widget source).
- TypeScript baseline diagnostics (pre-existing, not blockers):
  `geometryEngine` default export, `projectOperator`, `number[]` vs
  `Uint*Array`, `Property 'X' does not exist on type '{}'` in settings.

### Shipped in 1.1

- Disaster polylines: solid half-pipe walls (10-segment rounded top), densified
  stations, embedded flat base; dash/dot options removed.
- Disaster polygons: rounded rims via `disasterPolygonEdgeCurve` (cosine
  flare + bottom annulus); slabs draped top and bottom; densified boundary.
- Conforming edge subdivision replaces centroid fan (shared midpoints, up to
  240k triangles) for smoother relief.
- Exact terrain sampling: triangle bucket index + barycentric interpolation,
  vertical-face filtering, clamped queries, IDW fallback, sample counters.
- Admin dash-dot dots: 16-segment standing cylinders with terrain-following
  bottom.
- UI: edge-curve setting in runtime + builder, disabled single height when a
  class field is set, solid-line note, `stl-export-1.1-*.stl` filenames.
- Deployment: static app exported via `build:for-download` and published to
  Pages (`cdn/3/`).

### Rebuild / redeploy checklist

1. Edit widget under `client/your-extensions/widgets/stl-export-1.1`.
2. Dev watcher: `client/node_modules/.bin/webpack --mode development --watch`.
3. Production export: `npm run build:for-download` in `client`.
4. Download from dev server: `GET https://localhost:3001/download/1?token=x`
   then poll `GET /download/1/status?uuid=...` (`005` = done) and fetch
   `GET /download/1/getZip?uuid=...&token=x`.
5. Replace `index.html`, `web.config`, `service-worker.js`, and `cdn/<n>/` in
   this repo; update `<base href>` if the build number changes.
6. Commit/push `main`; Pages rebuilds automatically.

---

## Next agenda 1 - Smooth terrain blending for disaster polygons

Reference: https://zrnnn.github.io/topomapper/ (smooth, terrain-hugging
transitions instead of hard steps).

### Goal

Make raised disaster polygon surfaces flow into the surrounding terrain with a
soft shoulder/fillet, so class boundaries and the polygon-to-ground junction
look sculpted rather than cut. Keep the model watertight and 3D-printable.

### Current behavior to improve

`buildDrapedPolygonArea` builds a flat/ground-following top at
`ground + classHeight` and a rim:
- `edgeCurve = 0`: vertical side wall from top to terrain.
- `edgeCurve > 0`: cosine flare outward down to the ground, closed by a bottom
  annulus. This still forms a distinct outer lip at the terrain contact.

### Proposed design

1. Add an **outer blend apron**: extend beyond the polygon boundary by a
   configurable `blendWidth` (mm on plate), with height easing from the class
   top at the boundary to `0` at the apron edge using a smoothstep/cosine
   falloff; sample terrain along the apron and add a small negative embed at
   the outer edge so it welds into the ground.
2. Add an optional **inner shoulder**: round the top edge inside the polygon
   (small radius) so the plateau eases into the blend instead of a sharp top
   lip.
3. Make the blend **slope-aware**: widen on steep slopes, clamp by the
   distance to neighboring class polygons to avoid interpenetration. Evaluate
   polygon offsetting or a per-vertex distance limit (half the nearest edge
   distance between classes) rather than full boolean ops.
4. Keep watertightness: share apron vertices between adjacent classes where
   possible, always drop the outer ring below the terrain, and avoid
   coplanar surfaces (z-fighting does not matter for STL but self-intersection
   does).
5. New config fields (working names):
   - `disasterPolygonBlendWidth` (mm, default 0 = current behavior)
   - `disasterPolygonBlendProfile` (`cosine` / `smoothstep`)
   - `disasterPolygonTopRound` (mm, optional inner shoulder)
   - possibly `disasterPolygonBlendSlopeScale` (widen on slopes)
6. Reuse the existing `edgeCurve` or supersede it with the blend system; keep
   backward compatibility with old app configs.

### Open questions / research

- Inspect topomapper's geometry approach (contour smoothing, fillets, or
  shader-only smoothing). Shader blending is not printable; the mesh must
  carry the blend.
- Decide whether blending applies only outside the polygon or also flattens
  the interior plateau edge.
- Overlap policy for adjacent classes (higher class wins, or blend both and
  let union handle it).

### Acceptance criteria

- No visible seam/step at the polygon-terrain contact; smooth shoulder across
  slopes.
- Watertight, manifold mesh; no floating or inverted faces.
- Configurable blend width; `0` reproduces v1.1 output.
- Tested on flat, steep, and multi-class adjacent polygons.

---

## Next agenda 2 - Additional export formats (3MF, OBJ, ...)

### Goal

Keep STL as the default but let users download 3MF (preferred for modern
slicers) and OBJ, with an extensible writer layer for future formats (PLY,
GLB).

### Proposed design

1. Define a format-neutral mesh payload (`positions`, `indices`, optional
   normals, per-object names) or keep using `esri/geometry/Mesh` plus a small
   writer interface:
   `writeStl`, `writeObj`, `write3mf`, `writePly`.
2. OBJ writer (`writeObj.ts`):
   - Text output: `v`, optional `vn`, `f` with 1-based indices.
   - Add `# units: meters` and object/group headers.
   - Multiple objects: refactor the merge step to keep per-layer meshes (or
     emit groups) so OBJ can preserve layer structure.
3. 3MF writer (`write3mf.ts`):
   - 3MF is an OPC ZIP package: `[Content_Types].xml`, `_rels/.rels`,
     `3D/3dmodel.model` (XML), optional `Metadata/model_settings.config`.
   - Use `unit="millimeter"` and scale vertices from meters to mm (x1000).
   - ZIP options: use `jszip` (already a client dependency) or implement a
     minimal store-only ZIP + CRC32 writer to keep the widget bundle small.
     Prefer a tiny internal writer if bundle size matters.
   - Validate in PrusaSlicer/Bambu Studio/Windows 3D Viewer.
4. UI (Part 5):
   - Format selector: STL (binary), 3MF, OBJ.
   - Show file size estimate/extension/MIME; disable formats that are not
     implemented.
   - Keep the WebGL preview driven by the mesh, independent of the format.
5. Download naming: `stl-export-1.1-<timestamp>.<ext>`.
6. Optional later: GLB (binary glTF) and PLY; 3MF with multiple objects and
   per-object colors (base materials).
7. Tests: round-trip import into Blender/MeshLab; slicer opens 3MF at 1:1
   scale; triangle counts match the STL; meshes remain manifold.

### Notes

- 3MF expects millimeter units; the widget works in meters, so scale explicitly
  at write time and do not change the internal geometry.
- OBJ has no units; document meters in a comment.
- Keep binary STL output byte-identical for regression checks.

---

## Suggested order

1. Export format abstraction + OBJ + 3MF (lower risk, independent of geometry
   changes).
2. Terrain blending research + prototype on a single class polygon.
3. Slope/overlap handling and adjacent-class tests.
4. Expose settings, rebuild, export static app, update README and Pages.
