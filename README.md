# STL Export 1.1 - 3D Printable Map widget

An ArcGIS Experience Builder **custom widget** (SceneView) that converts the
area inside a user-drawn rectangle into a **binary STL file** for tactile 3D
printing: raised terrain, Braille labels, administrative boundary walls,
hazard raster bands, disaster vector layers, roads/rivers, and map-margin
furniture.

Live app: **https://jumivirt-13.github.io/stl-app-v1.1/**

---

## What it does

- Draw a rectangle on a 3D scene; the widget builds a printable solid model of
  everything inside it.
- **Terrain** - real DEM surface from the scene ground (falling back to World
  Terrain3D), optionally extruded to a watertight base or replaced by a flat
  base plate.
- **Scene layers** - buildings and other 3D meshes, optionally simplified,
  clipped to the AOI, lifted onto the terrain, with foundation pads under
  floating buildings.
- **Administrative boundaries** - village/district/city/province/country walls
  as a tactile dash-dot code, following the terrain surface. The dots are now
  standing cylinders that read as circles from above.
- **Braille labels** - admin names as raised 6-dot Braille on the map, plus
  furniture labels (title, scale bar, print scale) in the margin.
- **Disaster data** - polygons with per-class heights and rounded rims,
  polyline walls as solid rounded half-pipes, point domes, and raster contour
  bands with severity "texture" dots.
- **Roads & rivers** - raised wall lines and filled polygon areas.
- **Margin furniture** - title, alternating-block scale bar, print scale, and a
  3D north arrow.
- **Output** - binary STL download plus an in-widget WebGL 3D preview with an
  orbit camera.

---

## What's new in 1.1

### Disaster polylines: solid half-pipe profile

- Disaster polylines are no longer square dash/dot walls. Each path is built as
  a **solid half-pipe**: a 10-segment semicircular cross-section with a flat
  base sitting on the terrain and a rounded top rising by the wall height.
- The path is densified (`densifyPath`) so the base follows the ground at
  roughly one station per terrain cell (`max(2 * halfWidth, terrain cell)`);
  both ends are capped and the base is embedded 0.1 m.
- The dash/dot line-style settings are gone (`disasterWallStyle`,
  `disasterWallDashLength`, `disasterWallDotLength`, `disasterWallGap`). The
  rounded profile is what distinguishes disaster lines from the square
  dash-dot administrative boundaries.

### Rounded rims on disaster polygons

- New config option `disasterPolygonEdgeCurve` - "Edge curve width (mm)",
  default `1.0`, `0` restores a vertical wall.
- With a positive value, the edge flares outward along a cosine profile that
  starts horizontal at the class top and meets the ground flat (6 levels),
  closed by a bottom annulus. Adjacent classes read as rounded steps instead of
  sharp retaining walls.
- Per-class heights still come from the class attribute field. When a class
  field is set, the single "Polygon height" input is disabled in both the
  runtime panel and the builder settings, because the class heights take over.

### Fully draped polygon slabs

- Both surfaces of an area slab now follow the DEM: the top is
  `ground + class height` and the bottom is `ground - 0.1 m embed` (previously
  the bottom was a flat plane under the whole polygon). Slabs therefore hug
  slopes with no floating gaps.
- The boundary ring is densified (`densifyRing`) to about the terrain cell
  size (capped at 1500 vertices), so top, bottom, and side walls track the
  relief along the polygon edge instead of cutting straight across valleys.
- Interior refinement is now **conforming edge subdivision**: long triangle
  edges are split with shared midpoints (up to 9 iterations / 240k triangles)
  instead of centroid fans, which removes the stretched "straight across the
  valley" look and keeps the mesh watertight.

### Exact terrain interpolation

- `TerrainSampler` now builds a bucket grid over the terrain **triangles** and
  returns barycentric heights for points inside a triangle, instead of
  always blending nearby vertices with inverse distance.
- Vertical side faces are excluded by checking the face normal
  (`|nz| / |n| > 0.2`), so retaining walls or skirts in the terrain mesh do not
  pollute surface queries.
- Queries are clamped slightly inside the mesh bounds; points outside the mesh
  fall back to the inverse-distance lookup. The sampler counts
  `exactSamples` / `fallbackSamples` / `missSamples`, logged after each draped
  polygon.

### Administrative dash-dot dots as cylinders

- `buildStandingTube` replaces the rectangular dot blocks: a 16-segment
  cylinder with a flat circular top, radius `max(halfWidth, dotLength / 2)`.
- The bottom follows the terrain at every rim vertex and the top is lifted by
  the level height, so the footprint stays circular on slopes.

### UI, naming, and diagnostics

- Runtime wizard and builder settings both expose "Edge curve width (mm)";
  runtime shows a note that disaster lines are solid rounded half-pipes.
