<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { browser } from '$app/environment';
  import { PUBLIC_MAPBOX_TOKEN } from '$env/static/public';

  type Layer = 'density' | 'type-density' | 'broadleaved' | 'conifers' | 'forest-type';

  interface Props {
    step?: number;
    activeLayer?: Layer;
    onload?: () => void;
    onmapclick?: (data: { lng: number; lat: number }) => void;
    scrollZoomEnabled?: boolean;
  }

  let { step = 0, activeLayer = 'density', onload, onmapclick, scrollZoomEnabled = false }: Props = $props();

  let mapContainer: HTMLDivElement;
  let map: any;
  let isLoaded = $state(false);
  let cleanupFlyTo: (() => void) | undefined;
  let cleanupZoom: (() => void) | undefined;

  // Exact map positions from the Observable notebook @694
  const MAP_STEPS = [
    { center: [12.942, 53.300] as [number, number], zoom: 3.25 },   // 0 Europe overview
    { center: [-4.761, 36.636] as [number, number], zoom: 8.08 },   // 1 South of Spain
    { center: [19.227, 61.420] as [number, number], zoom: 5.36 },   // 2 Scandinavia
    { center: [12.942, 53.300] as [number, number], zoom: 3.25 },   // 3 Europe forest types
    { center: [7.296,  44.736] as [number, number], zoom: 4.21 },   // 4 Southern Europe broadleaved
    { center: [8.874,  57.116] as [number, number], zoom: 4.27 },   // 5 Northern Europe coniferous
    { center: [12.942, 53.300] as [number, number], zoom: 3.25 },   // 6 Europe combined
    { center: [-8.972, 38.697] as [number, number], zoom: 8.5  },   // 7 Lisbon
    { center: [14.362, 46.042] as [number, number], zoom: 8.5  },   // 8 Ljubljana
    { center: [12.942, 53.300] as [number, number], zoom: 3.25 },   // 9 Explore
  ];

  // ── WMS tile URL builder (GeoVille, Copernicus HRL 2021) ─────────────────────
  // GeoServer supports SLD_BODY for custom per-request styling. This lets us:
  //   • Filter DLT by pixel value (1=broadleaved, 2=coniferous)
  //   • Apply the notebook colour palette directly at the server
  // Confirmed: CORS access-control-allow-origin: * ✓  SLD_BODY returns image/png ✓
  const GV = 'https://geoserver.vlcc.geoville.com/geoserver/ows';
  const TILE_BASE = `service=WMS&version=1.3.0&request=GetMap&format=image/png` +
    `&transparent=true&width=256&height=256&crs=EPSG:3857&bbox={bbox-epsg-3857}`;

  function wmsUrl(layerName: string, sld: string): string {
    return `${GV}?${TILE_BASE}&layers=${layerName}&SLD_BODY=${encodeURIComponent(sld)}`;
  }

  // SLD bodies — inline GeoServer RasterSymbolizer styles
  // DLT values: 1 = broadleaved, 2 = coniferous, everything else = background (transparent)
  const SLD_DLT_BRO =
    `<StyledLayerDescriptor version="1.0.0" xmlns="http://www.opengis.net/sld">` +
    `<NamedLayer><Name>HRL_TCF:DLT_S2021</Name><UserStyle><FeatureTypeStyle><Rule>` +
    `<RasterSymbolizer><ColorMap type="values">` +
    `<ColorMapEntry color="#c7b447" quantity="1" opacity="1.0"/>` +
    `</ColorMap></RasterSymbolizer></Rule></FeatureTypeStyle></UserStyle></NamedLayer>` +
    `</StyledLayerDescriptor>`;

  const SLD_DLT_CON =
    `<StyledLayerDescriptor version="1.0.0" xmlns="http://www.opengis.net/sld">` +
    `<NamedLayer><Name>HRL_TCF:DLT_S2021</Name><UserStyle><FeatureTypeStyle><Rule>` +
    `<RasterSymbolizer><ColorMap type="values">` +
    `<ColorMapEntry color="#07523f" quantity="2" opacity="1.0"/>` +
    `</ColorMap></RasterSymbolizer></Rule></FeatureTypeStyle></UserStyle></NamedLayer>` +
    `</StyledLayerDescriptor>`;

  const SLD_DLT_BOTH =
    `<StyledLayerDescriptor version="1.0.0" xmlns="http://www.opengis.net/sld">` +
    `<NamedLayer><Name>HRL_TCF:DLT_S2021</Name><UserStyle><FeatureTypeStyle><Rule>` +
    `<RasterSymbolizer><ColorMap type="values">` +
    `<ColorMapEntry color="#c7b447" quantity="1" opacity="1.0"/>` +
    `<ColorMapEntry color="#07523f" quantity="2" opacity="1.0"/>` +
    `</ColorMap></RasterSymbolizer></Rule></FeatureTypeStyle></UserStyle></NamedLayer>` +
    `</StyledLayerDescriptor>`;

  // TCD values: 0–100 (% cover). 0 = no trees (transparent). 101+ = special/outside (transparent).
  // Gradient: sparse (low %) → yellow #ffec81, dense (high %) → dark green #005f00
  const SLD_TCD_DENSITY =
    `<StyledLayerDescriptor version="1.0.0" xmlns="http://www.opengis.net/sld">` +
    `<NamedLayer><Name>HRL_TCF:TCD_S2021</Name><UserStyle><FeatureTypeStyle><Rule>` +
    `<RasterSymbolizer><ColorMap type="ramp">` +
    `<ColorMapEntry color="#005f00" quantity="0"   opacity="0.0"/>` +
    `<ColorMapEntry color="#ffec81" quantity="5"   opacity="0.85"/>` +
    `<ColorMapEntry color="#4a9a20" quantity="50"  opacity="0.85"/>` +
    `<ColorMapEntry color="#005f00" quantity="100" opacity="0.85"/>` +
    `<ColorMapEntry color="#005f00" quantity="101" opacity="0.0"/>` +
    `</ColorMap></RasterSymbolizer></Rule></FeatureTypeStyle></UserStyle></NamedLayer>` +
    `</StyledLayerDescriptor>`;

  // TCD as a white opacity mask on top of DLT:
  // sparse areas → more opaque white (DLT colour washed out)
  // dense areas  → transparent (DLT colour shows at full saturation)
  // → darker/richer colour = denser forest
  const SLD_TCD_MASK =
    `<StyledLayerDescriptor version="1.0.0" xmlns="http://www.opengis.net/sld">` +
    `<NamedLayer><Name>HRL_TCF:TCD_S2021</Name><UserStyle><FeatureTypeStyle><Rule>` +
    `<RasterSymbolizer><ColorMap type="ramp">` +
    `<ColorMapEntry color="#ffffff" quantity="0"   opacity="0.0"/>` +
    `<ColorMapEntry color="#ffffff" quantity="1"   opacity="0.85"/>` +
    `<ColorMapEntry color="#ffffff" quantity="50"  opacity="0.4"/>` +
    `<ColorMapEntry color="#ffffff" quantity="100" opacity="0.0"/>` +
    `</ColorMap></RasterSymbolizer></Rule></FeatureTypeStyle></UserStyle></NamedLayer>` +
    `</StyledLayerDescriptor>`;

  // Source/layer registry — DLT layers first, TCD mask last (renders on top of DLT)
  const FOREST_SOURCES: Record<string, string> = {
    'dlt-bro':     wmsUrl('HRL_TCF:DLT_S2021', SLD_DLT_BRO),
    'dlt-con':     wmsUrl('HRL_TCF:DLT_S2021', SLD_DLT_CON),
    'dlt-both':    wmsUrl('HRL_TCF:DLT_S2021', SLD_DLT_BOTH),
    'tcd-density': wmsUrl('HRL_TCF:TCD_S2021', SLD_TCD_DENSITY),
    'tcd-mask':    wmsUrl('HRL_TCF:TCD_S2021', SLD_TCD_MASK),
  };

  // Which layers to show for each app-level mode
  const VISIBLE: Record<Layer, string[]> = {
    density:        ['tcd-density'],
    broadleaved:    ['dlt-bro'],
    conifers:       ['dlt-con'],
    'forest-type':  ['dlt-both'],
    'type-density': ['dlt-both', 'tcd-mask'],  // DLT base + white density mask on top
  };

  function applyLayer(layer: Layer) {
    if (!map || !isLoaded) return;
    const visible = new Set(VISIBLE[layer]);
    for (const id of Object.keys(FOREST_SOURCES)) {
      map.setLayoutProperty(`cop-${id}`, 'visibility', visible.has(id) ? 'visible' : 'none');
    }
  }

  // Fly to narrative step when `step` prop changes
  $effect(() => {
    if (!isLoaded) return;
    const cfg = MAP_STEPS[step] ?? MAP_STEPS[0];
    map?.easeTo({ center: cfg.center, zoom: cfg.zoom, duration: 2400, essential: true });
  });

  // Switch forest layer when `activeLayer` prop changes
  $effect(() => {
    if (!isLoaded) return;
    applyLayer(activeLayer);
  });

  // Enable/disable scroll zoom based on exploration mode
  $effect(() => {
    if (!isLoaded || !map) return;
    scrollZoomEnabled ? map.scrollZoom.enable() : map.scrollZoom.disable();
  });

  onMount(() => {
    if (!browser) return;

    import('mapbox-gl').then(({ default: mapboxgl }) => {
      (mapboxgl as any).accessToken = PUBLIC_MAPBOX_TOKEN;

      map = new (mapboxgl as any).Map({
        container: mapContainer,
        style: 'mapbox://styles/xocasgv/cmcc7qho5040j01sddn2j7yhm',
        projection: { name: 'albers', center: [10, 52], parallels: [40, 65] },
        center: [10, 52],
        zoom: 3.25,
        minZoom: 3,
        maxZoom: 12.9,
        maxBounds: [[-20, 30], [42, 72]] as [[number, number], [number, number]],
        pitch: 0,
        attributionControl: false,
        logoPosition: 'bottom-left',
        scrollZoom: false,
        // cooperativeGestures omitted: it calls preventDefault() on wheel events,
        // which blocks the page scroll that drives the scrollytelling story.
      });

      map.dragRotate.disable();
      map.touchZoomRotate.disableRotation();

      map.addControl(new (mapboxgl as any).AttributionControl({ compact: true }), 'bottom-left');
      map.getCanvas().style.cursor = 'crosshair';
      map.on('mousemove', () => { map.getCanvas().style.cursor = 'crosshair'; });

      map.on('load', () => {
        // Add all forest sources + layers (initially hidden), inserted before 'water'
        // so coastlines and water bodies render on top of forest tiles
        for (const [id, tileUrl] of Object.entries(FOREST_SOURCES)) {
          map.addSource(`cop-${id}`, {
            type: 'raster',
            tiles: [tileUrl],
            tileSize: 256,
            bounds: [-25, 34, 45, 72],
            attribution: '© Copernicus Land Monitoring Service / GeoVille 2021'
          });
          map.addLayer({
            id: `cop-${id}`,
            type: 'raster',
            source: `cop-${id}`,
            layout: { visibility: 'none' },
            paint: { 'raster-opacity': 0.9 }
          }, 'water');  // inserts under water/coastlines
        }

        // Suppress expected WMS tile errors (out-of-bounds, server hiccups)
        map.on('error', (e) => {
          if (e.error?.status === 503 || e.error?.status === 404) return;
          console.warn('Map error:', e.error);
        });

        // Set isLoaded BEFORE applyLayer so the guard inside it passes
        isLoaded = true;
        applyLayer(activeLayer);
        onload?.();

        const handleZoomIn  = () => map?.zoomIn({ duration: 300 });
        const handleZoomOut = () => map?.zoomOut({ duration: 300 });
        document.addEventListener('map:zoomin',  handleZoomIn);
        document.addEventListener('map:zoomout', handleZoomOut);
        cleanupZoom = () => {
          document.removeEventListener('map:zoomin',  handleZoomIn);
          document.removeEventListener('map:zoomout', handleZoomOut);
        };
      });

      map.on('click', (e: any) => {
        onmapclick?.({ lng: e.lngLat.lng, lat: e.lngLat.lat });
      });

      const handleFlyTo = (e: Event) => {
        const { lat, lng } = (e as CustomEvent).detail;
        map.flyTo({ center: [lng, lat], zoom: 11, duration: 2000, essential: true });
      };
      document.addEventListener('flyto', handleFlyTo);
      cleanupFlyTo = () => document.removeEventListener('flyto', handleFlyTo);
    });
  });

  onDestroy(() => {
    cleanupFlyTo?.();
    cleanupZoom?.();
    map?.remove();
  });
</script>

<div bind:this={mapContainer} class="map-container"></div>

<style>
  .map-container {
    width: 100%;
    height: 100%;
    position: absolute;
    inset: 0;
  }

  :global(.mapboxgl-ctrl-bottom-left) {
    z-index: 5;
    pointer-events: auto;
  }

  :global(.mapboxgl-ctrl-attrib) {
    font-family: var(--font);
    font-size: 10px;
    background: rgba(255,255,255,0.5) !important;
    backdrop-filter: blur(4px);
    border-radius: 4px !important;
  }

  :global(.mapboxgl-canvas) {
    outline: none;
  }
</style>
