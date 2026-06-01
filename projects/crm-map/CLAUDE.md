# Project: CRM Map

## Context
Interactive property map for commercial real estate CRM. Visualizes properties from Grist on a Leaflet map with marker clustering, classification filters, geocoding search, and a point-and-click sidebar for adding new properties and owners.

## Architecture
- Single standalone file: `index.html` (~1425 lines)
- Leaflet map with MarkerCluster plugin for performance
- Three basemap options: Google Hybrid, ArcGIS Street, OpenStreetMap
- Search via Nominatim geocoding (JSONP) and local property index match
- Classification filter system using owner-level Classify Color
- Sidebar form with Property + Owner tabs for creating records
- Grist integration reads Properties_ and Owners_ tables

## Specific conventions
- Table mapping via `COL` object (Name, Longitude, Latitude, Property_Id, etc.)
- Classification system with 8 categories mapped to colored Leaflet markers
- Owners_ table expected to have Classify__Color (or variants) and Name columns
- Properties_ table expected with Longitude, Latitude, Property_Address, City, etc.
- Marker icons use `leaflet-color-markers` GitHub CDN (red, violet, orange, grey, blue, yellow, green)
- Property ID format: `{number} - ❌` (auto-incremented from existing max)
- Owner lookup uses same classification color for property markers
- Popup cards with copyable address, street view link, and CoStar URL button

## Current state
- Map display with clustering working
- Classification filter chips with toggle per category
- Add-property sidebar with reverse geocoding (Nominatim)
- Owner search, existing/new owner toggle
- Property and Owner creation via `applyUserActions`
- Marker popup with expandable details and action buttons
- Search with property match + Nominatim fallback
- Basemap switcher

## Points of attention
- Nominatim JSONP callbacks used instead of fetch (no CORS issues)
- Property ID auto-increment may need manual adjustment for new datasets
- Owners_ and Properties_ table names are hardcoded — change `COL` object for different schemas
- City and County dropdowns pre-populated with Oregon-specific options — modify CITY_CHOICES/COUNTY_CHOICES for other regions
- Marker clustering threshold adjusts at zoom level 16 (z>=16: 10px cluster radius)
- Gold marker used for currently selected row via `grist.setCursorPos`
- Classification colors hardcoded in `classifyColorHex` — edit for custom categories
