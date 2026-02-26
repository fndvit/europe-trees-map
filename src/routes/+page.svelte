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
    density: number;    // 0–100 from Copernicus TCD, or -1 while loading
    leafType: number;   // 1=broadleaved, 2=coniferous, 0=unknown/no data
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
      text: "Here's a direct comparison. On the left, tree cover density alone. On the right, density combined with forest type — showing both how dense and what kind.",
      layer: 'density',
      mapStep: 6,
      isSplit: true
    },
    {
      text: "This view combines both tree cover density and the dominant leaf type. Darker shades of the same color mean denser coverage, while the hue still shows the forest type.",
      layer: 'type-density',
      mapStep: 6
    },
    {
      text: "Lisbon's forests are mostly broadleaved, but the density is low. Flying over to Slovenia…",
      layer: 'type-density',
      mapStep: 7
    },
    {
      text: "…the forests surrounding Ljubljana show one of the highest forest densities in Europe.",
      layer: 'type-density',
      mapStep: 8
    },
    {
      text: "Curious what grows near you? Use the map to explore forest patterns across Europe —by density, by type, and by place.",
      layer: 'type-density',
      mapStep: 9,
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
  let isSplit          = $derived(currentStepData?.isSplit === true);
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

  // Convert WGS84 lng/lat → EPSG:3857 (needed for EEA ImageServer identify)
  function to3857(lng: number, lat: number): [number, number] {
    const x = lng * 20037508.34 / 180;
    const y = Math.log(Math.tan((90 + lat) * Math.PI / 360)) / (Math.PI / 180) * 20037508.34 / 180;
    return [Math.round(x), Math.round(y)];
  }

  async function handleMapClick(data: { lng: number; lat: number }) {
    if (tooltipData) { tooltipData = null; return; }

    tooltipData = { name: '…', density: -1, leafType: 0, loading: true, x: lastClickX, y: lastClickY };

    const [x, y] = to3857(data.lng, data.lat);
    const EEA_BASE = 'https://image.discomap.eea.europa.eu/arcgis/rest/services/GioLandPublic';
    const IDENTIFY = (svc: string) =>
      `${EEA_BASE}/${svc}/ImageServer/identify?geometry=${x},${y}&geometryType=esriGeometryPoint&sr=3857&returnGeometry=false&returnCatalogItems=false&f=json`;

    const TCD_URL = IDENTIFY('HRL_TreeCoverDensity_2018');
    const DLT_URL = IDENTIFY('HRL_DominantLeafType2018');
    const NOM_URL = `https://nominatim.openstreetmap.org/reverse?lat=${data.lat}&lon=${data.lng}&format=json&zoom=10`;

    try {
      const [tcdRes, dltRes, nomRes] = await Promise.all([
        fetch(TCD_URL),
        fetch(DLT_URL),
        fetch(NOM_URL, { headers: { 'Accept-Language': 'en', 'User-Agent': 'EuropeTreesMap/1.0' } })
      ]);

      const [tcd, dlt, nom] = await Promise.all([tcdRes.json(), dltRes.json(), nomRes.json()]);

      // TCD: 0–100 are valid density %; special codes mean outside/sea/unclassified
      const rawTcd = parseInt(tcd.value ?? '-1', 10);
      const density = rawTcd >= 0 && rawTcd <= 100 ? rawTcd : 0;

      // DLT: 1=broadleaved, 2=coniferous; other values = no data
      const rawDlt = parseInt(dlt.value ?? '0', 10);
      const leafType = rawDlt === 1 || rawDlt === 2 ? rawDlt : 0;

      const a = nom.address ?? {};
      const name = a.city ?? a.town ?? a.village ?? a.municipality
                ?? a.county ?? a.state ?? nom.display_name?.split(',')[0]
                ?? `${data.lat.toFixed(2)}°, ${data.lng.toFixed(2)}°`;

      tooltipData = { name, density, leafType, loading: false, x: lastClickX, y: lastClickY };
    } catch {
      tooltipData = null;
    }
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
      />
    </div>

    <!-- Split stage (always mounted, fades in for split step) -->
    <div class="split-stage" class:visible={isSplit}>
      <div class="split-half">
        <Map step={mapStep} activeLayer="density" />
        <div class="split-label">Tree cover density</div>
      </div>
      <div class="split-divider"></div>
      <div class="split-half">
        <Map step={mapStep} activeLayer="type-density" />
        <div class="split-label">Forest type + density</div>
      </div>
    </div>

    <Navbar visible={showUI} />

    <SearchBar
      visible={isExploration && showUI}
      onflyto={handleFlyTo}
      dark={isExploration}
      exploration={isExploration}
    />

    {#if currentStepData}
      <InfoCard
        text={currentStepData.text}
        icon="/assets/trees-icon.svg"
        visible={showUI}
        exploration={isExploration}
        progress={scrollProgress}
      />
    {/if}

    <Legend
      layer={legendLayer}
      visible={showUI && !isExploration && !isSplit}
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
    <div style="height: 40vh"></div>
    {#each STEPS as _step}
      <div class="scroll-step"></div>
    {/each}
    <div style="height: 40vh"></div>
  </div>

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

  .split-label {
    position: absolute;
    bottom: 36px;
    left: 50%;
    transform: translateX(-50%);
    white-space: nowrap;
    background: var(--color-glass-heavy);
    backdrop-filter: var(--blur-medium);
    -webkit-backdrop-filter: var(--blur-medium);
    padding: 5px 14px;
    border-radius: var(--radius-pill);
    font-size: 11px;
    font-weight: 500;
    letter-spacing: 0.03em;
    color: var(--color-text-dark);
    pointer-events: none;
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
</style>
