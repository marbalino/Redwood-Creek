# Redwood Creek Map

Interactive web map for the Redwood Creek restoration project at 5750 Redwood Road, Oakland, CA. Built with Leaflet.js as a static site — no server-side dependencies, just clone and serve.

## Features

- **Layer toggles** — turn individual layers on/off from the side panel
- **V1 / V2 switcher** — compare two versions of the creek bank and centerline geometry
- **Basemaps** — toggle between OpenStreetMap and ESRI Satellite
- **Compass rotation** — drag the compass rose to rotate the map to any bearing; click to snap back to north
- **Right-click coordinates** — right-click anywhere to get lat/lng and copy to clipboard
- **Geolocation** — crosshair button (bottom-left) tracks your live GPS position on the map

## Layers

| Layer | Source |
|---|---|
| Creek Bank | Field survey shapefile |
| Creek Survey / Centerline | Field survey shapefile |
| Sewer Manhole | City infrastructure data |
| Sewer Lamphole | City infrastructure data |
| Outfall | Field survey shapefile |
| Creek Restoration Area | Project boundary shapefile |
| 300ft Radius | Project buffer shapefile |
| Inlets | City drainage dataset (filtered to project area) |
| Oak Drains | City drainage dataset (filtered to project area) |
| Elevation | GDAL hillshade derived from LiDAR surface model |

## Project Structure

```
redwood-creek-map/
├── index.html          # Full map application (single file)
├── data/               # GeoJSON exports of all vector layers
│   ├── creek_bank_v1.geojson
│   ├── creek_bank_v2.geojson
│   ├── creek_survey_v1.geojson
│   ├── creek_centerline_v2.geojson
│   ├── creek_restoration_area.geojson
│   ├── buffer_300ft.geojson
│   ├── outfall.geojson
│   ├── sewer_manhole.geojson
│   ├── sewer_lamphole.geojson
│   ├── inlets.geojson
│   └── oak_drains.geojson
└── tiles/
    └── redwood_rd_surface/   # XYZ tiles for elevation hillshade (zoom 14-20)
```

## Running Locally

Requires any static file server (the map uses `fetch()` so it won't work from `file://`).

**Node.js:**
```bash
npx serve .
```

**Python (via QGIS OSGeo4W):**
```bash
"C:\Users\...\OSGeo4W\bin\python3.exe" -m http.server 8080
```

Then open `http://localhost:8080` (or the port shown).

## Deploying to a Server

Clone the repo into your web root and serve as static files:

```bash
git clone https://github.com/marbalino/Redwood-Creek.git /var/www/html/redwood-creek
```

No build step required. Works with nginx, Apache, or any static host.

## Data Notes

- All vector layers are in **EPSG:4326** (WGS84)
- Source shapefiles use **EPSG:2227** (NAD83 / California Zone 3, US survey feet)
- Elevation tiles are hillshade-rendered from a LiDAR-derived surface model
- Inlet and drain datasets are filtered to the project bounding box — full Oakland datasets not included
- Geolocation requires **HTTPS** in production (works on localhost without it)