- Download filenames are now `stl-export-1.1-<timestamp>.stl`; the widget logs
  a mount banner and export mode (`includeTerrain`, `flatBase`, `baseZ0`,
  `extrudeBase`) with a warning if a flat base and terrain are both enabled.
- Manifest renamed to `stl-export-1.1` (label "STL Export 1.1").
- Terrain diagnostics now report face count, maximum Z span, and how many tall
  faces are vertical side walls.

### Deployment

- The production static app exported by Experience Builder (`dist-download`) is
  published from this repository root on GitHub Pages: `index.html`,
  `web.config`, `service-worker.js`, and the build under `cdn/3/`.

---

## Repository layout

```
.
|-- index.html                  # deployed static app entry (base href ./cdn/3/)
|-- web.config                  # IIS rewrite/compression rules for the app
|-- service-worker.js           # app shell/service worker
|-- cdn/3/                      # production Experience Builder app build
|   |-- widgets/stl-export-1.1/
|   `-- ...
|-- config.json                 # default widget instance config
|-- icon.svg                    # widget icon
|-- manifest.json               # widget manifest (name stl-export-1.1)
|-- src/                        # widget source (install this folder)
|-- README.md
```

The built app is only for the live demo. To use the widget in Experience
Builder, copy the repository (or at least `src/`, `manifest.json`,
`config.json`, `icon.svg`) into the widgets folder as shown under
[Installation](#installation).

---

## Source layout

```
src/
|-- config.ts                   # config model, defaults, admin levels, plate presets
|-- runtime/
|   `-- widget.tsx              # Part 1-5 wizard UI, AOI drawing, runExport, preview
|       `-- lib/
|           |-- exportScene.ts  # main geometry/exporter (~3965 lines)
|           |-- writeStl.ts     # binary STL serialization + browser download
|           `-- preview3d.ts    # dependency-free WebGL preview renderer
`-- setting/
    `-- setting.tsx             # builder settings panel (mirrors the config model)
```

| File | Size | Lines | Description |
|---|---:|---:|---|
| `src/config.ts` | 11.5 KB | 228 | Config model & defaults: terrain, admin levels, labels, disaster classes, roads/rivers, raster, margins, plate layout |
| `src/runtime/widget.tsx` | 88.5 KB | 1850 | 5-part wizard UI (AOI, layer settings, map components, layout, export), `runExport` orchestration, static layout preview, WebGL preview |
| `src/runtime/lib/exportScene.ts` | 151.8 KB | 3965 | Main algorithm: terrain, exact sampling, clipping, admin dash-dot walls, half-pipe disaster lines, Braille, raster bands, draped slabs, margin furniture, STL mesh assembly |
| `src/runtime/lib/writeStl.ts` | 3.3 KB | 100 | Binary STL serialization + browser download |
| `src/runtime/lib/preview3d.ts` | 13.7 KB | 372 | Dependency-free WebGL2/1 preview renderer (hand-written shaders, orbit camera) |
| `src/setting/setting.tsx` | 35.5 KB | 792 | Builder settings panel (mirrors the config model) |
| `manifest.json` | 0.5 KB | - | Widget manifest (name `stl-export-1.1`, label `STL Export 1.1`) |

---

## Architecture & data flow

```
widget.tsx (Part 1-5 wizard)
  -> exportScene(view, rectangle, config) -> ExportResult
       createTerrainMesh
       TerrainSampler (triangle index + barycentric lookup)
       querySceneLayerMeshes -> liftBuildingsToTerrain
       buildDivisionBoundaryWalls -> buildDashDotWallFromRing -> buildDashDotBlock + buildStandingTube
       buildHalfPipeFromPaths (disaster polylines; densifyPath)
       buildDrapedPolygonArea (disaster polygons; densifyRing + conforming subdivision + edgeCurve)
       buildRasterBands (hazard contour bands + dots)
       buildDisasterVectorLayers (polygons, polylines, points; per-class heights)
       buildLineFeatureWalls (roads/rivers; draped walls and polygon areas)
       buildMarginFrame -> title / scale bar / print scale / north arrow
       extrudeTerrain / buildFlatBase
       merge -> convertVertexSpace
  -> writeStl.meshToStl -> binary STL Blob -> download
  -> extractPreviewGeometry -> preview3d.Preview3D (WebGL)
```

The exporter is pure geometry code on the ArcGIS JS API
(`esri/geometry/Mesh`, `meshUtils.createFromElevation`, `meshUtils.merge`,
`MeshLocalVertexSpace`); no 3D engine is used to generate geometry. The only
WebGL code is the optional preview.

---

## Key algorithms

