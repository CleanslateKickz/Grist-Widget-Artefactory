# "Geo Data in Grist" Convention (v1)

**Shared contract** for the geospatial suite, centered on **Grist as the single
source of truth**. Three components, one convention:

| Component | Role | Direction |
|---|---|---|
| **QgisRemoteMCP** (server, PyQGIS) | exports native deliverables (`.qgz`, GPKG, styled) **and** to Grist; AI automation | QGIS ↔ Grist (heavy) |
| **qgis2grist** (browser) | imports QGIS exports (qgis2web / `.qgz` / GPKG / QField / QML) → Grist tables | QGIS → Grist (interactive) |
| **Atlas** (browser) | reads / renders (2D+3D) / edits / filters / tells stories (storymaps); lightweight export | Grist ↔ map |

Goal: **universal canvas** for maps + QGIS projects, standardized and reversible.
Both entry points (QgisRemoteMCP server, qgis2grist browser) must **write the same
standard tables** → Atlas is source-agnostic.

> Status: **v1 — implemented on the Atlas side** (reading/linking, auto-table-ization,
> per-object config, controls, storymaps). Still pending formal alignment of
> QgisRemoteMCP/qgis2grist with this contract (see §11). Evolve this document, not
> the code in silos; **version it** (bump the header on any contract change).

---

## 1. Principle

- A **layer** = a *view* on a **source**. Two source types:
  - `kind: 'blob'` — the layer carries its own GeoJSON (drawings, quick imports).
    Atlas works **as-is**, without Grist.
  - `kind: 'table'` — the layer is **linked** to a "1 row = 1 feature" table.
    Atlas builds the FeatureCollection **on the fly** from the table (the join
    lives in the widget, see §6), edits **row by row**, and can keep a blob
    as a **cache** (standalone / perf / snapshot).
- Stored geo data is always in **WGS84 (EPSG:4326)**.

---

## 2. "1 row = 1 feature" Data Table

One table per layer. One row = one feature. Columns:

| Column | Role | qgis2grist Status |
|---|---|---|
| `geometry_json` | **GeoJSON (string)** geometry, WGS84 | ✅ written |
| `latitude` / `longitude` | convenience for points | ✅ written |
| *attributes* (typed) | data; `Choice` (ValueMap), `Ref`/`RefList` (relations) | ✅ written |
| `fill_color` | **per-feature** color (hex), derived from QGIS renderer | ✅ written |
| `stroke_color`, `size`/`radius`, `opacity` | per-feature style (optional) | ➕ to add |
| `height`, `elevation` | 3D: extrusion / altitude (optional) | ➕ |
| `model_id` | per-feature 3D model: id from integrated catalog, or custom URL | ➕ |
| `model_glb` | per-feature 3D model **as attachment** on the feature (*Attachments* type) — **takes priority** | ➕ |
| `label` *(or designated field)* | label | ➕ |
| `hidden` | hides the feature | ➕ |

**Geometry detection** (order): `geometry_json` → `geometry` → `geom` →
`wkt` (parse) → `latitude`+`longitude` pair (aliases `lat`/`lng`/`lon`).

**Identity**: the **Grist rowId** is the stable key (selection, editing, write-back).

**Override = the column itself.** Each display attribute column **IS**
the per-feature override: a non-null value overrides the layer default (see §2bis).
Only populate what you customize (*sparse* storage, efficient).

---

## 2bis. Parameter Resolution (layer default → feature override)

For each display parameter (`model`, `scale`, `rotation_*`, `offset_*`,
`height`, `fill_color`, `label`, `hidden`), precedence from most specific to most
general:

1. **Per-feature override** — value from the attribute column (layer `table`); or
   `feature.properties._*` (layer `blob`).
2. **Field binding** — categorized/graduated symbology (param ↔ field).
3. **Layer default** — `style.common` / `style.library`.
4. **Model default** — params carried by the model (catalog).

**3D model**: `model_glb` (feature attachment) > `model_id` (catalog / URL) > field
binding > layer default > model default.

**Loading a model as attachment**: read the cell's attachment id →
`getAccessToken()` → `{baseUrl}/attachments/{id}/download?auth={token}` →
`GLTFLoader`. Model is **cached** (loaded once); short-lived token → fetched
on the fly at load time.
**Upload from Atlas**: no plugin method available → REST `POST {baseUrl}/attachments`
with the token (⚠️ confirm that the token allows writing; otherwise fallback:
attach via the Grist UI, Atlas reads).

**Instancing**: a model shared between multiple features = **same URL / same
attachment id** → a single `InstancedMesh` (performance preserved). To apply a model
to an entire layer/selection, propagate the same id.

---

## 3. Layer Registry

Table describing **what is a geo layer and how to display it**. This is the
generalization of `Maquette_Layers`. Atlas displays **only layers declared
here**; other geo tables in the document are only *proposed "to link"*.

One row per layer:

| Field | Role |
|---|---|
| `name` | layer name |
| `kind` | `'blob'` \| `'table'` |
| `sourceTable` | **linked table id** (text; resolved by Atlas — not a Grist Ref, see §7) |
| `geometryColumn` | name of the geometry column in the source |
| `geomType` | `Point` \| `Line` \| `Polygon` (+ Multi) |
| `geojson` | blob FeatureCollection — source if `kind:'blob'`, **cache** if `'table'` |
| `styleJSON` | portable style (see §4) |
| `color`, `visible`, `order`, `group` | default color, visibility, order, group (QGIS tree) |

---

## 4. Portable Style (`styleJSON`)

Common superset **QGIS QML ↔ Atlas symbols**. Format:

