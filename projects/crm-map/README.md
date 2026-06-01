# CRM Map

> Interactive property map widget for Grist — visualize, filter, and add properties directly on a map. Built for commercial real estate CRM workflows.

## Features
- **Property visualization** — Plot all properties from your Grist document on an interactive Leaflet map
- **Marker clustering** — Automatic clustering for performance with large datasets
- **Classification filters** — Filter markers by owner classification (Contact, IPA, Not Interested/DNC, etc.)
- **Geocoding search** — Search addresses via Nominatim or match existing properties
- **Add property** — Click on the map, fill in property + owner details, save to Grist
- **Reverse geocoding** — Auto-fills address, city, state, zip from map click location
- **Owner management** — Search existing owners or create new ones inline
- **Popup cards** — Expandable property details with copy address, street view, and external links
- **Basemap switcher** — Google Hybrid, ArcGIS Street, or OpenStreetMap

## Installation
1. In Grist: **Add Widget → Custom → Enter Custom URL**
2. URL: `https://CleanslateKickz.github.io/Grist-Widget-Artefactory/crm-map/`
3. Grant **"Full document access"**

## Required tables
The widget expects two tables:

### Properties_
| Column | Type | Description |
|--------|------|-------------|
| Name | Text | Property name or owner names |
| Latitude | Numeric | Latitude coordinate |
| Longitude | Numeric | Longitude coordinate |
| Property_Address | Text | Street address |
| Property_Id | Text | Auto-generated property ID |
| City | Text | City |
| State | Text | State |
| Zip | Text | ZIP code |
| ImageURL | Text | Property image URL |
| CoStar_URL | Text | CoStar listing link |
| PrimaryOwner | Text | Owner name for classification |

### Owners_
| Column | Type | Description |
|--------|------|-------------|
| Name | Text | Owner name |
| Classify__Color | Text | Classification (Contact, IPA, etc.) |

## Usage
1. **Browse** — Properties appear as colored markers based on owner classification
2. **Filter** — Click the target icon to toggle classification visibility
3. **Search** — Click the search icon and type an address or property name
4. **Add** — Click the + button, then click anywhere on the map
5. **Fill** — The address form auto-populates; choose existing or new owner
6. **Save** — Click Save to create the property and owner in Grist

## Classification colors
| Status | Marker Color |
|--------|-------------|
| Contact / Call Again | Green |
| Not Interested/DNC | Red |
| IPA | Violet |
| Eric | Orange |
| Broker/Eh | Grey |
| Never | Blue |
| Call Relationship | Yellow |

## CRE CRM Use Cases
- **Portfolio map** — See all properties at a glance with color-coded owner status
- **Property intake** — Click on a map to add a new property with coordinates
- **Market analysis** — Filter by owner classification to target motivated sellers
- **Field work** — Search properties by address, get directions, view street imagery
