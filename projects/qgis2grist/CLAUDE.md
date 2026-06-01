# Project: qgis2grist

Grist widget for importing QGIS projects (qgis2web HTML/ZIP, .qgz/.qgs, GeoPackage,
QField packages). Creates the Grist table schema and populates it with data; then renders
a live-synchronized Leaflet map.

## Context

The goal is not to display a map on top of QGIS — it is to **make QGIS data natively
usable in Grist**: typed columns, choices, relations, and human-readable labels. The map
is a byproduct for visualizing the result.

## Architecture

### State Machine

```
drop → reading → preview → importing → map ↔ original
                                              ↓
                                           error
```

Current state in `currentState`. Transitions via `setState(name)`.

### Parsing Pipeline

| Format | Entry point | Common output |
|--------|-----------|---------------|
| qgis2web HTML | `parseQgis2webHtml(html)` | `[layer, ...]` |
| qgis2web ZIP | `parseQgis2webZip(zip)` | same |
| QGZ | `parseQgzFile(ab, parentZip, name)` | same (delegates to `parseQgsXml`) |
| GeoPackage standalone | `parseGpkgFile(ab)` | same |
| GeoJSON in ZIP | `parseGeojsonFiles(zip, names)` | same |

`layer` = `{name, displayName, geomType, fields, features, featureCount, hasData,
datasource, style, _layerId?}`.

`fields[]` = `{name, _rawKey, label, qType, gType, widgetOptions?, description?,
_valueMap?, _refTargetTable?, _refTargetField?, _externalResource?}`.

### Grist Import Pipeline

`startImport()` :
1. Topological sort `topoSortByRefs` — parents before children for Refs.
2. For each layer, in order:
   - `listTables()` → generate a unique name.
   - `AddTable` with `label`, `widgetOptions`, `description`.
   - `BulkAddRecord` loop in batches (100 if Polygon/Line, 500 if Point).
   - Capture new rowIds via `result.retValues[0]`.
   - If the layer is referenced by others: index `value → rowId` in `parentMaps[tableName][refField]`.
3. When inserting a child layer, transform each FK to rowId via `parentMaps`.

### Column Metadata (passes 1b + 2)

| QGIS Source | → Grist |
|-------------|---------|
| `<aliases><alias>` | `label` |
| `<editWidget type="ValueMap">` | `Choice` + `widgetOptions.choices` + `_valueMap` (transform on import) |
| `<editWidget type="Range">` | `Numeric` (or `Int` if Step is integer without Precision) |
| `<editWidget type="CheckBox">` | `Bool` |
| `<editWidget type="DateTime">` | `Date` or `DateTime` depending on `field_format` |
| `<editWidget type="ValueRelation">` | `Ref:` or `RefList:` (pass 3) |
| `<editWidget type="ExternalResource">` | `_externalResource` marker (Attachments coming soon) |
| `<relations><relation>` | `Ref:<TargetTable>` on the referencing field |
| GPKG `gpkg_data_columns.title/description` | `label`/`description` |
| GPKG `gpkg_data_column_constraints type='enum'` | `Choice` |
| GPKG `qgis_projects.xml` | full `.qgs` → aliases + editWidgets if not already read |

### Persistence

A `QgisWidgets` table (auto-created) stores a JSON config per import. Enables restoration on next widget load. Schema: `widget_name`, `source_file`, `config_json`, `created_at`. The `config_json` contains meta BigQgisMCP, enriched fields, refs, styles.

No `setOption()` here: we want the config to survive a document duplicate (the table is part of the document, the option is not).

## Project-Specific Conventions

### Geometry

- Point: `latitude`, `longitude` columns (Numeric).
- Line / Polygon: `geometry_json` (Text, JSON GeoJSON rounded to 5 decimals ≈ 1 m), `centroid_lat`, `centroid_lon`.
- No Grist `Ref:Geometry` — geometry is serialized.

### Per-Feature Color

Auto `fill_color` column (Text) calculated at import via QML / qgis2web functions. Allows any Grist view to retrieve the color without re-parsing the style. **Downside**: if the user edits the classification value, `fill_color` is not recalculated. To document for the user.

### `_rawKey` vs `label` vs `name`

- `_rawKey` = raw key in GeoJSON properties (e.g.: `ht_max`).
- `label` = human-readable label displayed (QGIS alias / `gpkg_data_columns.title` / or `_rawKey` as fallback).
- `name` = sanitized Grist id (e.g.: `ht_max`, or `Hauteur_d_eau` if special characters).

`flattenGeoJsonFeatures` looks for `props[_rawKey]` first.
`makeMarkerColorFn` (style) searches in this order: `_rawKey`, `label`, `name`.

## Current Status

Done:
- Parsers for qgis2web HTML/ZIP, QGZ/QGS, GPKG (with sql.js + pure JS WKB), GeoJSON.
- Native EPSG:3857 reprojection, other CRS via proj4js on-demand.
- QML (categorized/graduated/single) color extraction.
- BigQgisMCP: title, slider, flood/building palettes, dynamic legend.
- Import: labels, Choice widgetOptions, Ref: relations, FK → rowId, GPKG metadata.
- Leaflet live-synchro map (Grist polling 5 s) + restoration banner.

To do / known limitations:
- **ExternalResource → Attachments**: marker placed (`_externalResource`) but file upload from QField package not yet implemented. Requires `grist.docApi.uploadAttachment(blob)` + relative path reconstruction.
- **QGIS 2.x**: legacy `<edittypes>` detected but their `widgetv2config` format (with `<value key= value=>`) is not parsed — ValueMap degraded to Text.
- **5 s polling without backoff**: expensive on large tables. No pause when tab is in background.
- **`adaptHtmlForGrist` / `renderAsWidget`**: ~360 dead lines, to remove or re-integrate.

## Points of Attention

### `BulkAddRecord` retVal capture

`result.retValues[0]` contains the array of new rowIds. This capture is essential for Refs; without it we cannot index parents.

### Topological Sort

Cycles ignored via `visiting`/`visited`. If A references B and B references A, both are emitted in discovery order; cyclic Refs will not be resolved correctly (acceptable, this is an exotic case in QGIS).

### Table Rename on Collision

`tableNameRemap[layer.name] = tableName` records the mapping before `AddTable`. `resolveRefType` then translates `Ref:OldName` → `Ref:NewName` when creating Ref columns in children.

### `_valueMap` and the Grist Label

QGIS categorized values are transformed into Choice LABELS at import. If the user adds a new value in Grist that doesn't exist in `_valueMap`, it will be stored as-is. This is OK: Grist accepts out-of-`choices` values (they are marked invalid in the UI but persisted).

## Reusable Patterns

- `parseOptionTree(el)`: recursive parser for `<Option type="Map|List|...">`, `QgsXmlUtils::writeVariant` serialization format. Reusable for other QGIS web tools.
- `WkbReader`: pure JS WKB/EWKB/ISO Z·M·ZM parser, ~50 lines.
- `topoSortByRefs(layers)`: generic topological sort on Ref graphs.

## Manual Tests

No automated test suite. To validate a change, test with:
1. A simple `qgis2web` ZIP export (single Polygon layer).
2. A `.qgz` QField project with 1-N relations and ValueMap.
3. A standalone GeoPackage with `gpkg_data_columns` and `gpkg_data_column_constraints`.
4. A BigQgisMCP project (HTML flood with slider).

Verify in Grist:
- Human labels present on columns.
- `Choice` with valid choices for ValueMap.
- `Ref:Parent` clickable on FKs (the Grist widget must display the parent row).
- Restoration after widget reload.
