# The Most Detailed Map of Europe's Trees

An interactive scrollytelling web map showing Europe's forest cover density and tree types, built by **ViT** using real satellite data from the Copernicus Land Monitoring Service.
<img width="1418" height="893" alt="image" src="https://github.com/user-attachments/assets/d2c4c298-ff17-4fbd-bc10-5970df11659f" />


---

## What this project is

A data journalism / visual storytelling piece. The user scrolls through a narrative while a full-screen map automatically pans to relevant regions and switches data layers. The final step unlocks a free-exploration mode where the user can search locations and switch layers manually.

**Live experience flow:**
1. **Intro screen** — full-screen title overlay with a loading/ready state indicator
2. **Step 0** — Europe-wide view, tree cover density layer
3. **Step 1** — Zooms into southern Spain to show dry coverage
4. **Step 2** — Alternatively, zooms into Sweden to show sparse coverage
5. **Step 3** - Europe-wide view again, switches to layer that shows tree type (broadleaved or coniferous).
6. **Step 4** - Shows broadleaved territory.
7. **Step 5** - Shows coniferous territory.
8. **Step 6** - Introduces new layer that has both: tree cover density and tree type.
9. **Step 7** — Compares Lisbon (low density broadleaved) vs Ljubljana (high density)
10. **Step 8** — Zoom out, introduces exploration.
11. **Exploration** — Free exploration: search any location, switch between 5 layer modes

---

## Data sources

All forest data comes from the **Copernicus Land Monitoring Service (CLMS)**, a programme of the European Environment Agency (EEA). It is served live via EEA's public ArcGIS ImageServer — no file download or pre-processing required.

| Layer | Service name | What it shows |
|---|---|---|
| Tree Cover Density (TCD) | `HRL_TreeCoverDensity_2018` | 0–100 % canopy cover per 10 m pixel. Yellow = sparse, dark green = dense. Special codes: 253 = outside area, 254 = sea, 255 = unclassified. |
| Dominant Leaf Type (DLT) | `HRL_DominantLeafType2018` | Broadleaved (warm/yellow hue) vs coniferous (cool/green hue) pixels. |
| Forest Type (FTY) | `HRL_ForestType_2018` | FAO-based forest classification raster. |

**Endpoint pattern** (all three follow the same structure):
```
https://image.discomap.eea.europa.eu/arcgis/rest/services/GioLandPublic/{SERVICE}/ImageServer/exportImage
  ?bbox={bbox-epsg-3857}
  &bboxSR=3857
  &size=256,256
  &format=png
  &transparent=true
  &f=image
```

MapLibre GL fills in `{bbox-epsg-3857}` automatically for each map tile request.

**Point identify endpoint** (used on map click to read the TCD value at a coordinate):
```
https://image.discomap.eea.europa.eu/arcgis/rest/services/GioLandPublic/HRL_TreeCoverDensity_2018/ImageServer/identify
  ?geometry={x},{y}
  &geometryType=esriGeometryPoint
  &sr=3857
  &returnGeometry=false
  &returnCatalogItems=false
  &f=json
```
The coordinate must be in **EPSG:3857** (Web Mercator). The app converts WGS84 → EPSG:3857 inline using the standard formula in `+page.svelte:to3857()`.