```json
{
  "mode": "single | categorized | graduated | model",
  "field": "<field for categorized/graduated>",
  "color": "#RRGGBB",                         // single
  "categories": [{ "value": "...", "color": "#...", "modelId": "..." }],
  "ranges":     [{ "lower": 0, "upper": 10, "color": "#..." }],
  "size":   { "mode": "fixed|graduated", "value": 8, "field": "...", "range": [0.5,3] },
  "height": { "field": "...", "value": 12 },  // extrusion / 3D
  "model":  { "modelId": "...", "field": "..." },
  "label":  { "field": "...", "enabled": true }
}
```

**Precedence** (strongest to weakest):
layer's `styleJSON` (rule) **>** per-feature style columns (`fill_color`…,
the QGIS "baked" style) **>** layer default.
→ QGIS rendering is faithful by default, and remains editable in Atlas.

**QML → styleJSON mapping** (already parsed by qgis2grist):
`singleSymbol → single`, `categorizedSymbol → categorized`,
`graduatedSymbol → graduated`.

---

## 5. Reversible Conversions (2 commands)

- **Downward — layer → 1-1 table** ("Explode to table"): Atlas reads the blob,
  creates the standard table (§2), switches the layer to a linked `kind:'table'`.
- **Upward — 1-1 table → layer** ("Link a table"): the user chooses a geo table;
  Atlas creates the registry row (binding + detected style) and renders.

Both share this contract → qgis2grist imports and Atlas drawings converge.

---

## 6. Join & Refresh

- The **join lives in Atlas (JS)**: it reads any table via `docApi`
  and builds the FeatureCollection. (Grist doesn't allow a generic formula/Ref
  to a variable table — see §7.)
- `kind:'table'` editing → **row-by-row write-back** to the columns in §2.
- Linked table refresh: **on focus / tab return** (lightweight), no real-time
  multi-table.

---

## 7. Grist Constraints (must respect)

- Reactivity (`grist.onRecords`) **only** on the widget's mapped table → other
  linked tables are read via `docApi.fetchTable` (one-shot) + refresh §6.
- A `Ref`/`RefList` column targets **a fixed table** → the layer→table link is
  stored as **text** (`sourceTable`) and resolved by Atlas.
- `requiredAccess: 'full'` required (multi-table reads + creation/writing).

---

## 8. Automations

**Safety line**: *automatic* for **read / display / derive / suggest**;
*explicit (or reversible + confirmed)* for **write / create / modify** Grist data
(table creation, bulk write-back, reprojection).

Automatic:
- mounting layers **declared in the registry**; detecting other geo tables →
  proposed "to link";
- geometry column detection, geo type, label field detection;
- **auto-fit** camera on data extent;
- applying per-feature "baked" style; otherwise **default symbol** (categorized on
  low-variance text, graduated on numeric, auto palette);
- **auto-suggested controls** from fields (date → time slider/animation,
  numeric → range, categorical → selector), bounds/options inferred from data;
- 3D placement on terrain + re-calibration (DEM tiles, terrain toggle, ground, resize);
- fallback: missing model → hit circle; unreadable DEM → flat tile.

Explicit / confirmed:
- registry creation, blob→table explosion, table linking;
- write-back to Grist;
- **non-WGS84 CRS**: best-effort reprojection if projection info is available,
  **otherwise alert** (redirect to qgis2grist).

---

## 9. Controls (filter/animation rack) — implemented

Per layer, a `controls` array (persisted in `StyleJSON._controls` on the Grist side,
and in the JSON project):
```json
[{ "field": "...", "type": "time|range|select", "active": true,
   "min": <num|ts>, "max": <num|ts>, "values": ["..."] }]
```
- `time` (date field), `range` (numeric), `select` (categorical) — **auto-detected**
  from fields. Filter **both 2D and 3D map** (same predicate). `time` animates.
- Orientation precedence: `styleJSON.orientation.field` binds azimuth to a field.

## 10. Story / Storymaps — `Atlas_Story` table

Sequence of replayable steps (presentations). One row = one step:

| Column | Role |
|---|---|
| `Step` (Int) | order |
| `Title` (Text) | step title |
| `Description` (Text) | narrative text |
| `StateJSON` (Text) | snapshot: `camera` (center/zoom/pitch/bearing), `projection`, `timeOfDay`, `date`, `terrain3D`, `layers[]` (visibility + active controls) |

Atlas captures / replays (presentation mode). Editable on the Grist side. (Media as attachments: coming soon.)

## 11. Roles & Compliance (the 3 components)

- **Writers** (QgisRemoteMCP, qgis2grist): produce **compliant geo tables**
  (§2) — WGS84 geometry column, typed attributes, optional per-feature
  `fill_color`, optional `model_id`/`model_glb`. **Same schema regardless of source.**
- **Reader/editor** (Atlas): mounts declared layers, applies style/controls,
  edits per feature (column write-back), invents nothing outside convention.
- **Round-trip**:
  - **light / browser** = Atlas exports **GeoJSON + QML** (style) → QGIS-readable.
  - **heavy / server** = QgisRemoteMCP rebuilds native `.qgz`/GPKG from Grist.
- **Auto-table-ization** (Atlas): every import becomes a standard table (the reference
  data); blob fallback outside Grist.
- **Deletion** (Atlas): "remove from Atlas" (keeps the table) vs "delete the
  table" (`RemoveTable`).
- **Compliance test** (recommended): a writer must be able to re-read what it
  wrote via the reference reader (geometry detection + types + style).

## 12. Out of scope / upcoming

- **GPKG** (binary) and `.qgz`: reserved for QgisRemoteMCP (server).
- **In-widget attachment upload**: depends on instance CORS (otherwise native attach).
- Storymap media (image attachments), transitions; layer tree/groups; inter-widget
  selection (map click ↔ table row).
