<script lang="ts">
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';
  import Map from '$lib/components/Map.svelte';
  import Intro from '$lib/components/Intro.svelte';
  import Navbar from '$lib/components/Navbar.svelte';
  import InfoCard from '$lib/components/InfoCard.svelte';
  import Legend from '$lib/components/Legend.svelte';
  import SearchBar from '$lib/components/SearchBar.svelte';
  import LayerSelector from '$lib/components/LayerSelector.svelte';
  import Tooltip from '$lib/components/Tooltip.svelte';

  type Layer = 'density' | 'type-density' | 'broadleaved' | 'conifers' | 'forest-type';

  interface TooltipData {
    name: string;
    density: number;   // 0–100 from Copernicus TCD, or -1 while loading
    loading?: boolean;
    x: number;
    y: number;
  }

  interface ScrollStep {
    text: string;
    layer: Layer;
    mapStep: number;
    isExploration?: boolean;
  }

  // ─── Story steps ────────────────────────────────────────────────────────────
  const STEPS: ScrollStep[] = [
    {
      text: "Europe's forests vary wildly. This map reveals where trees thrive —and where they don't— using high-resolution satellite data.",
      layer: 'density',
      mapStep: 0
    },
    {
      text: "Zooming into southern Spain, you can see where tree cover density drops. Dry climate, land use, and desertification mean many areas fall below 30% coverage —shown here as yellowish shades.",
      layer: 'density',
      mapStep: 1
    },
    {
      text: "Lisbon's forests are mostly broadleaved, but the density is low. Compare that with the forests surrounding Ljubljana, with one of the highest forest densities in Europe.",
      layer: 'type-density',
      mapStep: 2
    },
    {
      text: "This view combines both tree cover density and dominant leaf. Darker shades mean denser coverage, while the hue shows the forest type: yellowish for broadleaf, bluish for conifer.",
      layer: 'broadleaved',
      mapStep: 3,
      isExploration: true
    }
  ];

  // ─── State ───────────────────────────────────────────────────────────────────
  let mapLoaded      = $state(false);
  let introVisible   = $state(true);
  let currentStep    = $state(-1);
  let activeLayer    = $state<Layer>('density');
  let mapStep        = $state(0);
  let tooltipData    = $state<TooltipData | null>(null);
  let lastClickX     = $state(0);
  let lastClickY     = $state(0);
  let scrollProgress = $state(0); // 0–1 within the current step

  // ─── Derived ─────────────────────────────────────────────────────────────────
  let isExploration    = $derived(currentStep === STEPS.length - 1);
  let currentStepData  = $derived(currentStep >= 0 ? STEPS[currentStep] : null);
  let showUI           = $derived(!introVisible && currentStep >= 0);
  let legendLayer      = $derived(
    (['density', 'type-density', 'broadleaved'].includes(activeLayer)
      ? activeLayer
      : 'density') as 'density' | 'type-density' | 'broadleaved'
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

  // Convert WGS84 lng/lat → EPSG:3857 (needed for Copernicus TCD identify endpoint)
  function to3857(lng: number, lat: number): [number, number] {
    const x = lng * 20037508.34 / 180;
    const y = Math.log(Math.tan((90 + lat) * Math.PI / 360)) / (Math.PI / 180) * 20037508.34 / 180;
    return [Math.round(x), Math.round(y)];
  }

  async function handleMapClick(data: { lng: number; lat: number }) {
    if (tooltipData) { tooltipData = null; return; }

    // Show loading state immediately so the user knows the click registered
    tooltipData = { name: '…', density: -1, loading: true, x: lastClickX, y: lastClickY };

    const [x, y] = to3857(data.lng, data.lat);
    const TCD_URL = `https://image.discomap.eea.europa.eu/arcgis/rest/services/GioLandPublic/HRL_TreeCoverDensity_2018/ImageServer/identify?geometry=${x},${y}&geometryType=esriGeometryPoint&sr=3857&returnGeometry=false&returnCatalogItems=false&f=json`;
    const NOM_URL = `https://nominatim.openstreetmap.org/reverse?lat=${data.lat}&lon=${data.lng}&format=json&zoom=10`;

    try {
      const [tcdRes, nomRes] = await Promise.all([
        fetch(TCD_URL),
        fetch(NOM_URL, { headers: { 'Accept-Language': 'en', 'User-Agent': 'EuropeTreesMap/1.0' } })
      ]);

      const [tcd, nom] = await Promise.all([tcdRes.json(), nomRes.json()]);

      // TCD: 0–100 are valid density %; 253/254/255 are special codes (outside/sea/unclassified)
      const raw = parseInt(tcd.value ?? '-1', 10);
      const density = raw >= 0 && raw <= 100 ? raw : 0;

      // Best available place name from Nominatim address hierarchy
      const a = nom.address ?? {};
      const name = a.city ?? a.town ?? a.village ?? a.municipality
                ?? a.county ?? a.state ?? nom.display_name?.split(',')[0]
                ?? `${data.lat.toFixed(2)}°, ${data.lng.toFixed(2)}°`;

      tooltipData = { name, density, loading: false, x: lastClickX, y: lastClickY };
    } catch {
      tooltipData = null;
    }
  }

  // ─── Scroll-driven step detection ────────────────────────────────────────────
  onMount(() => {
    if (!browser) return;

    // One step = two full viewport heights of scroll (slower card travel)
    // Layout: [intro-spacer 40vh] [step-0 200vh] [step-1 200vh] ... [outro 40vh]
    const INTRO_SPACER = 0.4;   // fraction of vh
    const STEP_HEIGHT  = 2.0;   // fraction of vh per step

    function onScroll() {
      const scrollY = window.scrollY;
      const vh      = window.innerHeight;

      // Show intro when at top, hide as soon as user scrolls
      introVisible = scrollY <= 10;

      // Which step?
      const relativeScroll = scrollY / vh - INTRO_SPACER;
      const raw = Math.floor(relativeScroll / STEP_HEIGHT);
      const next = Math.min(Math.max(raw, -1), STEPS.length - 1);

      // Progress within the current step (0 = just entered, 1 = about to leave)
      scrollProgress = next >= 0
        ? Math.max(0, Math.min(1, (relativeScroll - next * STEP_HEIGHT) / STEP_HEIGHT))
        : 0;

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
    onScroll(); // run once on mount in case page is reloaded mid-scroll
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
    <Map
      step={mapStep}
      activeLayer={activeLayer}
      onload={handleMapLoad}
      onmapclick={handleMapClick}
    />

    <Navbar visible={showUI} />

    <SearchBar
      visible={showUI}
      onflyto={handleFlyTo}
      dark={isExploration}
      exploration={isExploration}
    />

    {#if currentStepData}
      <InfoCard
        text={currentStepData.text}
        visible={showUI}
        exploration={isExploration}
        progress={scrollProgress}
      />
    {/if}

    <Legend
      layer={legendLayer}
      visible={showUI && !isExploration}
    />

    <LayerSelector
      visible={isExploration && showUI}
      activeLayer={activeLayer}
      onchange={handleLayerChange}
    />

    <Tooltip data={tooltipData} onclose={() => (tooltipData = null)} />

    <!-- Step progress dots -->
    <div class="step-dots" class:visible={showUI} aria-hidden="true">
      {#each STEPS as _step, i}
        <div class="dot" class:active={i === currentStep}></div>
      {/each}
    </div>
  </div>

  <!-- Scroll sections: invisible, just create scroll height to drive the story -->
  <div class="scroll-driver" aria-hidden="true">
    <!-- Small spacer before first step -->
    <div style="height: 40vh"></div>
    <!-- One section per story step -->
    {#each STEPS as _step}
      <div class="scroll-step"></div>
    {/each}
    <!-- Outro space -->
    <div style="height: 40vh"></div>
  </div>

</div>

<style>
  /* ── page wraps the sticky map + scroll driver ───── */
  .page {
    position: relative;
    width: 100%;
  }

  /* ── Sticky full-viewport map ────────────────────── */
  .map-stage {
    position: sticky;
    top: 0;
    width: 100%;
    height: 100svh;   /* svh = small viewport height — avoids mobile browser chrome issues */
    overflow: hidden;
    z-index: 1;
  }

  /* ── Invisible scroll track below the sticky map ─── */
  .scroll-driver {
    /* sits in the normal flow below map-stage, creating scroll height */
    position: relative;
    z-index: 0;
    pointer-events: none;
  }

  .scroll-step {
    height: 200vh;
  }

  /* ── Step progress dots ──────────────────────────── */
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

  /* Mobile: move dots to bottom-center as a horizontal row */
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
</style>
