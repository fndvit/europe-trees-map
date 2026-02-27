<script lang="ts">
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';
  import Map from '$lib/components/Map.svelte';
  import Intro from '$lib/components/Intro.svelte';
  import Navbar from '$lib/components/Navbar.svelte';
  import InfoCard from '$lib/components/InfoCard.svelte';
  import SearchBar from '$lib/components/SearchBar.svelte';
  import LayerSelector from '$lib/components/LayerSelector.svelte';
  import StoryLayerCard from '$lib/components/StoryLayerCard.svelte';
  import Tooltip from '$lib/components/Tooltip.svelte';

  type Layer = 'density' | 'type-density' | 'broadleaved' | 'conifers' | 'forest-type';

  interface TooltipData {
    name: string;
    density: number;     // 0–100 from EEA TCD identify
    leafType: number;    // 1=broadleaved, 2=coniferous, 0=unknown
    conifers: number;    // ha from DLT computeHistograms (-1 = unavailable)
    broadleaves: number; // ha from DLT computeHistograms (-1 = unavailable)
    loading?: boolean;
    x: number;
    y: number;
  }

  interface ScrollStep {
    text: string;
    layer: Layer;
    mapStep: number;
    isExploration?: boolean;
    isSplit?: boolean;
    splitSteps?: [number, number]; // [leftMapStep, rightMapStep]
  }

  // ─── Story steps — matching Observable notebook @694 ────────────────────────
  const STEPS: ScrollStep[] = [
    {
      text: "Europe's forests vary wildly. This map reveals where trees thrive —and where they don't— using high-resolution satellite data.",
      layer: 'density',
      mapStep: 0
    },
    {
      text: "Zooming into southern Spain, you can see where tree cover density drops. Dry climate, land use, and desertification mean many areas fall below 30% coverage —shown here in yellowish shades.",
      layer: 'density',
      mapStep: 1
    },
    {
      text: "In contrast, Scandinavia is blanketed in darker green. Sweden and Finland have among the highest tree densities in Europe, with many regions over 80% coverage.",
      layer: 'density',
      mapStep: 2
    },
    {
      text: "But tree density is only one part of the story —and the data. We can also split trees by type —broadleaved and coniferous.",
      layer: 'forest-type',
      mapStep: 3
    },
    {
      text: "In the south — Portugal, Italy, the Balkans — broadleaved trees dominate: oaks, birches, beeches, and chestnuts, rooted in temperate and Mediterranean soils.",
      layer: 'broadleaved',
      mapStep: 4
    },
    {
      text: "From Germany through the Baltic to the Nordic countries, the landscape is mainly coniferous forests: pines, spruces, cypresses, and firs, which are well-adapted to the colder, nutrient-poor soils.",
      layer: 'conifers',
      mapStep: 5
    },
    {
      text: "This view combines both tree cover density and the dominant leaf type. Darker shades of the same color mean denser coverage, while the hue still shows the forest type.",
      layer: 'type-density',
      mapStep: 6
    },
    {
      text: "Lisbon's forests are mostly broadleaved, but the density is low. Contrast that with Ljubljana — surrounded by one of the highest forest densities in Europe.",
      layer: 'type-density',
      mapStep: 7,
      isSplit: true,
      splitSteps: [7, 8]
    },
    {
      text: "Curious what grows near you? Use the map to explore forest patterns across Europe —by density, by type, and by place.",
      layer: 'type-density',
      mapStep: 9,
      isExploration: true
    }
  ];

  // ─── State ───────────────────────────────────────────────────────────────────
  let mapLoaded        = $state(false);
  let introVisible     = $state(true);
  let currentStep      = $state(-1);
  let activeLayer      = $state<Layer>('density');
  let mapStep          = $state(0);
  let tooltipData      = $state<TooltipData | null>(null);
  let lastClickX       = $state(0);
  let lastClickY       = $state(0);
  let scrollProgress   = $state(0); // 0–1 within the current step
  let explorationActive = $state(false);

  // ─── Derived ─────────────────────────────────────────────────────────────────
  let isExploration    = $derived(currentStep === STEPS.length - 1);
  let currentStepData  = $derived(currentStep >= 0 ? STEPS[currentStep] : null);
  let isSplit          = $derived(currentStepData?.isSplit === true);
  let splitLeftStep    = $derived(currentStepData?.splitSteps?.[0] ?? mapStep);
  let splitRightStep   = $derived(currentStepData?.splitSteps?.[1] ?? mapStep);
  let showUI           = $derived(!introVisible && currentStep >= 0);
  let storyLayers = $derived<Layer[]>(
    isSplit ? ['type-density'] : [activeLayer]
  );

  // ─── Handlers ────────────────────────────────────────────────────────────────
  function handleMapLoad() {
    mapLoaded = true;
  }

  function handleFlyTo(coords: { lat: number; lng: number }) {
    document.dispatchEvent(new CustomEvent('flyto', { detail: coords }));
  }

  function handleLayerChange(layer: Layer) {
    activeLayer = layer;
  }

  function goToTop() {
    explorationActive = false;
    document.body.style.overflow = '';
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }

  function zoomIn()  { document.dispatchEvent(new CustomEvent('map:zoomin')); }
  function zoomOut() { document.dispatchEvent(new CustomEvent('map:zoomout')); }

  // Convert WGS84 lng/lat → EPSG:3857 (needed for EEA ImageServer identify)
  function to3857(lng: number, lat: number): [number, number] {
    const x = lng * 20037508.34 / 180;
    const y = Math.log(Math.tan((90 + lat) * Math.PI / 360)) / (Math.PI / 180) * 20037508.34 / 180;
    return [Math.round(x), Math.round(y)];
  }

  async function handleMapClick(data: { lng: number; lat: number }) {
    if (tooltipData) { tooltipData = null; return; }

    tooltipData = { name: '…', density: -1, leafType: 0, conifers: -1, broadleaves: -1, loading: true, x: lastClickX, y: lastClickY };

    const [x, y] = to3857(data.lng, data.lat);
    const EEA_BASE = 'https://image.discomap.eea.europa.eu/arcgis/rest/services/GioLandPublic';
    const IDENTIFY = (svc: string) =>
      `${EEA_BASE}/${svc}/ImageServer/identify?geometry=${x},${y}&geometryType=esriGeometryPoint&sr=3857&returnGeometry=false&returnCatalogItems=false&f=json`;

    const TCD_URL = IDENTIFY('HRL_TreeCoverDensity_2018');
    const DLT_URL = IDENTIFY('HRL_DominantLeafType2018');
    const NOM_URL = `https://nominatim.openstreetmap.org/reverse?lat=${data.lat}&lon=${data.lng}&format=json&zoom=10&polygon_geojson=1`;

    let density = 0;
    let leafType = 0;
    let name = `${data.lat.toFixed(2)}°, ${data.lng.toFixed(2)}°`;
    let conifers = -1;
    let broadleaves = -1;

    try {
      const [tcdRes, dltRes, nomRes] = await Promise.all([
        fetch(TCD_URL),
        fetch(DLT_URL),
        fetch(NOM_URL, { headers: { 'Accept-Language': 'en', 'User-Agent': 'EuropeTreesMap/1.0' } })
      ]);

      const [tcd, dlt, nom] = await Promise.all([tcdRes.json(), dltRes.json(), nomRes.json()]);

      // TCD: 0–100 are valid density %; special codes mean outside/sea/unclassified
      const rawTcd = parseInt(tcd.value ?? '-1', 10);
      density = rawTcd >= 0 && rawTcd <= 100 ? rawTcd : 0;

      // DLT: 1=broadleaved, 2=coniferous; other values = no data
      const rawDlt = parseInt(dlt.value ?? '0', 10);
      leafType = rawDlt === 1 || rawDlt === 2 ? rawDlt : 0;

      const a = nom.address ?? {};
      name = a.city ?? a.town ?? a.village ?? a.municipality
              ?? a.county ?? a.state ?? nom.display_name?.split(',')[0]
              ?? `${data.lat.toFixed(2)}°, ${data.lng.toFixed(2)}°`;

      // Fetch histogram data from EEA DLT using the Nominatim polygon boundary.
      // ArcGIS ImageServer expects ArcGIS JSON (rings[]), not raw GeoJSON.
      // Each pixel = 10m × 10m = 0.01 ha; bins[1]=broadleaved, bins[2]=coniferous
      if (nom.geojson) {
        try {
          const gj = nom.geojson;
          let rings: number[][][] = [];
          if (gj.type === 'Polygon') {
            rings = gj.coordinates;
          } else if (gj.type === 'MultiPolygon') {
            rings = gj.coordinates.flatMap((p: number[][][]) => p);
          }
          if (rings.length > 0) {
            const arcGeom = { rings, spatialReference: { wkid: 4326 } };
            const histUrl = `${EEA_BASE}/HRL_DominantLeafType2018/ImageServer/computeHistograms`
              + `?geometry=${encodeURIComponent(JSON.stringify(arcGeom))}`
              + `&geometryType=esriGeometryPolygon&sr=4326&f=json`;
            const histRes = await fetch(histUrl);
            const hist = await histRes.json();
            const bins = hist?.histograms?.[0]?.counts ?? [];
            broadleaves = Math.round((bins[1] ?? 0) * 0.01);
            conifers    = Math.round((bins[2] ?? 0) * 0.01);
          }
        } catch {
          // histogram unavailable — keep -1 to show "—" in tooltip
        }
      }
    } catch {
      // main fetches failed — show coordinate fallback with no stats
    }

    tooltipData = { name, density, leafType, conifers, broadleaves, loading: false, x: lastClickX, y: lastClickY };
  }

  // ─── Scroll-driven step detection ────────────────────────────────────────────
  onMount(() => {
    if (!browser) return;

    const INTRO_SPACER = 0.4;   // fraction of vh
    const STEP_HEIGHT  = 2.0;   // fraction of vh per step

    function onScroll() {
      const scrollY = window.scrollY;
      const vh      = window.innerHeight;

      introVisible = scrollY <= 10;

      const relativeScroll = scrollY / vh - INTRO_SPACER;
      const raw = Math.floor(relativeScroll / STEP_HEIGHT);
      const next = Math.min(Math.max(raw, -1), STEPS.length - 1);

      scrollProgress = next >= 0
        ? Math.max(0, Math.min(1, (relativeScroll - next * STEP_HEIGHT) / STEP_HEIGHT))
        : 0;

      const stepIsExploration = next >= 0 && STEPS[next].isExploration === true;

      if (stepIsExploration && scrollProgress > 0.75 && !explorationActive) {
        explorationActive = true;
        document.body.style.overflow = 'hidden';
      }

      if (!stepIsExploration && explorationActive) {
        explorationActive = false;
        document.body.style.overflow = '';
      }

      if (next !== currentStep) {
        currentStep = next;
        if (next >= 0) {
          activeLayer = STEPS[next].layer as Layer;
          mapStep     = STEPS[next].mapStep;
          tooltipData = null;
        }
      }
    }

    window.addEventListener('scroll', onScroll, { passive: true });
    onScroll();
    return () => window.removeEventListener('scroll', onScroll);
  });