**Basemap tiles**: [OpenFreeMap](https://openfreemap.org) (`tiles.openfreemap.org`) — free, open, no API key needed. Uses the **Liberty** style.

**Geocoding / reverse geocoding**: [Nominatim](https://nominatim.openstreetmap.org) (OpenStreetMap). Used for:
- SearchBar autocomplete (`/search` endpoint, viewbox-biased to Europe)
- Reverse lookup on map click (`/reverse` endpoint, zoom=10)

No API keys are required for any of these services. Nominatim requires a valid `User-Agent` header, which is sent as `EuropeTreesMap/1.0`.

### Data exploration and testing

During development, the Copernicus services and identify endpoints were tested and explored in an [Observable notebook](https://observablehq.com/d/d11fa10c09d1dcd4).

---

## Tech stack

| Technology | Version | Role |
|---|---|---|
| [SvelteKit](https://svelte.dev/docs/kit) | ^2.50 | Full-stack web framework (SSR + client routing) |
| [Svelte](https://svelte.dev) | ^5.51 | UI component framework (uses Svelte 5 runes API) |
| [TypeScript](https://www.typescriptlang.org) | ^5.9 | Type safety across all components |
| [Vite](https://vite.dev) | ^7.3 | Build tool and dev server |
| [MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/) | ^5.18 | WebGL map renderer |
| [PMTiles](https://github.com/protomaps/PMTiles) | ^4.4 | Protocol handler for `pmtiles://` tile sources (registered but basemap currently uses HTTPS tiles) |

**Adapter**: `@sveltejs/adapter-auto` — auto-detects deployment platform (Vercel, Netlify, Cloudflare Pages, Node, etc.).

---

## Project structure

```
mcp figma test/
├── src/
│   ├── app.css                  # Global CSS variables and resets
│   ├── app.d.ts                 # SvelteKit ambient types
│   ├── app.html                 # HTML shell
│   ├── lib/
│   │   ├── assets/
│   │   │   └── favicon.svg
│   │   ├── components/
│   │   │   ├── Map.svelte       # MapLibre map, Copernicus raster layers
│   │   │   ├── Intro.svelte     # Full-screen intro overlay
│   │   │   ├── Navbar.svelte    # Top bar (logo + title)
│   │   │   ├── InfoCard.svelte  # Scroll-driven story card (left side)
│   │   │   ├── Legend.svelte    # Color scale legend (bottom-right)
│   │   │   ├── LayerSelector.svelte  # Layer picker strip (exploration mode)
│   │   │   ├── SearchBar.svelte # Nominatim autocomplete search
│   │   │   └── Tooltip.svelte   # Click-to-query popup
│   │   └── index.ts             # $lib barrel (empty by default)
│   └── routes/
│       ├── +layout.svelte       # Imports global CSS + MapLibre CSS
│       └── +page.svelte         # Main page: scroll driver, state, step logic
├── static/
│   └── assets/                  # SVG/PNG assets served at /assets/*
│       ├── vit-logo-bw.svg
│       ├── vit-logo-color.svg
│       ├── vit-logo-color-mobile.svg
│       ├── badam-tree.svg       # Animated loading tree
│       ├── pinus-tree.svg       # Ready-state tree
│       ├── github-mark-white.svg
│       ├── legend-density.png
│       ├── legend-type-density.png
│       ├── legend-thumb.png
│       ├── scroll-arrow.svg
│       └── trees-icon.svg
├── package.json
├── svelte.config.js
├── tsconfig.json
├── vite.config.ts
└── .gitignore
```

---

## Architecture overview

### Scrollytelling pattern

The page uses a **sticky map + scroll driver** pattern:

```
.page (relative)
├── .map-stage (position: sticky; top: 0; height: 100svh)  ← always visible
│   ├── <Map>            WebGL canvas
│   ├── <Navbar>
│   ├── <SearchBar>
│   ├── <InfoCard>
│   ├── <Legend>
│   ├── <LayerSelector>
│   ├── <Tooltip>
│   └── .step-dots
└── .scroll-driver (normal flow, creates scroll height)
    ├── 40vh spacer
    ├── 200vh × 4 steps
    └── 40vh outro
```

A `scroll` event listener in `+page.svelte:onMount` converts `window.scrollY` into:
- `currentStep` (−1 to 3) — which narrative step is active
- `scrollProgress` (0–1) — progress within the current step, used to animate the InfoCard vertically

**Step layout formula:**
```
relativeScroll = scrollY / vh - 0.4   // subtract 40vh intro spacer
step = floor(relativeScroll / 2.0)    // each step = 200vh
progress = (relativeScroll - step * 2.0) / 2.0
```

### Map steps

Each narrative step maps to a camera position defined in `Map.svelte:MAP_STEPS`:

| Step | Center | Zoom | Region |
|---|---|---|---|
| 0 | [10, 52] | 4 | All of Europe |
| 1 | [−4, 37] | 7.2 | Southern Spain |
| 2 | [5, 46] | 5.2 | Western-Central Europe (Lisbon → Ljubljana) |
| 3 | [13, 46.5] | 6.2 | Central Europe (exploration start) |

Camera transitions use `map.easeTo()` with `duration: 2400ms`.

### Layer system

Five named layer modes, each mapping to one of three Copernicus raster sources:

| App layer ID | Copernicus source | What the user sees |
|---|---|---|
| `density` | TCD | Yellow → green density ramp |
| `type-density` | DLT | Combined leaf type + density |
| `broadleaved` | DLT | Same as type-density (future: filter broadleaved only) |
| `conifers` | DLT | Same as type-density (future: filter conifers only) |
| `forest-type` | FTY | FAO forest type classification |

All three raster layers are loaded at map init with `raster-opacity: 0`. Switching layers just changes opacity (0 or 0.85) via `map.setPaintProperty()` — no tile re-fetch.

### PMTiles protocol

The PMTiles protocol handler (`pmtiles://`) is registered at map init even though the current basemap uses standard HTTPS tiles. This makes it trivial to swap in a local `.pmtiles` file as a source in future without any extra setup.

### Cross-component communication

The map's `flyTo` action is triggered by SearchBar via a **browser CustomEvent**:

```
SearchBar → document.dispatchEvent('flyto', { lat, lng })
         ← Map listens: document.addEventListener('flyto', ...)
```

This avoids prop-drilling through the page component for what is essentially a fire-and-forget command.

---

## Component API reference

### `Map.svelte`

| Prop | Type | Default | Description |
|---|---|---|---|
| `step` | `number` | `0` | Index into `MAP_STEPS`; triggers `easeTo` on change |
| `activeLayer` | `Layer` | `'density'` | Which Copernicus layer to display |
| `onload` | `() => void` | — | Fires once when the MapLibre map finishes loading |
| `onmapclick` | `(data: { lng, lat }) => void` | — | Fires on every map click |

Listens to the `flyto` CustomEvent on `document` to navigate to search results.

### `Intro.svelte`

| Prop | Type | Default | Description |
|---|---|---|---|
| `visible` | `boolean` | `true` | Shows/hides the overlay (CSS opacity + visibility) |
| `loading` | `boolean` | `true` | Shows animated badam tree (loading) vs pinus tree (ready) |

### `Navbar.svelte`

| Prop | Type | Default | Description |
|---|---|---|---|
| `visible` | `boolean` | `false` | Slides the bar in from the top |

### `InfoCard.svelte`

| Prop | Type | Default | Description |
|---|---|---|---|
| `text` | `string` | `''` | Story text to display |
| `visible` | `boolean` | `true` | CSS opacity toggle |
| `icon` | `string` | `''` | Optional image URL shown above text |
| `exploration` | `boolean` | `false` | Switches to compact bottom-anchored layout with no scroll animation |
| `progress` | `number` | `0` | 0–1 scroll progress; drives the vertical scroll animation |

**Animation formula**: `translateY(calc(-50% + (1 - progress * 2) * 100vh))`
- At `progress=0`: card starts off-screen below (`+100vh`)
- At `progress=0.5`: card is centred (`0vh`)
- At `progress=1`: card exits off-screen above (`-100vh`)

### `Legend.svelte`

| Prop | Type | Default | Description |
|---|---|---|---|
| `layer` | `'density' \| 'type-density' \| 'broadleaved'` | `'density'` | Determines which legend items to show |
| `visible` | `boolean` | `true` | CSS opacity + translate toggle |
| `activeIndex` | `number` | `1` | Highlights the nth item (1-based) when multiple items shown |

### `LayerSelector.svelte`

| Prop | Type | Default | Description |
|---|---|---|---|
| `visible` | `boolean` | `false` | Shows/hides the selector strip |
| `activeLayer` | `Layer` | `'broadleaved'` | Highlights the currently active layer button |
| `onchange` | `(layer: Layer) => void` | — | Called when user selects a different layer |

### `SearchBar.svelte`

| Prop | Type | Default | Description |
|---|---|---|---|
| `visible` | `boolean` | `true` | CSS opacity + translate toggle |
| `onflyto` | `(coords: { lat, lng, name }) => void` | — | Called when user selects a result; parent dispatches `flyto` CustomEvent |
| `dark` | `boolean` | `false` | Applies a more opaque white background (used in exploration mode) |
| `exploration` | `boolean` | `false` | Centres the search bar horizontally and makes it wider |

Debounces input by 400 ms. Hits Nominatim with a European viewbox bias (`-25,71,45,34`) and `limit=5`.

### `Tooltip.svelte`

| Prop | Type | Default | Description |
|---|---|---|---|
| `data` | `TooltipData \| null` | `null` | If null, tooltip is hidden |
| `onclose` | `() => void` | — | Called when user clicks the close button or clicks the map again |

`TooltipData` shape:
```ts
{
  name: string;      // Place name from Nominatim reverse geocoding
  density: number;   // 0–100 from Copernicus TCD; -1 while loading
  loading?: boolean; // Shows skeleton shimmer when true
  x: number;         // clientX of the originating click
  y: number;         // clientY of the originating click
}
```

The tooltip positions itself above the click point, clamped to the viewport edges.

---

## CSS design system

All design tokens live in `src/app.css` as CSS custom properties:

```css
/* Typography */
--font: 'Helvetica Neue', Helvetica, Arial, sans-serif;

/* Colors */
--color-bg-intro: #0a1208;          /* Near-black green intro background */
--color-white: #ffffff;
--color-glass-heavy: rgba(255,255,255,0.75);
--color-glass-light: rgba(255,255,255,0.35);
--color-text-dark: #333333;
--color-text-medium: #505050;
--color-text-light: #888888;

/* Forest data colors */
--color-dense: #005f00;             /* Dense forest green */
--color-sparse: #ffec81;            /* Sparse coverage yellow */
--color-broadleaved: #c7b447;       /* Broadleaved warm yellow */
--color-conifer: #07523f;           /* Conifer dark green */

/* Shadows */
--shadow-card: 0px 2px 32px 0px rgba(0,0,0,0.05);
--shadow-search: 0px 1px 16px 0px rgba(0,0,0,0.2);
--shadow-legend: 0px -1px 16px 0px rgba(0,0,0,0.1);
--shadow-tooltip: 0px 1px 8px 0px rgba(0,0,0,0.15);

/* Blur (glassmorphism) */
--blur-heavy: blur(24px);
--blur-medium: blur(8px);
--blur-light: blur(16px);

/* Border radius */
--radius-card: 12px;
--radius-pill: 999px;
```

Components use `backdrop-filter` for the glassmorphism effect on cards, legend, search bar, and tooltip. All UI transitions are CSS-only (opacity + transform), keeping JS minimal.

---

## Setup and development

### Prerequisites

- **Node.js** 18+ (the `.npmrc` sets `engine-strict=true`)
- **npm** (included with Node)

### Install

```sh
npm install
```

### Run the dev server

```sh
npm run dev
```

Open [http://localhost:5173](http://localhost:5173). The map loads live Copernicus tiles from EEA — an internet connection is required.

```sh
npm run dev -- --open   # open browser automatically
```

### Type-check

```sh
npm run check           # one-time
npm run check:watch     # watch mode
```

### Build for production

```sh
npm run build
npm run preview         # preview the production build locally
```

### Deploying

The project uses `@sveltejs/adapter-auto`, which auto-detects:
- **Vercel** — deploy by connecting the repo; zero config
- **Netlify** — same
- **Cloudflare Pages** — same
- **Node server** — switch adapter to `@sveltejs/adapter-node`

See the [SvelteKit adapters docs](https://svelte.dev/docs/kit/adapters) for details.

---

## Key implementation notes for new developers

**Svelte 5 runes**: This project uses the new Svelte 5 reactivity API. If you are familiar with Svelte 4, note:
- `$state(...)` replaces `let x = ...` for reactive variables
- `$derived(...)` replaces `$: x = ...`
- `$props()` replaces `export let`
- `$effect(...)` replaces `$: { ... }` for side effects

**`svh` units**: The sticky map uses `height: 100svh` (small viewport height). On mobile, this avoids the browser chrome (address bar) causing layout overflow. Do not change to `100vh` without testing on iOS Safari.

**Scroll architecture**: Never put meaningful content inside `.scroll-driver` — it is invisible (`pointer-events: none`) and only creates scroll height. All interactive UI lives inside `.map-stage`.

**Copernicus tiles are raster (PNG)**: They are not vector tiles. Each 256×256 tile is a server-rendered image. This means you cannot style individual features — color coding is baked into the server response. The `transparent=true` parameter makes non-forest pixels transparent so the basemap shows through.

**EPSG:3857 conversion**: The Copernicus identify endpoint requires EPSG:3857 coordinates, not WGS84. The `to3857` function in `+page.svelte` does this inline. Do not use lat/lng directly with that endpoint.

**No environment variables**: All API endpoints are public and require no keys. The only "secret" is the Nominatim `User-Agent` header (`EuropeTreesMap/1.0`), which is included in each fetch call.

**MapLibre is loaded dynamically**: `import('maplibre-gl')` is called inside `onMount` with a dynamic import. This keeps MapLibre out of the SSR bundle (it requires `window`). PMTiles is loaded in the same `Promise.all` so the protocol handler is ready before any `pmtiles://` URL is resolved.

---

## External API rate limits

| Service | Limit | Notes |
|---|---|---|
| EEA DiscoMap (Copernicus tiles) | No documented limit | Public, but don't hammer it with automated tools |
| EEA DiscoMap (identify endpoint) | No documented limit | One request per map click |
| Nominatim search | 1 req/sec per IP | The 400 ms debounce keeps the app well within this |
| Nominatim reverse | 1 req/sec per IP | One request per map click |
| OpenFreeMap tiles | No documented limit | Free service, be respectful |

---

## Attribution

- Forest data: **© Copernicus Land Monitoring Service / European Environment Agency**, 2018
- Basemap: **© OpenFreeMap contributors**, **© OpenMapTiles**, **© OpenStreetMap contributors**
- Geocoding: **© OpenStreetMap contributors** (Nominatim)
- Built by **ViT**