| Area | Function(s) | What it does |
|---|---|---|
| AOI drawing | `buildRectangleFromScreen` (widget.tsx) | Fixes tilted-camera foreshortening: anchors on the screen-centre ground point, measures ground m/pixel, rebuilds the AOI with the drawn on-screen aspect ratio |
| Terrain mesh | `createTerrainMesh` | `meshUtils.createFromElevation` from the scene ground sampler; falls back to the online World Terrain3D ElevationLayer |
| Terrain indexing | `TerrainSampler.buildTriangleIndex` | Bucket grid over terrain triangles; marks horizontal/sloped surface faces and skips vertical side walls |
| Terrain lookup | `TerrainSampler.exactHeightAt` / `heightAt` | Barycentric height inside the containing triangle; clamped query point; inverse-distance fallback; exact/fallback/miss counters |
| Terrain fallback | `terrainHeightAtLinear` | Linear scan of terrain vertices when no grid index can be built |
| Simplify | `simplifyMeshByClustering` | Vertex clustering to a 3D grid; each cell -> centroid; drops degenerate triangles |
| Clipping | `clipMeshToRectangle` | Sutherland-Hodgman half-space clip of each triangle; cap faces follow the roof profile |
| Footprints | `buildingFootprintRings` | Rasterizes triangles into an occupancy grid and traces the boundary -> closed rings |
| Foundation pads | `addFoundationPads` | Adds pads under floating buildings so they connect to the ground |
| Admin walls | `buildDivisionBoundaryWalls`, `buildDashDotWallFromRing`, `buildDashDotBlock` | Per-level dash + N-dot pattern; blocks extruded along the terrain normal so side faces stand 90 deg to slopes |
| Admin dots | `buildStandingTube` | 16-segment standing cylinder with terrain-following bottom rim and flat circular top |
| Braille | `buildBrailleLabelMesh`, `brailleCellsForText`, `buildDomeMesh` | Standard 6-dot table (digits use the 3456 prefix); each dot is a rounded dome; labels avoid walls and the AOI edge |
| Ring densify | `densifyRing`, `densifyPath` | Inserts intermediate vertices so no segment exceeds a spacing; ring version raises spacing to respect a vertex cap |
| Draped polygons | `buildDrapedPolygonArea` | Ear-clipped ring, densified boundary, conforming edge refinement, top = ground + height, bottom = ground - embed |
| Rounded rims | `buildDrapedPolygonArea` (edgeCurve branch) | Cosine-flared skirt from class top to ground plus a bottom annulus, offset outward by the edge curve width |
| Half-pipe lines | `buildHalfPipeFromPaths` | Solid semicircular profile (10 segments) along densified paths, flat base embedded in the terrain, end caps |
| Road/river walls | `buildDrapedLineWallsFromPaths`, `buildLineWallFromPaths` | Wall top/bottom follow the terrain profile; bottom embedded below the surface |
| Raster bands | `buildRasterBands` | Samples raster on a grid; normalizes and quantizes into contour bands; slabs follow terrain; higher bands get larger texture dots |
| Disaster vectors | `buildDisasterVectorLayers` | Polygon/polyline/point layers, recursing sublayers; per-class heights from an attribute field; polygons use `edgeCurve`, polylines use half-pipes |
| Margin furniture | `buildMarginFrame`, `buildMarginTitle`, `buildMarginScaleBar`, `buildMarginPrintScale`, `buildMarginNorthArrow` | Flat offset frame plus raised Braille/3D furniture |
| Layout preview | `computeLayoutPreviewData` | 2D division outlines + Braille label positions for the static layout preview |
| STL | `meshToStl` (writeStl.ts) | Binary STL: 80-byte header, triangle count, per-triangle normal + 3 vertices + attribute; filters non-finite/huge vertices |
| Preview | `Preview3D` (preview3d.ts) | WebGL2/1 renderer: hand-written shaders, Lambert lighting + edge darkening, orbit camera, smooth per-vertex normals |

---

## Unit conventions

All user-facing "mm on plate" values (wall heights, half-widths, Braille font
sizes, edge curve width, etc.) are converted to ground meters by `mmToMeters`
at the print scale:

```
meters = mm * effectiveScaleDenom / 1000
```

`effectiveScaleDenom` uses `layoutFitToPlate` (plate map frame vs AOI extent)
when enabled, otherwise `layoutScaleDenom` (default 1:5000). This keeps the
printed model at the configured scale.

---

## Settings overview

### Terrain and base

| Setting | Default | Notes |
|---|---:|---|
| Include terrain | on | DEM surface from the scene ground / World Terrain3D |
| Extrude base | off | Extrude the terrain to a watertight base |
| Extrusion depth | - | Depth below the lowest terrain point |
| Flat base plate | off | Flat plate spanning the AOI (no terrain needed) |
| Flat base thickness | - | Plate thickness below the surface |

### Layers, labels, and simplify