</script>

<svelte:head>
  <title>The Most Detailed Map of Europe's Trees</title>
  <meta name="description" content="An interactive scrollytelling map of Europe's forest cover density and types." />
</svelte:head>

<!-- ── Intro overlay (fixed, fades on first scroll) ──────────────────────── -->
<Intro visible={introVisible} loading={!mapLoaded} />

<!-- ── Page ──────────────────────────────────────────────────────────────── -->
<div class="page">

  <!-- Sticky map canvas: stays pinned to viewport while user scrolls -->
  <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_noninteractive_element_interactions -->
  <div
    class="map-stage"
    role="application"
    aria-label="Interactive forest map"
    onclick={(e) => { lastClickX = e.clientX; lastClickY = e.clientY; }}
    onkeydown={() => {}}
  >
    <!-- Single map (always mounted, fades out in split mode) -->
    <div class="single-map" class:faded={isSplit}>
      <Map
        step={mapStep}
        activeLayer={activeLayer}
        onload={handleMapLoad}
        onmapclick={handleMapClick}
        scrollZoomEnabled={explorationActive}
      />
    </div>

    <!-- Split stage (always mounted, fades in for split step) -->
    <div class="split-stage" class:visible={isSplit}>
      <div class="split-half">
        <Map step={splitLeftStep} activeLayer="type-density" />
      </div>
      <div class="split-divider"></div>
      <div class="split-half">
        <Map step={splitRightStep} activeLayer="type-density" />
      </div>
    </div>

    <Navbar visible={showUI} />

    <SearchBar
      visible={explorationActive && showUI}
      onflyto={handleFlyTo}
      dark={explorationActive}
      exploration={explorationActive}
    />

    {#if currentStepData}
      <InfoCard
        text={currentStepData.text}
        icon="/assets/trees-icon.svg"
        visible={showUI && !explorationActive}
        progress={scrollProgress}
      />
    {/if}

    <!-- Story card: compact, bottom right, informational -->
    <StoryLayerCard
      visible={showUI && !isExploration}
      layers={storyLayers}
    />

    <!-- Exploration bar: full width, all 5 layers, interactive -->
    <LayerSelector
      visible={explorationActive && showUI}
      activeLayer={activeLayer}
      onchange={handleLayerChange}
    />

    <!-- Map controls: zoom in/out + go to top (visible in exploration) -->
    <div class="map-controls" class:visible={explorationActive && showUI} aria-label="Map controls">
      <button class="map-btn" onclick={zoomIn} title="Zoom in" aria-label="Zoom in">+</button>
      <button class="map-btn" onclick={zoomOut} title="Zoom out" aria-label="Zoom out">−</button>
    </div>

    <button class="back-to-top-btn" class:visible={explorationActive && showUI} onclick={goToTop} aria-label="Back to the top">
      <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><polyline points="18 15 12 9 6 15"/></svg>
      Back to the top
    </button>

    <Tooltip data={tooltipData} onclose={() => (tooltipData = null)} />

    <!-- Step progress dots -->
    <div class="step-dots" class:visible={showUI && !explorationActive} aria-hidden="true">
      {#each STEPS as _step, i}
        <div class="dot" class:active={i === currentStep}></div>
      {/each}
    </div>
  </div>

  <!-- Scroll sections: invisible, just create scroll height to drive the story -->
  <div class="scroll-driver" aria-hidden="true">
    <div style="height: 40vh"></div>
    {#each STEPS as _step}
      <div class="scroll-step"></div>
    {/each}
    <div style="height: 40vh"></div>
  </div>

  <footer class="site-footer" class:visible={explorationActive}>
    <a href="https://www.fundaciovit.org/" target="_blank" rel="noopener noreferrer" aria-label="Fundació ViT">
      <img src="/assets/vit-logo-bw.svg" alt="ViT" width="48" />
    </a>
  </footer>

</div>

<style>
  .page {
    position: relative;
    width: 100%;
  }

  .map-stage {
    position: sticky;
    top: 0;
    width: 100%;
    height: 100svh;
    overflow: hidden;
    z-index: 1;
  }

  .scroll-driver {
    position: relative;
    z-index: 0;
    pointer-events: none;
  }

  .scroll-step {
    height: 200vh;
  }

  .step-dots {
    position: absolute;
    right: 20px;
    top: 50%;
    transform: translateY(-50%);
    display: flex;
    flex-direction: column;
    gap: 10px;
    z-index: 20;
    opacity: 0;
    transition: opacity 0.4s ease;
    pointer-events: none;
  }

  .step-dots.visible {
    opacity: 1;
  }

  .dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.35);
    transition: background 0.3s ease, transform 0.3s ease;
  }

  .dot.active {
    background: rgba(255, 255, 255, 0.9);
    transform: scale(1.5);
  }

  @media (max-width: 600px) {
    .step-dots {
      right: auto;
      top: auto;
      bottom: 80px;
      left: 50%;
      transform: translateX(-50%);
      flex-direction: row;
    }

    .step-dots.visible {
      opacity: 1;
    }
  }

  /* Single map wrapper — full viewport, fades out in split mode */
  .single-map {
    position: absolute;
    inset: 0;
    opacity: 1;
    transition: opacity 0.5s ease;
  }
  .single-map.faded {
    opacity: 0;
    pointer-events: none;
  }

  /* Split comparison layout */
  .split-stage {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: row;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.5s ease;
  }
  .split-stage.visible {
    opacity: 1;
    pointer-events: auto;
  }

  .split-half {
    position: relative;
    flex: 1;
    min-width: 0;
    height: 100%;
    overflow: hidden;
  }

  .split-divider {
    width: 2px;
    flex-shrink: 0;
    background: rgba(255, 255, 255, 0.55);
    z-index: 10;
  }

  @media (max-width: 600px) {
    .split-stage {
      flex-direction: column;
    }
    .split-divider {
      width: auto;
      height: 2px;
    }
  }

  /* Map controls (zoom + go-to-top) — bottom right, above LayerSelector */
  .map-controls {
    position: absolute;
    right: 20px;
    bottom: 224px;
    display: flex;
    flex-direction: column;
    gap: 8px;
    z-index: 20;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.4s ease;
  }

  .map-controls.visible {
    opacity: 1;
    pointer-events: auto;
  }

  .map-btn {
    width: 40px;
    height: 40px;
    border: none;
    border-radius: 8px;
    background: var(--color-glass-heavy);
    backdrop-filter: var(--blur-heavy);
    -webkit-backdrop-filter: var(--blur-heavy);
    box-shadow: var(--shadow-card);
    color: var(--color-text-dark);
    font-size: 20px;
    line-height: 1;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.2s ease;
  }

  .map-btn:hover {
    background: rgba(255, 255, 255, 0.85);
  }

  .back-to-top-btn {
    position: absolute;
    height: 40px;
    top: 76px;
    right: 20px;
    z-index: 20;
    border: none;
    border-radius: 8px;
    background: var(--color-glass-heavy);
    backdrop-filter: var(--blur-heavy);
    -webkit-backdrop-filter: var(--blur-heavy);
    box-shadow: var(--shadow-card);
    color: var(--color-text-dark);
    font-size: 13px;
    font-weight: 700;
    letter-spacing: 0.02em;
    padding: 7px 12px;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 5px;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.4s ease, background 0.25s ease, box-shadow 0.25s ease, transform 0.2s ease;
  }

  .back-to-top-btn.visible {
    opacity: 1;
    pointer-events: auto;
  }

  .back-to-top-btn:hover {
    background: rgba(255, 255, 255, 0.72);
    box-shadow: var(--shadow-card), 0 2px 8px rgba(0, 0, 0, 0.08);
    transform: translateY(-1px);
  }

  .back-to-top-btn:active {
    transform: translateY(0);
    transition-duration: 0.1s;
  }

  @media (max-width: 600px) {
    .map-controls {
      bottom: 244px;
      right: 16px;
    }

    .back-to-top-btn {
      top: 76px;
      right: 16px;
    }
  }

  .site-footer {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    z-index: 10;
    background:  #6EAF40;;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 32px;
    padding: 2.5px;
    border-top: 1px solid rgba(255, 255, 255, 0.08);
    opacity: 0;
    pointer-events: none;
    transform: translateY(100%);
    transition: opacity 0.4s ease, transform 0.4s ease;
  }

  .site-footer.visible {
    opacity: 1;
    pointer-events: auto;
    transform: translateY(0);
  }

  .site-footer a {
    display: flex;
    align-items: center;
    gap: 8px;
    color: rgba(255, 255, 255, 0.5);
    text-decoration: none;
    font-size: 0.875rem;
    transition: color 0.2s ease, opacity 0.2s ease;
  }

  .site-footer img {
    opacity: 1;
    transition: opacity 0.2s ease;
  }

  .site-footer a:hover {
    color: rgba(255, 255, 255, 0.9);
  }

  .site-footer a:hover img {
    opacity: 0.5;
  }
</style>
