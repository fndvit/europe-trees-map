<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { browser } from '$app/environment';

  type Layer = 'density' | 'type-density' | 'broadleaved' | 'conifers' | 'forest-type';

  interface Props {
    step?: number;
    activeLayer?: Layer;
    onload?: () => void;
    onmapclick?: (data: { lng: number; lat: number }) => void;
  }

  let { step = 0, activeLayer = 'density', onload, onmapclick }: Props = $props();

  let mapContainer: HTMLDivElement;
  let map: any;
  let isLoaded = $state(false);
  let cleanupFlyTo: (() => void) | undefined;

  const MAP_STEPS = [
    { center: [10, 52] as [number, number], zoom: 4 },
    { center: [-4, 37] as [number, number], zoom: 7.2 },
    { center: [5, 46] as [number, number], zoom: 5.2 },
    { center: [13, 46.5] as [number, number], zoom: 6.2 }
  ];

  // ── Real forest data: Copernicus High Resolution Layers via EEA ImageServer ──
  // Source: Copernicus Land Monitoring Service (CLMS), © European Union, 2018
  // Served via EEA DiscoMap ArcGIS ImageServer exportImage endpoint.
  // MapLibre fills in {bbox-epsg-3857} with the tile's EPSG:3857 bounding box.
  // transparent=true makes non-forest pixels transparent so basemap shows through.
  const EEA_BASE = 'https://image.discomap.eea.europa.eu/arcgis/rest/services/GioLandPublic';
  const EXPORT   = (svc: string) =>
    `${EEA_BASE}/${svc}/ImageServer/exportImage?bbox={bbox-epsg-3857}&bboxSR=3857&size=256,256&format=png&transparent=true&f=image`;

  const FOREST_SOURCES = {
    // Tree Cover Density: 0–100 % per 10 m pixel (yellow = sparse, green = dense)
    tcd: EXPORT('HRL_TreeCoverDensity_2018'),
    // Dominant Leaf Type: broadleaved (warm hue) vs coniferous (cool hue)
    dlt: EXPORT('HRL_DominantLeafType2018'),
    // Forest Type: FAO-based forest classification
    fty: EXPORT('HRL_ForestType_2018')
  } as const;

  // Which Copernicus layer to show for each app-level layer mode
  const LAYER_SOURCE: Record<Layer, keyof typeof FOREST_SOURCES> = {
    density:        'tcd',
    'type-density': 'dlt',
    broadleaved:    'dlt',
    conifers:       'dlt',
    'forest-type':  'fty'
  };

  function applyLayer(layer: Layer) {
    if (!map || !isLoaded) return;
    const active = LAYER_SOURCE[layer];
    (Object.keys(FOREST_SOURCES) as Array<keyof typeof FOREST_SOURCES>).forEach(key => {
      map.setPaintProperty(`cop-${key}`, 'raster-opacity', key === active ? 0.85 : 0);
    });
  }

  // Fly to narrative step when `step` changes
  $effect(() => {
    if (!isLoaded) return;
    const cfg = MAP_STEPS[step] ?? MAP_STEPS[0];
    map?.easeTo({ center: cfg.center, zoom: cfg.zoom, duration: 2400, essential: true });
  });

  // Switch forest data layer when `activeLayer` changes
  $effect(() => {
    if (!isLoaded) return;
    applyLayer(activeLayer);
  });

  onMount(() => {
    if (!browser) return;

    // Load MapLibre and PMTiles together so PMTiles protocol is ready before
    // any pmtiles:// URLs are resolved.
    Promise.all([
      import('maplibre-gl'),
      import('pmtiles')
    ]).then(([{ default: maplibregl }, { Protocol }]) => {

      // Register the PMTiles protocol handler — enables pmtiles:// source URLs
      const pmtilesProtocol = new Protocol();
      maplibregl.addProtocol('pmtiles', pmtilesProtocol.tile.bind(pmtilesProtocol));

      map = new maplibregl.Map({
        container: mapContainer,
        style: 'https://tiles.openfreemap.org/styles/liberty',
        center: [10, 52],
        zoom: 4,
        pitch: 0,
        attributionControl: false,
        logoPosition: 'bottom-left',
        scrollZoom: false
      });

      map.addControl(new maplibregl.AttributionControl({ compact: true }), 'bottom-left');

      map.getCanvas().style.cursor = 'crosshair';
      map.on('mousemove', () => { map.getCanvas().style.cursor = 'crosshair'; });

      map.on('load', () => {
        // Add each Copernicus source + an initially-hidden raster layer
        (Object.entries(FOREST_SOURCES) as [keyof typeof FOREST_SOURCES, string][])
          .forEach(([key, url]) => {
            map.addSource(`cop-${key}`, {
              type: 'raster',
              tiles: [url],
              tileSize: 256,
              attribution: '© Copernicus Land Monitoring Service / EEA'
            });
            map.addLayer({
              id: `cop-${key}`,
              type: 'raster',
              source: `cop-${key}`,
              paint: { 'raster-opacity': 0 }
            });
          });

        // Reveal the active layer
        applyLayer(activeLayer);

        isLoaded = true;
        onload?.();
      });

      map.on('click', (e: any) => {
        onmapclick?.({ lng: e.lngLat.lng, lat: e.lngLat.lat });
      });

      // External flyTo command fired by SearchBar
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

  :global(.maplibregl-ctrl-bottom-left) {
    z-index: 5;
    pointer-events: auto;
  }

  :global(.maplibregl-ctrl-attrib) {
    font-family: var(--font);
    font-size: 10px;
    background: rgba(255,255,255,0.5) !important;
    backdrop-filter: blur(4px);
    border-radius: 4px !important;
  }

  :global(.maplibregl-canvas) {
    outline: none;
  }
</style>
