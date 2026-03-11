<script lang="ts">
  import { onMount, flushSync } from "svelte";
  import { fade } from "svelte/transition";
  import { browser } from "$app/environment";
  import Map from "$lib/components/Map.svelte";
  import Intro from "$lib/components/Intro.svelte";
  import Navbar from "$lib/components/Navbar.svelte";
  import InfoCard from "$lib/components/InfoCard.svelte";
  import SearchBar from "$lib/components/SearchBar.svelte";
  import LayerSelector from "$lib/components/LayerSelector.svelte";
  import Tooltip from "$lib/components/Tooltip.svelte";

  type Layer =
    | "density"
    | "type-density"
    | "broadleaved"
    | "conifers"
    | "forest-type";

  interface TooltipData {
    name: string;
    density: number; // 0–100 from EEA TCD identify
    leafType: number; // 1=broadleaved, 2=coniferous, 0=unknown
    conifers: number; // ha from DLT computeHistograms (-1 = unavailable)
    broadleaves: number; // ha from DLT computeHistograms (-1 = unavailable)
    loading?: boolean;
    x: number;
    y: number;
  }

  interface ScrollStep {
    text: string;
    layer: Layer;
    mapStep: number;
    icon?: string;
    isExploration?: boolean;
    isSplit?: boolean;
    splitSteps?: [number, number]; // [leftMapStep, rightMapStep]
  }

  // ─── Story steps — matching Observable notebook @694 ────────────────────────
  const STEPS: ScrollStep[] = [
    {
      text: "Europe's forests vary wildly. This map reveals where trees <strong>thrive</strong> —and where they <strong>don't</strong>— using high-resolution satellite data.",
      layer: "density",
      mapStep: 0,
      icon: "/assets/trees-icon.svg",
    },
    {
      text: "Zooming into southern Spain, you can see where <strong>tree cover density drops</strong>. Dry climate, land use, and desertification mean many areas fall below <strong>30% coverage</strong> —shown here in yellowish shades.",
      layer: "density",
      mapStep: 1,
    },
    {
      text: "In contrast, Scandinavia is blanketed in darker green. Sweden and Finland have among the <strong>highest tree densities in Europe</strong>, with many regions over <strong>80% coverage</strong>.",
      layer: "density",
      mapStep: 2,
    },
    {
      text: "But tree density is only <strong>one part of the story</strong> —and the data. We can also split trees by type —<strong>broadleaved</strong> and <strong>coniferous</strong>.",
      layer: "forest-type",
      mapStep: 3,
    },
    {
      text: "In the south — Portugal, Italy, the Balkans — <strong>broadleaved trees</strong> dominate: <strong>oaks, birches, beeches, and chestnuts</strong>, rooted in temperate and Mediterranean soils.",
      layer: "broadleaved",
      mapStep: 4,
    },
    {
      text: "From Germany through the Baltic to the Nordic countries, the landscape is mainly <strong>coniferous forests</strong>: <strong>pines, spruces, cypresses, and firs</strong>, which are well-adapted to the colder, nutrient-poor soils.",
      layer: "conifers",
      mapStep: 5,
    },
    {
      text: "This view combines both <strong>tree cover density</strong> and the <strong>dominant leaf type</strong>. Darker shades of the same color mean <strong>denser coverage</strong>, while the hue still shows the forest type.",
      layer: "type-density",
      mapStep: 6,
    },
    {
      text: "Lisbon's forests are mostly broadleaved, but the density is <strong>low</strong>. Contrast that with <strong>Ljubljana</strong> — surrounded by one of the <strong>highest forest densities in Europe</strong>.",
      layer: "type-density",
      mapStep: 7,
      isSplit: true,
      splitSteps: [7, 8],
    },
    {
      text: "Curious what grows near you? Use the map to explore <strong>forest patterns across Europe</strong> —by density, by type, and by place.",
      layer: "type-density",
      mapStep: 9,
      icon: "/assets/trees-icon.svg",
      isExploration: true,
    },
  ];

  // ─── State ───────────────────────────────────────────────────────────────────
  let mapLoaded = $state(false);
  let introVisible = $state(true);
  let currentStep = $state(-1);
  let activeLayer = $state<Layer>("density");
  let mapStep = $state(0);
  let tooltipData = $state<TooltipData | null>(null);
  let lastClickX = $state(0);
  let lastClickY = $state(0);
  let scrollProgress = $state(0); // 0–1 within the current step
  let explorationActive = $state(false);
  let cursorX = $state(0);
  let cursorY = $state(0);
  let hoverTimer: ReturnType<typeof setTimeout> | null = null;
  let fetchController: AbortController | null = null;
  let showGuide = $state(false);
  let guideShown = false; // plain ref — only show once per session

  $effect(() => {
    if (explorationActive && !guideShown) {
      guideShown = true;
      showGuide = true;
    }
    if (!explorationActive) showGuide = false;
  });

  // ─── Derived ─────────────────────────────────────────────────────────────────
  let isExploration = $derived(currentStep === STEPS.length - 1);
  let currentStepData = $derived(currentStep >= 0 ? STEPS[currentStep] : null);
  let isSplit = $derived(currentStepData?.isSplit === true);
  let splitLeftStep = $derived(currentStepData?.splitSteps?.[0] ?? mapStep);
  let splitRightStep = $derived(currentStepData?.splitSteps?.[1] ?? mapStep);
  let showUI = $derived(!introVisible && currentStep >= 0);
  // Legend builds up as layers are introduced, ending with the full legend
  let cumulativeLayers = $derived<Layer[]>(
    currentStep < 0  ? [] :
    currentStep <= 2 ? ['density'] :
    currentStep === 3 ? ['density', 'forest-type'] :
    currentStep === 4 ? ['density', 'forest-type', 'broadleaved'] :
    currentStep === 5 ? ['density', 'forest-type', 'broadleaved', 'conifers'] :
                        ['density', 'forest-type', 'broadleaved', 'conifers', 'type-density']
  );

  // ─── Handlers ────────────────────────────────────────────────────────────────
  function handleMapLoad() {
    mapLoaded = true;
  }

  function handleFlyTo(coords: { lat: number; lng: number }) {
    document.dispatchEvent(new CustomEvent("flyto", { detail: coords }));
  }

  function handleLayerChange(layer: Layer) {
    activeLayer = layer;
  }

  function goToTop() {
    explorationActive = false;
    document.body.style.overflow = "";
    window.scrollTo({ top: 0, behavior: "smooth" });
  }

  function zoomIn() {
    document.dispatchEvent(new CustomEvent("map:zoomin"));
  }
  function zoomOut() {
    document.dispatchEvent(new CustomEvent("map:zoomout"));
  }

  // Convert WGS84 lng/lat → EPSG:3857 (needed for EEA ImageServer identify)
  function to3857(lng: number, lat: number): [number, number] {
    const x = (lng * 20037508.34) / 180;
    const y =
      ((Math.log(Math.tan(((90 + lat) * Math.PI) / 360)) / (Math.PI / 180)) *
        20037508.34) /
      180;
    return [Math.round(x), Math.round(y)];
  }

  async function fetchAtPosition(
    lng: number,
    lat: number,
    x: number,
    y: number,
    signal: AbortSignal,
  ): Promise<void> {
    const [mx, my] = to3857(lng, lat);
    const EEA_BASE =
      "https://image.discomap.eea.europa.eu/arcgis/rest/services/GioLandPublic";
    const IDENTIFY = (svc: string) =>
      `${EEA_BASE}/${svc}/ImageServer/identify?geometry=${mx},${my}&geometryType=esriGeometryPoint&sr=3857&returnGeometry=false&returnCatalogItems=false&f=json`;

    const TCD_URL = IDENTIFY("HRL_TreeCoverDensity_2018");
    const DLT_URL = IDENTIFY("HRL_DominantLeafType2018");
    const NOM_URL = `https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json&zoom=10&polygon_geojson=1`;

    let density = 0;
    let leafType = 0;
    let name = `${lat.toFixed(2)}°, ${lng.toFixed(2)}°`;
    let conifers = -1;
    let broadleaves = -1;

    try {
      const [tcdRes, dltRes, nomRes] = await Promise.all([
        fetch(TCD_URL, { signal }),
        fetch(DLT_URL, { signal }),
        fetch(NOM_URL, {
          signal,
          headers: {
            "Accept-Language": "en",
            "User-Agent": "EuropeTreesMap/1.0",
          },
        }),
      ]);

      const [tcd, dlt, nom] = await Promise.all([
        tcdRes.json(),
        dltRes.json(),
        nomRes.json(),
      ]);

      // TCD: 0–100 are valid density %; special codes mean outside/sea/unclassified
      const rawTcd = parseInt(tcd.value ?? "-1", 10);
      density = rawTcd >= 0 && rawTcd <= 100 ? rawTcd : -1; // -1 = sea/outside coverage

      // DLT: 1=broadleaved, 2=coniferous; other values = no data
      const rawDlt = parseInt(dlt.value ?? "0", 10);
      leafType = rawDlt === 1 || rawDlt === 2 ? rawDlt : 0;

      const a = nom.address ?? {};
      name =
        a.city ??
        a.town ??
        a.village ??
        a.municipality ??
        a.county ??
        a.state ??
        nom.display_name?.split(",")[0] ??
        `${lat.toFixed(2)}°, ${lng.toFixed(2)}°`;

      // Fetch histogram data from EEA DLT using the Nominatim polygon boundary.
      // ArcGIS ImageServer expects ArcGIS JSON (rings[]), not raw GeoJSON.
      // Each pixel = 10m × 10m = 0.01 ha; bins[1]=broadleaved, bins[2]=coniferous
      if (nom.geojson) {
        try {
          const gj = nom.geojson;
          let rings: number[][][] = [];
          if (gj.type === "Polygon") {
            rings = gj.coordinates;
          } else if (gj.type === "MultiPolygon") {
            rings = gj.coordinates.flatMap((p: number[][][]) => p);
          }
          if (rings.length > 0) {
            const arcGeom = { rings, spatialReference: { wkid: 4326 } };
            const histUrl =
              `${EEA_BASE}/HRL_DominantLeafType2018/ImageServer/computeHistograms` +
              `?geometry=${encodeURIComponent(JSON.stringify(arcGeom))}` +
              `&geometryType=esriGeometryPolygon&sr=4326&f=json`;
            const histRes = await fetch(histUrl, { signal });
            const hist = await histRes.json();
            const bins = hist?.histograms?.[0]?.counts ?? [];
            broadleaves = Math.round((bins[1] ?? 0) * 0.01);
            conifers = Math.round((bins[2] ?? 0) * 0.01);
          }
        } catch (err: any) {
          if (err?.name === "AbortError") return;
          // histogram unavailable — keep -1 to show "—" in tooltip
        }
      }
    } catch (err: any) {
      if (err?.name === "AbortError") return; // silently cancel stale requests
      // main fetches failed — show coordinate fallback with no stats
    }

    if (density === -1) {
      tooltipData = null;
      return;
    }

    tooltipData = {
      name,
      density,
      leafType,
      conifers,
      broadleaves,
      loading: false,
      x,
      y,
    };
  }

  async function handleMapClick(data: { lng: number; lat: number; x: number; y: number }) {
    if (tooltipData) {
      tooltipData = null;
      return;
    } // toggle-close
    // flushSync forces Svelte to paint the loading skeleton before the fetch starts
    flushSync(() => {
      tooltipData = {
        name: '...',
        density: 0,
        leafType: 0,
        conifers: -1,
        broadleaves: -1,
        loading: true,
        x: data.x,
        y: data.y,
      };
    });
    fetchController?.abort();
    fetchController = new AbortController();
    await fetchAtPosition(
      data.lng,
      data.lat,
      data.x,
      data.y,
      fetchController.signal,
    );
  }

  function handleMapMouseMove(data: {
    lng: number;
    lat: number;
    x: number;
    y: number;
  }) {
    if (!explorationActive) return;
    cursorX = data.x;
    cursorY = data.y;
    tooltipData = null; // clear stale tooltip immediately
    fetchController?.abort(); // cancel any in-flight request
    fetchController = null;
    if (hoverTimer !== null) clearTimeout(hoverTimer);
    hoverTimer = setTimeout(async () => {
      fetchController = new AbortController();
      await fetchAtPosition(
        data.lng,
        data.lat,
        data.x,
        data.y,
        fetchController.signal,
      );
    }, 100);
  }

  function handleMapMouseLeave() {
    if (hoverTimer !== null) {
      clearTimeout(hoverTimer);
      hoverTimer = null;
    }
    cursorX = 0;
    cursorY = 0;
  }

  // ─── Scroll-driven step detection ────────────────────────────────────────────
  onMount(() => {
    if (!browser) return;

    const INTRO_SPACER = 0.4; // fraction of vh
    const STEP_HEIGHT = 2.0; // fraction of vh per step

    function onScroll() {
      const scrollY = window.scrollY;
      const vh = window.innerHeight;

      introVisible = scrollY <= 10;

      const relativeScroll = scrollY / vh - INTRO_SPACER;
      const raw = Math.floor(relativeScroll / STEP_HEIGHT);
      const next = Math.min(Math.max(raw, -1), STEPS.length - 1);

      scrollProgress =
        next >= 0
          ? Math.max(
              0,
              Math.min(1, (relativeScroll - next * STEP_HEIGHT) / STEP_HEIGHT),
            )
          : 0;

      const stepIsExploration = next >= 0 && STEPS[next].isExploration === true;

      if (stepIsExploration && scrollProgress > 0.75 && !explorationActive) {
        explorationActive = true;
        document.body.style.overflow = "hidden";
      }

      if (!stepIsExploration && explorationActive) {
        explorationActive = false;
        document.body.style.overflow = "";
      }

      if (next !== currentStep) {
        currentStep = next;
        if (next >= 0) {
          activeLayer = STEPS[next].layer as Layer;
          mapStep = STEPS[next].mapStep;
          tooltipData = null;
        }
      }
    }

    window.addEventListener("scroll", onScroll, { passive: true });
    onScroll();
    return () => window.removeEventListener("scroll", onScroll);
  });
