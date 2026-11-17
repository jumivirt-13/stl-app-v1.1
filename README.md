# STL Export 1.1

ArcGIS Experience Builder custom widget (SceneView) that exports the current map
as a 3D-printable STL, including terrain, buildings/layers, labels, raster
contour bands, and disaster-related overlays.

## Live demo

https://jumivirt-13.github.io/stl-app-v1.1/

The deployed static app is served from the repository root (`index.html`,
`web.config`, `service-worker.js`, `cdn/3/`). The widget source remains at the
top level (`src/`, `config.json`, `manifest.json`, `icon.svg`) for installing
into Experience Builder.

## Features

- Terrain draped meshes with high-resolution adaptive refinement.
- Optional flat base plate mode.
- Disaster polygon classes with per-class heights and a rounded (curved) edge
  profile instead of vertical retaining walls.
- Disaster polylines rendered as solid half-pipe walls (rounded top) so they are
  visually distinct from administrative boundaries.
- Administrative boundaries rendered as dash-dot tactile walls; the dots are
  standing cylinders (circular from above).
- Braille labels, tone dots, map furniture (title, scale bar, north arrow).
- Plate layout presets (A4/A3, custom size, scale denominator).

## Installation

Copy the `stl-export-1.1` folder (this repository) into:

```
client/your-extensions/widgets/stl-export-1.1
```

The widget manifest name is `stl-export-1.1` and the label is `STL Export 1.1`.
Build the client with the Experience Builder dev build (webpack watch); the
widget bundles to `client/dist/widgets/stl-export-1.1/dist/`.

## Settings overview

- Terrain: include terrain, extrude base, flat base plate.
- Disaster polygon: layer id, class field, per-class heights, edge curve width.
- Disaster polyline: layer id, wall height, wall half-width, class heights.
- Admin levels: per-level height, half-width, dash/dot length, gap.
- Labels, raster bands, map components, plate layout.
