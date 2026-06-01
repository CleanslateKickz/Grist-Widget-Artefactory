# Project: Atlas — Territorial 3D Mockup

## Context

Complete UX/UI redesign of the "Territorial 3D Mockup" Grist widget,
from the Claude Design canvas (`carte-grist`, **Atlas** direction).

Target: elected officials / public presentations, but task #1 remains **building**
the mockup (data import, editing, symbolization, 3D model placement).

Major difference from the old widget: **Mapbox has been replaced by MapLibre GL JS**,
and GLTF 3D models (which relied on Mapbox Standard's native `model` layer) are now
rendered via a **three.js custom layer**.

## File Architecture

```
Atlas/
├── index.html   # Atlas Chrome (HTML/CSS) + DOM structure
├── app.js       # All logic (ES module)
└── CLAUDE.md
```

- `index.html`: **Atlas** design (paper `#F4EFE3`, Newsreader serif + Hanken
  Grotesk, terracotta accent `#C44536`). Header + icon rail (6 modules) +
  contextual module panel (left) + map + inspector (right) +
  map overlays (compass, HUD, legend, solar band, selection bar).
- `app.js` (module, imports `three` + `GLTFLoader` via importmap): state,
  MapLibre map, 3D custom layer, modules, symbolization, selection, OSM/file
  import, Grist persistence, project save, command palette.

## Technical Decisions (validated with user)

1. **Direction = Atlas** (among Atlas / Workbench / Compass from the design canvas).
2. **Basemaps**: OpenFreeMap (Liberty 3D / Bright / Positron, no key;
   3D buildings via `fill-extrusion`) **+ IGN Géoplateforme** (Plan IGN, Ortho IGN
   — raster, FR territorial target). IGN basemaps are pure raster: `ignRasterStyle`
   re-injects the OpenFreeMap vector source (`openmaptiles`) + the `building-3d`
   layer to keep 3D buildings and the same Z alignment as Liberty.
   `setBasemap` cuts the terrain (`setTerrain(null)`) before `setStyle` then
   re-applies it in `onStyleReady` (otherwise the terrain depth pass fails on the
   removed DEM source — "shaderPreludeCode").
3. **3D models = three.js custom layer in InstancedMesh** (engine inspired by
   EclExt): MapLibre has no native `model` layer. See `Models3D`.
4. **MapLibre GL JS v5** (`maplibre-gl@5`) for the **globe projection**
   (Google Earth style, auto-switches to mercator at zoom). `applyProjection` /
   `STATE.settings.projection` ('globe' | 'mercator'), toggled in the View module.
   The 3D custom layer reads `args.defaultProjectionData.mainMatrix` (v5 render
   signature); models are correct in mercator (useful zoom), at globe very zoomed
   out they may be slightly offset (tiny objects).

## 3D Engine (`Models3D`) — adapted from EclExt

- **InstancedMesh**: 1 `InstancedMesh` per GLTF sub-mesh × model group
  (instead of one clone per object) → thousands of objects feasible.
  Cap `MAX_3D_INSTANCES = 20000`.
- **Local scene origin** (`setOrigin`/`localMeters`): objects expressed in
  local meters, a single origin matrix per frame (`_m4Origin`) → precision +
  constant instance matrices.
- **Terrain placement**: `elevAt` (cache) queries `queryTerrainElevation`
  per object; auto re-sync when DEM tiles arrive (drift in `render` →
  `recomputeAll`).
- **Fast-path editing** (`updateEdited`): inspector sliders only
  recompute matrices for affected objects (no rebuild/reload GLTF).
- **Viewport culling + zoom gate** (`collect`, `cull` at `moveend`):
  only instantiates objects within the extent; below `MODEL3D_ZOOM_GATE` and
  beyond `MODEL3D_GATE_COUNT`, 3D is hidden.
- Materials forced to `DoubleSide` (the origin matrix has a negative mercator
  Y → would otherwise cull front faces of arbitrary GLTFs).

## Relief & Ambiance

- **Terrain sources** (`TERRAIN_SOURCES`): global terrarium (no key) **or
  IGN LIDAR HD** (France) decoded GeoTIFF Float32 → TerrainRGB via the
  `ignmnt://` protocol in a **Web Worker pool** (created on first use).
- **Lighting**: `computeAmbient` (day/dusk/night) + `computeMoon`
  (lunar illumination SunCalc) drive `map.setLight`, the sky `setSky`, and
  three.js lights (`Models3D.setSun`).

## Conventions

- No framework (vanilla JS + three.js). UI handlers exposed via the global
  object `window.A` (inline `onclick` calls `A.xxx(...)`).
- English for UI and comments.
- Atlas color tokens as CSS variables (`:root`).
- Optional Grist persistence: `Maquette_Layers` table (Name, Color, Visible,
  GeomType, StyleJSON, GeoJSON). The widget also works in **standalone** mode
  (without Grist) with autosave `localStorage` + JSON export/import.

## Current Status — Working

- MapLibre map + OpenFreeMap, 3D buildings, terrain (AWS terrarium DEM, free),
  sky/atmosphere, solar lighting SunCalc (realistic direction).
- Modules: **Location** (Nominatim search, geoloc, coords, radius, project name),
  **Layers** (list, visibility, deletion), **Symbolize** (fixed/categorized/graduated
  color, size, 3D model, label — live preview), **3D Models** (GLTF library),
  **Sun** (presets, hour, date, shadows), **View & render** (pitch/bearing, basemap,
  buildings/terrain/labels/sky).
- **OSM** import (Overpass, presets) and **GeoJSON file** (drag & drop).
- **Object selection** on map (click, shift-click, box-select), selection bar with
  prev/next navigation, edit inspector (single + relative batch).
- **Command palette** `⌘K`.
- Project save/load JSON, GeoJSON export, autosave.

## Points of Attention / Known Limitations

- **Ground shadows**: MapLibre does not project ground shadows like Mapbox Standard.
  The "Shadows" toggle modulates three.js model lighting (no ground shadow map).
  The light direction does follow the solar position.
- **3D perf**: InstancedMesh rendering (see "3D Engine" section), cap
  `MAX_3D_INSTANCES = 20000`, viewport culling + zoom gate.
- **Rotation semantics**: `rotationZ` = azimuth (yaw, Euler order `YXZ`) — the primary
  control. `rotationX`/`rotationY` (pitch/roll) remain approximate.
- **IGN DEM**: `ignmnt://` requires `OffscreenCanvas` (Chrome/Edge/Firefox,
  Safari ≥ 16.4); flat tile fallback if decoding fails. France coverage only.
- **GLB models**: procedural catalog generated in the repo by
  `scripts/generate-models.js` (`npm run models`) → `published/atlas/models/<set>/*.glb`
  (+ `catalog.json`). **37 models × 2 sets** (`colored` / `mono`), low-poly,
  modeled in meters (scale 1), base on ground. Co-located with the published widget
  (`published/atlas/models/`); `MODEL_LIBRARY.baseRoot` is resolved **relatively**
  to the widget (`new URL('./models/', import.meta.url)`) → portable, no hardcoded
  URL/repo (served from `…/atlas/models/` in prod). Set chosen in the Models module
  (`A.setModelSet`). In dev, `probeLocalModels()` finds `published/atlas/models/`
  if the repo root is served; otherwise falls back to hit circles.
  To add an object: edit the `CATALOG` in the generator and re-run `npm run models`.
- **3D model interaction**: three.js objects are not queryable by
  `queryRenderedFeatures`. We therefore add a MapLibre circles layer (low opacity)
  serving as click zone / highlight.
- **Tests**: not tested in a real browser in the dev environment (no headless
  browser). To validate: open in Grist or a browser for WebGL rendering and GLTF
  loading (`nic01asfr.github.io/3D-Models/`).

## Publication

Not published. To publish: `published/atlas/` with `package.json` (`grist` section) +
copy of `index.html`/`app.js`, then `npm run manifest`.
`requiredAccess: 'full'` is required (creates `Maquette_Layers` table).
