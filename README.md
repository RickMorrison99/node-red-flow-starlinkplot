# node-red-flow-starlinkplot

Real-time Starlink satellite tracker for Node-RED with three simultaneous visualizations: SVG polar radar, Canvas polar radar, and a Leaflet world-map ground track.

![Three views: SVG radar, Canvas radar, Leaflet map](https://img.shields.io/badge/Node--RED-Dashboard%202-red)
![Satellites: ~300–500 visible](https://img.shields.io/badge/Starlink-300--500%20visible-blue)

## Features

- **Real-time position propagation** — two-body orbital mechanics with GAST, updated every 2 seconds
- **Three simultaneous views** — SVG radar, Canvas radar, Leaflet ground-track map
- **TLE caching** — fetches from Celestrak every 2 hours; falls back to local cache on HTTP error
- **Click to select** — click any satellite dot to see elevation, azimuth, lat/lon, illumination status
- **Filters** — min elevation slider and per-shell toggles (43°/53°/70°/97°)
- **AOS/LOS alerts** — configurable favorites list with notification banners
- **Illumination indicator** — gold ring on satellites that are sunlit while you're in darkness
- **TLE freshness badge** — green/amber/red age indicator in every view header
- **Pass prediction** — on-demand next-6-hour pass table for any selected satellite
- **CPU-efficient** — compute loop pauses automatically when no browser clients are connected

## Prerequisites

| Requirement | Version |
|---|---|
| Node-RED | ≥ 3.1 (junction nodes require 3.1+) |
| `@flowfuse/node-red-dashboard` | ≥ 1.30 (Dashboard 2) |
| Internet access | For initial Celestrak fetch |

## Import

1. Open Node-RED → **☰ Menu → Import**
2. Paste the contents of `AllCharts.json`
3. Click **Import**, then **Deploy**

## Configuration checklist

### Required
- [ ] **Cache file path** — defaults to `/data/starlink_tle_cache.csv` (Docker mount).
  For bare Node-RED, change the path in three nodes: *Check Cache Age*, *Save & Store TLE*, *Load Cache Fallback*.
  Example: `/home/pi/.node-red/starlink_cache.csv`

### Recommended
- [ ] **Observer location** — open the Settings page and enter your lat/lon (or click *Use My Location*).
  Default is Calgary, AB (51.0447°N, 114.0719°W).

- [ ] **Persistent context** — required for observer location and favorites to survive Node-RED restarts.
  Add to `settings.js`:
  ```js
  contextStorage: {
    default: { module: 'memory' },
    file:    { module: 'localfilesystem' }
  }
  ```

### Optional
- [ ] **Favorites / AOS alerts** — add satellite names on the Settings page to receive rise/set notifications.

## Dashboard pages

| Page | URL | Description |
|---|---|---|
| SVG Radar | `/starlink/svg-radar` | Vue SVG polar radar, elevation rings, click-to-select, filter controls |
| Canvas Radar | `/starlink/canvas-radar` | HTML5 Canvas radar, same layout as SVG, click hit-test |
| Ground Track | `/starlink/ground-track` | Leaflet world map, antimeridian-safe trails, click markers |
| Settings | `/starlink/settings` | Observer location, favorites, AOS/LOS threshold |

Access at: `http://<your-node-red-host>:1880/starlink/svg-radar`

## Flow architecture

```
LEFT COLUMN — Logic                    RIGHT COLUMN — UI Templates
─────────────────────────────────────  ───────────────────────────────────────
[Fetch Inject (2h)]                    [link in] → [SVG Radar Template]
  → [Check Cache Age]                                  ↓ (user events)
  → [GET Celestrak CSV]                [link in] → [Canvas Radar Template]
  → [HTTP OK?]                                         ↓
      → jct → [Save & Store TLE]       [link in] → [Ground Track Template]
      ↘ jct → [Load Cache Fallback]                    ↓
                                       [link in] → [Alerts Template]
[Animate Inject (2s)]
  → [Viewer Gate]  ←── [ui-control]   [Settings Template]
  → [Compute Positions]                    ↓
      → link out ──────────────────→  [link in] ─→ [Apply Settings]
      → link out ──────────────────→
      → link out ──────────────────→  All template events → shared link-in
      → link out ──────────────────→
```

## Satellite color coding

| Color | Orbital Shell |
|---|---|
| 🟠 Orange `#ff7700` | 43° inclination |
| 🔵 Blue `#00aaff` | 53° inclination |
| 🟢 Green `#00ee88` | 70° inclination |
| 🟣 Purple `#cc44ff` | 97° inclination |
| 🟡 Gold ring | Satellite is sunlit (observer in darkness) |

## Data source

Orbital elements from [Celestrak GP](https://celestrak.org/NORAD/elements/gp.php?GROUP=starlink&FORMAT=csv) in CSV format.
Downloaded every 2 hours and cached locally. The orbital math uses a two-body approximation with GAST-based ECI→ECEF conversion — accurate to within a few degrees for visualization purposes.

## Files

| File | Description |
|---|---|
| `AllCharts.json` | Main flow — all three views, caching, features |
| `flows.json` | Original single-view scatter chart (legacy) |
| `.github/copilot-instructions.md` | Developer context for Copilot sessions |