</script>

<svelte:head>
  <title>The Most Detailed Map of Europe's Trees</title>
  <meta
    name="description"
    content="An interactive scrollytelling map of Europe's forest cover density and types."
  />
</svelte:head>

<!-- ── Intro overlay (fixed, fades on first scroll) ──────────────────────── -->
<Intro visible={introVisible} loading={!mapLoaded} />

<!-- ── Page ──────────────────────────────────────────────────────────────── -->
<div class="page">
  <!-- Sticky map canvas: stays pinned to viewport while user scrolls -->
  <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_noninteractive_element_interactions -->
  <div
    class="map-stage"
    class:exploration={explorationActive}
    class:has-tooltip={explorationActive && tooltipData != null}
    role="application"
    aria-label="Interactive forest map"
    onclick={(e) => {
      lastClickX = e.clientX;
      lastClickY = e.clientY;
      if (showGuide) showGuide = false;
    }}
    onkeydown={() => {}}
  >
    <!-- Single map (always mounted, fades out in split mode) -->
    <div class="single-map" class:faded={isSplit}>
      <Map
        step={mapStep}
        {activeLayer}
        onload={handleMapLoad}
        onmapclick={handleMapClick}
        onmapmousemove={handleMapMouseMove}
        onmapmouseleave={handleMapMouseLeave}
        onmapmove={() => { tooltipData = null; fetchController?.abort(); }}
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

    <button
      class="back-to-top-btn"
      class:visible={explorationActive && showUI}
      onclick={goToTop}
      aria-label="Back to story"
    >
      <svg width="14" height="14" viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
        <polyline points="2,9 7,4 12,9"/>
      </svg>
      Top
    </button>

    {#if currentStepData}
      <InfoCard
        text={currentStepData.text}
        icon={currentStepData.icon}
        visible={showUI && !explorationActive}
        progress={scrollProgress}
      />
    {/if}

    <!-- Legend: builds up during story, full interactive bar during exploration -->
    <LayerSelector
      visible={showUI}
      {activeLayer}
      layers={cumulativeLayers}
      onchange={explorationActive ? handleLayerChange : undefined}
    />

    <!-- Map controls: zoom in/out + go to top (visible in exploration) -->
    <div
      class="map-controls"
      class:visible={explorationActive && showUI}
      aria-label="Map controls"
    >
      <button
        class="map-btn"
        onclick={zoomIn}
        title="Zoom in"
        aria-label="Zoom in">+</button
      >
      <button
        class="map-btn"
        onclick={zoomOut}
        title="Zoom out"
        aria-label="Zoom out">−</button
      >
    </div>

    {#if showGuide}
      <div class="explore-guide" transition:fade={{ duration: 300 }} aria-live="polite" role="button" tabindex="0" onclick={() => showGuide = false} onkeydown={(e) => e.key === 'Enter' && (showGuide = false)}>
        <p class="guide-title">Explore the map</p>
        <ul class="guide-tips">
          <li>
            <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" aria-hidden="true">
              <circle cx="8" cy="8" r="6"/>
              <circle cx="8" cy="8" r="1.5" fill="currentColor" stroke="none"/>
            </svg>
            <span>Hover anywhere to reveal forest data</span>
          </li>
          <li>
            <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" aria-hidden="true">
              <circle cx="6.5" cy="6.5" r="4"/>
              <line x1="9.5" y1="9.5" x2="13.5" y2="13.5"/>
            </svg>
            <span>Search for any place in Europe</span>
          </li>
          <li>
            <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
              <rect x="2" y="3" width="12" height="2.5" rx="0.5"/>
              <rect x="2" y="6.75" width="12" height="2.5" rx="0.5"/>
              <rect x="2" y="10.5" width="12" height="2.5" rx="0.5"/>
            </svg>
            <span>Switch layers to explore by density or type</span>
          </li>
        </ul>
        <p class="guide-hint">
          Start exploring
          <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <line x1="2" y1="6" x2="10" y2="6"/>
            <polyline points="7,3 10,6 7,9"/>
          </svg>
        </p>
      </div>
    {/if}

    {#if explorationActive && tooltipData != null}
      <div
        class="cursor-ring"
        style="transform: translate({tooltipData.x}px, {tooltipData.y}px)"
        aria-hidden="true"
      ></div>
    {/if}

    <Tooltip data={tooltipData} onclose={() => (tooltipData = null)} />

    <!-- Step progress dots -->
    <div
      class="step-dots"
      class:visible={showUI && !explorationActive}
      aria-hidden="true"
    >
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
    <a
      href="https://www.fundaciovit.org/"
      target="_blank"
      rel="noopener noreferrer"
      aria-label="Fundació ViT"
    >
      <img src="/assets/vit-logo-bw.svg" alt="ViT" width="40" />
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

  .map-stage.exploration,
  .map-stage.exploration :global(.mapboxgl-canvas) {
    cursor: pointer !important;
  }

  .map-stage.has-tooltip :global(.mapboxgl-canvas) {
    cursor: none !important;
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
    transition:
      background 0.3s ease,
      transform 0.3s ease;
  }

  .dot.active {
    background: rgba(255, 255, 255, 0.9);
    transform: scale(1.5);
  }

  @media (max-width: 600px) {
    .step-dots {
      display: none;
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
    width: 5px;
    flex-shrink: 0;
    background: rgba(255, 255, 255, 1);
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
    background: white;
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
    transition:
      opacity 0.4s ease,
      background 0.25s ease,
      box-shadow 0.25s ease,
      transform 0.2s ease;
  }

  .back-to-top-btn.visible {
    opacity: 1;
    pointer-events: auto;
  }

  .back-to-top-btn:hover {
    background: rgba(255, 255, 255, 0.72);
    box-shadow:
      var(--shadow-card),
      0 2px 8px rgba(0, 0, 0, 0.08);
    transform: translateY(-1px);
  }

  .back-to-top-btn:active {
    transform: translateY(0);
    transition-duration: 0.1s;
  }

  @media (max-width: 600px) {
    .map-controls {
      bottom: 190px;
      right: 16px;
    }

    .back-to-top-btn {
      top: auto;
      right: auto;
      bottom: 190px;
      left: 16px;
    }
  }

  .cursor-ring {
    position: fixed;
    top: -8px;
    left: -8px;
    width: 16px;
    height: 16px;
    border-radius: 50%;
    border: 2px solid rgba(0, 0, 0, 0.65);
    background: transparent;
    pointer-events: none;
    z-index: 100;
    transition: opacity 0.15s ease;
  }

  .site-footer {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    z-index: 10;
    background: #6eaf40;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 32px;
    padding: 2.5px;
    border-top: 1px solid rgba(255, 255, 255, 0.08);
    opacity: 0;
    pointer-events: none;
    transform: translateY(100%);
    transition:
      opacity 0.4s ease,
      transform 0.4s ease;
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
    transition:
      color 0.2s ease,
      opacity 0.2s ease;
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

  .explore-guide {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 50;
    background: rgba(255, 255, 255, 0.75);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    border-radius: 12px;
    box-shadow: 0 -2px 16px 0 rgba(0, 0, 0, 0.10);
    padding: 24px 28px;
    color: var(--color-text-dark);
    max-width: 300px;
    cursor: pointer;
  }

  .guide-title {
    font-size: 1rem;
    font-weight: 700;
    margin: 0 0 16px;
    letter-spacing: 0.01em;
  }

  .guide-tips {
    list-style: none;
    margin: 0 0 16px;
    padding: 0;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .guide-tips li {
    display: flex;
    align-items: flex-start;
    gap: 10px;
    font-size: 0.85rem;
    line-height: 1.4;
    color: var(--color-text-dark);
  }

  .guide-tips svg {
    flex-shrink: 0;
    margin-top: 1px;
    opacity: 0.55;
  }

  .guide-hint {
    margin: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 5px;
    font-size: 0.78rem;
    font-weight: 600;
    color: var(--color-text-dark);
    padding: 5px 14px;
    letter-spacing: 0.02em;
    transition: background 0.15s ease;
  }

  .explore-guide:hover .guide-hint {
    opacity: 0.75;
  }

  @media (max-width: 600px) {
    .explore-guide {
      left: 16px;
      right: 16px;
      top: auto;
      bottom: 300px;
      transform: none;
      max-width: none;
    }
  }
</style>
