# Project: EclExt — Public Lighting (CNIG)

## Context

Grist widget for management / simulation of **public lighting** compliant with the
**CNIG EclExt v1.1** standard. MapLibre map + IGN basemaps (Géoplateforme),
LIDAR HD terrain relief, 3D rendering of luminaires (light cones + GLB models),
hourly day/night simulation (SunCalc) and night profiles (switch-on, switch-off,
power/flux/temperature variations).

Single standalone file: `index.html` (static widget). Works in Grist
(`requiredAccess: full`, creates the tables) or in **standalone demo mode**.

## Architecture

- MapLibre map, IGN WMTS basemaps (Plan / Ortho / "Dark").
- Terrain: IGN LIDAR HD DEM (WMS GeoTIFF Float32) decoded to TerrainRGB via a
  **`float32dem://` protocol**.
- 3D: `LuminaireCustomLayer` (custom three.js r128 layer) in **InstancedMesh**
  (3 draw calls: cones / ground disks / ULR cones) + individual GLBs
  viewport-culled.
- CNIG data: tables `Modeles_Luminaires`, `ProfilNocturne`,
  `PlageVariation`, `PointLumineux`.
- Modules (toolbar): Data · Profiles · Rendering · Models · Input · Map.

## Applied Optimizations (vs original version)

1. **Terrain off main thread**: GeoTIFF→TerrainRGB decoding in a **Web Worker pool**
   (`makeTerrainDecoderPool`, inline worker as Blob). And **single DEM source**
   shared between terrain + hillshade (before: 2 sources → double decoding).
2. **Time slider without re-clustering**: `onTimeChange` updates 3D colors
   immediately (fast path) and **debounces** 2D `setData` (`scheduleRender2D`,
   90 ms) to avoid re-clustering on every tick.
3. **Pan without terrain = no recalculation**: `onMapMoveEnd` only calls
   `updateAll` (matrices) if 3D relief is active (otherwise invariant on pan).
4. **VRAM leak fixed**: `_applyGLBEmissive` clones the material **only once**
   (`_eclextCloned` flag) then only mutates `emissive`/intensity.
5. **Range compliance**: `computeIntensity` handles `%` and `VA` (absolute value,
   P→power, F→flux); `getActiveKelvin` applies **TC** ranges (color temperature).
   Support for simultaneous multiple ranges (`getActivePlages`).
6. `WebGLRenderer.dispose()` on `onRemove`.
7. **XSS**: helper `esc()` applied to all Grist data injected into
   `innerHTML` / `value` attribute.
8. `grist.onRecords` **debounced** (400 ms) → no more reloading 4 tables on every keystroke.
9. OSM/CSV/GeoJSON imports via **`BulkAddRecord`** (`bulkAddPoints`) instead of one `AddRecord` per point.
10. **Responsive**: on narrow container, fullscreen panel + scrollable toolbar. Dead code cleanup.

## Points of Attention / Limitations

- **three.js r128** kept (UMD build + `examples/js/GLTFLoader.js` global).
  Migration to a maintained version = switch to ESM/importmap (would change the
  global pattern). To do later if needed.
- `addProtocol` workers: `importScripts` loads geotiff from CDN;
  `OffscreenCanvas`/`convertToBlob` required (Chrome/Edge/Firefox, Safari ≥ 16.4).
  Flat tile fallback if decoding fails.
- Light cones with `depthTest:false` (not occluded by relief/buildings) — original
  visual choice preserved.
- **Not tested in a real browser** in the dev environment (no headless browser):
  validate WebGL rendering, DEM decoding, and Grist integration in real conditions.

## Publication

Not published. To publish: `published/eclext/` + `package.json` (`grist` section,
`accessLevel: full`) + copy of `index.html`, then `npm run manifest`.