| Setting | Default | Notes |
|---|---:|---|
| Include 3D layers | on | Buildings and other meshes |
| Simplify meshes | - | Vertex clustering to shrink STL size |
| Simplify cell | - | Cell size in mm on plate |
| Admin boundary walls | on | Village/district/city/province/country |
| Boundary wall height | 0.6 mm | Per level |
| Boundary wall half-width | 0.3 mm | Per level |
| Dash / dot / gap | 1.8 / 0.6 / 0.4 mm | Per level dash-dot code |
| Braille labels | on | Layer labels and furniture labels |
| Label dome height / font size / edge gap | - | Font size sets dot pitch, radius (32%) and cell spacing (80%) |
| Furniture labels | on | Title, scale bar, print scale |

### Disaster layers

| Setting | Default | Notes |
|---|---:|---|
| Hazard raster | on | Contour bands with texture dots |
| Polygon layer id | none | Empty/none excludes the layer |
| Polygon height | 0.6 mm | Disabled when a class field is set |
| **Edge curve width** | **1.0 mm** | New in 1.1; `0` = vertical wall |
| Polygon class field / heights | empty | Per-class height overrides |
| Polyline layer id | none | Rendered as solid half-pipes |
| Wall height / half-width | 0.6 / 0.1 mm | Per class optional |
| Point layer id | none | Raised domes |
| Point radius / height | 0.3 / 0.5 mm | Per class optional |
| Raster bands / grid / max height | 4 / 24 / 1.6 mm | Band heights can be overridden per band |
| Hazard dots / per cell / radius / height | on / - / - / - | Severity texture on the bands |

### Roads, rivers, and layout

| Setting | Default | Notes |
|---|---:|---|
| Roads / rivers | on | Polyline walls and/or polygon areas |
| Wall heights / half-widths | 0.6 / 0.1 mm | Per category |
| Polygon heights | 0.6 mm | Per category |
| Use layout mode | off | Plate size + scale drive margins and furniture |
| Plate preset | A4-L | A4-L / A3-L / A2-L / CUSTOM |
| Custom plate width / height | 400 / 300 mm | Used when preset is CUSTOM |
| Print scale | 1:5000 | Or auto-fit to plate (`layoutFitToPlate`) |
| Margins (T/L/R/B) | 25 / 10 / 10 / 10 mm | Furniture placement |
| Title / scale bar / print scale / north arrow | - | Individual switches and dimensions |

---

## Installation

Copy the widget into your Experience Builder client:

```
client/your-extensions/widgets/stl-export-1.1/
|-- manifest.json
|-- config.json
|-- icon.svg
`-- src/...
```

Keep the folder name `stl-export-1.1` (the manifest name is `stl-export-1.1`,
label "STL Export 1.1"). Build the client with the Experience Builder dev build
(webpack watch); the widget bundles to
`client/dist/widgets/stl-export-1.1/dist/`.

To add it to an app, register the extension, drop the widget into a scene app,
draw the AOI, configure the parts in the wizard, and export.

---

## Development notes

- Widget build (watch): `client/node_modules/.bin/webpack --mode development --watch`
  from the client folder.
- Production export for the static app: `npm run build:for-download`
  (builds to `client/dist-download`); the Experience Builder download endpoint
  packages the app and its custom widgets into a zip.
- The deployed app expects `index.html` with `isOutOfExb: true`,
  `appFolderName: "."` and `<base href="./cdn/<buildNumber>/"/>`, plus
  `web.config` and `service-worker.js`.
- GitHub Pages serves this repository from `main` (root). The demo build is
  under `cdn/3/`; a new Experience Builder download may produce `cdn/4/`, in
  which case replace the `cdn/` build and update the `<base href>` in
  `index.html`.
- `service-worker.js` caches the app shell. After deploying a new build, bump
  the deployment/clear site data if the browser keeps serving the old bundle.
- Known pre-existing TypeScript baseline diagnostics in this project:
  `esri/geometry/geometryEngine` default export, `projectOperator` export,
  `number[]` vs `Uint*Array` position types, and `Property 'X' does not exist
  on type '{}'` in the settings panel.
- The live demo is a static Experience Builder export; it does not include the
  builder UI. Use the widget source for authoring.

---

## Version history

- **1.1** - Solid half-pipe disaster lines (dash/dot options removed); rounded
  disaster polygon rims via edge curve width; fully draped polygon slabs with
  conforming edge subdivision; exact barycentric terrain sampling with
  triangle buckets; admin dash-dot dots as standing cylinders; disabled
  single-height input when classes are used; static app deployed to GitHub
  Pages.
- **1.0** - Initial STL export: terrain, scene layers, dash-dot admin walls,
  Braille labels, raster bands, disaster vectors, roads/rivers, margin
  furniture, binary STL + WebGL preview.
