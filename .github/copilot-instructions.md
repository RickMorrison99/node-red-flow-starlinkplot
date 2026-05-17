# Copilot Instructions

## Project Overview

This is a **Node-RED flow** (`flows.json`) that fetches live Starlink satellite orbital elements from Celestrak and plots visible satellites on a real-time scatter chart via Node-RED Dashboard 2.

## Architecture

The flow (`Flow 2`) has three independent pipelines that share state through Node-RED global memory:

### Pipeline 1 — Live Data Fetch
`Inject → HTTP GET (Celestrak API) → Change (set filename) → File write (/data/sat_<timestamp>.csv)`

Fetches from: `https://celestrak.org/NORAD/elements/gp.php?GROUP=starlink&FORMAT=csv`

### Pipeline 2 — Load from Disk
`Inject → File read (/data/sat.csv) → f(x): CSV to Memory`

Reads a local file and stores the raw CSV string into `global.rawStarlinkData`.

### Pipeline 3 — Animation Loop
`Frame Generator (repeats every 1s) → F(x): Parse and Calculate Live Positions → ui-chart (scatter) + debug`

Reads `global.rawStarlinkData` on each tick, computes azimuth/elevation for each satellite, and sends up to 75 visible satellites to the dashboard chart.

## Key Conventions

### Data Format
`data.csv` and the Celestrak API both return GP (General Perturbations) orbital elements in CSV format with this column order:
```
OBJECT_NAME, OBJECT_ID, EPOCH, MEAN_MOTION, ECCENTRICITY, INCLINATION(col5), RA_OF_ASC_NODE(col6), ARG_OF_PERICENTER, MEAN_ANOMALY, ...
```
Rows are filtered by checking `row[0].startsWith("STARLINK")`.

### Position Calculation
The flow does **not** use SGP4 propagation. It uses a two-body approximation:

1. **Propagate mean anomaly**: `M = M0 + n * dt_days * 360` (using epoch + mean motion from CSV)
2. **Argument of latitude**: `u = argPericenter + M` (valid for low-eccentricity Starlink orbits)
3. **Sub-satellite point**: spherical trig using inclination and `u` → geocentric lat/lon in ECI frame
4. **ECI → ECEF**: subtract Greenwich Apparent Sidereal Time (`GAST = 280.46061837 + 360.98564736629 * t_J2000_days`)
5. **Elevation**: `atan((cos_c - Re/(Re+h)) / sin(c))` where `c` = central angle, `Re=6371 km`, `h=550 km`
6. **Azimuth**: standard spherical bearing formula
7. **Polar chart**: `radius = 90 - elevation`, project to Cartesian `x/y` with `angleRad = (az - 90) * deg2rad`

Only satellites with `elevation > 0` (within ~23° central angle from observer) are plotted. Typically ~400–500 visible satellites over Calgary at any moment.

### Observer Coordinates
Hardcoded in the `F(x): Parse and Calculate Live Positions` function node:
```js
const OBSERVER_LAT = 51.0447;   // Calgary, AB
const OBSERVER_LON = -114.0719;
```
Change these to reposition the observer.

### Global State
- `global.rawStarlinkData` — raw CSV string set by Pipeline 2; read by Pipeline 3 on every animation frame. If absent or empty, Pipeline 3 returns `null` (stops silently).

### Dashboard
Uses `@flowfuse/node-red-dashboard` v1.30.2 (FlowFuse Dashboard 2, not the older `node-red-dashboard`). The chart node uses `action: "append"` with a 1-hour rolling window. Send `msg.payload = []` to clear the chart.

## AllCharts.json

A second flow providing three simultaneous visualizations, all driven by the same compute pipeline:

| Page | Path | Type | Node |
|---|---|---|---|
| SVG Radar | `/starlink/svg-radar` | `ui-template` (Vue SVG) | `ac_svg_tpl` |
| Canvas Radar | `/starlink/canvas-radar` | `ui-template` (HTML5 Canvas) | `ac_canvas_tpl` |
| Ground Track | `/starlink/ground-track` | `ui-template` (Leaflet.js) | `ac_map_tpl` |

**Pipeline:** `Inject (hourly) → HTTP GET → Store TLE` and `Inject (2s) → Compute Positions (3 outputs) → 3 × ui-template`

**Trail history** stored in `global.satHistory` — map of satellite name → last 20 positions, max 60 s age. Reset to `{}` on each TLE fetch. The compute function propagates each satellite's mean anomaly from epoch to `Date.now()` on every 2-second tick.

**Color coding by inclination shell:** 43°=`#ff7700`, 53°=`#00aaff`, 70°=`#00ee88`, 97°=`#cc44ff`

**Leaflet** tiles load from CartoDB dark CDN (`unpkg.com/leaflet@1.9.4`). Uses `preferCanvas:true` + `circleMarker` for performance with 300+ markers. Trail polylines use `weight:0.8, opacity:0.35`.

### File Paths
Downloaded satellite data is written to `/data/sat_<millis>.csv`. The manual-load pipeline reads from `/data/sat.csv`. These paths are absolute and assume Node-RED runs with access to a `/data/` directory (e.g., in Docker).
