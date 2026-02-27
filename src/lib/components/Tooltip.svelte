<script lang="ts">
  // Raw SVG tree icons (broadleaved and coniferous)
  import bro1 from '$lib/assets/icons_bro-1.svg?raw';
  import bro2 from '$lib/assets/icons_bro-2.svg?raw';
  import bro3 from '$lib/assets/icons_bro-3.svg?raw';
  import con1 from '$lib/assets/icons_con-1.svg?raw';
  import con2 from '$lib/assets/icons_con-2.svg?raw';
  import con3 from '$lib/assets/icons_con-3.svg?raw';

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

  interface Props {
    data?: TooltipData | null;
    onclose?: () => void;
  }

  let { data = null, onclose }: Props = $props();

  const W = 280;
  const H = 260; // approximate height for upward offset

  let cx = $derived(
    data
      ? Math.min(Math.max(data.x, W / 2 + 12), (typeof window !== 'undefined' ? window.innerWidth : 1200) - W / 2 - 12)
      : 0
  );
  let cy = $derived(data ? Math.max(data.y - H - 16, 60) : 0);

  function densityCategory(d: number): string {
    if (d === 0)   return 'no';
    if (d <= 10)   return 'very sparse';
    if (d <= 35)   return 'sparse';
    if (d <= 70)   return 'moderate';
    if (d <= 80)   return 'dense';
    return 'very dense';
  }

  function leafTypeLabel(leafType: number, density: number): string {
    if (density === 0) return 'No tree cover detected in this area.';
    if (leafType === 1) return `A ${densityCategory(density)} canopy, mostly broadleaved trees.`;
    if (leafType === 2) return `A ${densityCategory(density)} canopy, mostly coniferous trees.`;
    return `A ${densityCategory(density)} canopy.`;
  }

  function pickIcon(type: 'broadleaved' | 'coniferous', seed: number): string {
    const pool = type === 'coniferous' ? [con1, con2, con3] : [bro1, bro2, bro3];
    return pool[seed % 3];
  }

  // Build a compact row of 1–3 tree icons based on density and leaf type
  function buildTreeIcons(density: number, leafType: number): Array<{ type: 'broadleaved' | 'coniferous'; seed: number }> {
    if (density === 0) return [];
    const count = Math.min(3, Math.max(1, Math.round(density / 35)));
    let types: Array<'broadleaved' | 'coniferous'>;
    if (leafType === 2) {
      types = Array(count).fill('coniferous');
    } else if (leafType === 1) {
      types = Array(count).fill('broadleaved');
    } else {
      types = count === 1 ? ['broadleaved'] : ['coniferous', 'broadleaved'];
    }
    return types.map((type, i) => ({ type, seed: i }));
  }

  let treeIcons = $derived(
    data && !data.loading ? buildTreeIcons(data.density, data.leafType) : []
  );

  // ── Bar position markers (0–100 = CSS left%) ──────────────────────────────
  let tcdPos = $derived(data ? Math.max(0, Math.min(100, data.density)) : 0);

  let totalForest = $derived(
    data && data.conifers >= 0 && data.broadleaves >= 0
      ? data.conifers + data.broadleaves
      : 0
  );

  let conifersPos = $derived(
    totalForest > 0 && data && data.conifers >= 0
      ? Math.round((data.conifers / totalForest) * 100)
      : 0
  );

  let broadleavesPos = $derived(
    totalForest > 0 && data && data.broadleaves >= 0
      ? Math.round((data.broadleaves / totalForest) * 100)
      : 0
  );

  function formatHa(val: number): string {
    if (val < 0) return '—';
    if (val >= 1_000_000) return `${(val / 1_000_000).toFixed(1)}M ha`;
    if (val >= 1_000)     return `${Math.round(val / 1_000)}k ha`;
    return `${val} ha`;
  }
</script>

{#if data}
  <div class="tooltip" style="left:{cx}px; top:{cy}px;">
    <div class="bubble">

      <button class="close-btn" onclick={onclose} aria-label="Close">
        <svg xmlns="http://www.w3.org/2000/svg" width="12" height="12" viewBox="0 0 24 24"
          fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
          <path d="M18 6 6 18M6 6l12 12"/>
        </svg>
      </button>

      <h3 class="place-name">{data.name}</h3>
      <hr class="divider" />

      {#if data.loading}
        <div class="skeletons">
          <div class="skeleton desc-skel"></div>
          <div class="skeleton bar-skel"></div>
          <div class="skeleton bar-lbl"></div>
          <div class="skeleton bar-skel"></div>
          <div class="skeleton bar-lbl"></div>
          <div class="skeleton bar-skel"></div>
        </div>
      {:else}
        <!-- Description + tree icons row (hidden on mobile) -->
        <div class="desc-row">
          <p class="nl-sentence">{leafTypeLabel(data.leafType, data.density)}</p>
          {#if treeIcons.length > 0}
            <div class="tree-icons" aria-hidden="true">
              {#each treeIcons as tree}
                <span class="tree-icon">
                  {@html pickIcon(tree.type, tree.seed)}
                </span>
              {/each}
            </div>
          {/if}
        </div>

        <hr class="divider" />

        <!-- Desktop: gradient bars with position markers -->
        <div class="bars-desktop">
          <div class="stat-row">
            <span class="stat-lbl">Tree cover density</span>
            <span class="stat-val">{data.density}%</span>
          </div>
          <div class="bar-track">
            <div class="bar-gradient tcd-gradient"></div>
            <div class="bar-marker" style="left:{tcdPos}%"></div>
          </div>

          <div class="stat-row">
            <span class="stat-lbl">Conifers</span>
            <span class="stat-val">{formatHa(data.conifers)}</span>
          </div>
          <div class="bar-track">
            <div class="bar-gradient con-gradient"></div>
            <div class="bar-marker" style="left:{conifersPos}%"></div>
          </div>

          <div class="stat-row">
            <span class="stat-lbl">Broadleaves</span>
            <span class="stat-val">{formatHa(data.broadleaves)}</span>
          </div>
          <div class="bar-track">
            <div class="bar-gradient bro-gradient"></div>
            <div class="bar-marker" style="left:{broadleavesPos}%"></div>
          </div>
        </div>

        <!-- Mobile: plain numbers only, no bars -->
        <div class="stats-mobile">
          <div class="mobile-stat">
            <span class="stat-lbl">Tree cover density</span>
            <span class="stat-val">{data.density}%</span>
          </div>
          <div class="mobile-stat">
            <span class="stat-lbl">Conifers</span>
            <span class="stat-val">{formatHa(data.conifers)}</span>
          </div>
          <div class="mobile-stat">
            <span class="stat-lbl">Broadleaves</span>
            <span class="stat-val">{formatHa(data.broadleaves)}</span>
          </div>
        </div>

        <p class="source">Tiles: Copernicus HRL 2023 · Identify: 2018 · © EEA</p>
      {/if}
    </div>
    <div class="arrow"></div>
  </div>
{/if}

<style>
  .tooltip {
    position: fixed;
    transform: translateX(-50%);
    z-index: 50;
    display: flex;
    flex-direction: column;
    align-items: center;
    pointer-events: auto;
    animation: pop 0.18s ease;
  }

  @keyframes pop {
    from { opacity: 0; transform: translateX(-50%) scale(0.92) translateY(4px); }
    to   { opacity: 1; transform: translateX(-50%) scale(1)    translateY(0);   }
  }

  .bubble {
    background: var(--color-glass-heavy);
    backdrop-filter: var(--blur-light);
    -webkit-backdrop-filter: var(--blur-light);
    border-radius: 8px;
    padding: 14px 14px 12px;
    width: 280px;
    box-shadow: var(--shadow-tooltip);
    position: relative;
  }

  @media (max-width: 600px) {
    .bubble {
      width: 220px;
      padding: 12px 12px 10px;
    }
  }

  .close-btn {
    position: absolute;
    top: 8px;
    right: 8px;
    background: none;
    border: none;
    cursor: pointer;
    color: var(--color-text-medium);
    padding: 4px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    transition: background 0.15s;
  }
  .close-btn:hover { background: rgba(0,0,0,0.06); }

  .place-name {
    font-family: var(--font);
    font-weight: 600;
    font-size: 16px;
    color: var(--color-text-dark);
    margin-bottom: 10px;
    padding-right: 22px;
    line-height: 1.2;
  }

  .divider {
    border: none;
    border-top: 1px solid rgba(0,0,0,0.08);
    margin: 0 0 10px;
  }

  /* ── Loading skeletons ─────────────── */
  .skeletons { display: flex; flex-direction: column; gap: 8px; }

  .skeleton {
    border-radius: 5px;
    background: linear-gradient(90deg, rgba(0,0,0,0.06) 25%, rgba(0,0,0,0.12) 50%, rgba(0,0,0,0.06) 75%);
    background-size: 200% 100%;
    animation: shimmer 1.2s ease-in-out infinite;
  }
  .skeleton.desc-skel { height: 28px; width: 100%; }
  .skeleton.bar-skel  { height: 8px; width: 100%; }
  .skeleton.bar-lbl   { height: 10px; width: 70%; }

  @keyframes shimmer {
    0%   { background-position: 200% 0; }
    100% { background-position: -200% 0; }
  }

  /* ── Description + icons row ────────── */
  .desc-row {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 10px;
  }

  .nl-sentence {
    flex: 1;
    font-family: var(--font);
    font-size: 12px;
    font-style: italic;
    color: var(--color-text-medium);
    line-height: 1.4;
    margin: 0;
  }

  .tree-icons {
    flex-shrink: 0;
    width: 64px;
    height: 26px;
    display: flex;
    align-items: flex-end;
    gap: 2px;
  }

  .tree-icon {
    display: flex;
    align-items: flex-end;
    flex: 1;
    max-width: 22px;
  }

  .tree-icon :global(svg) {
    width: 100%;
    height: 26px;
    filter: invert(1);
    overflow: visible;
  }

  /* ── Stat rows ─────────────────────── */
  .stat-row {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    margin-bottom: 5px;
  }

  .stat-lbl {
    font-family: var(--font);
    font-weight: 300;
    font-size: 11px;
    color: var(--color-text-dark);
  }

  .stat-val {
    font-family: var(--font);
    font-weight: 700;
    font-size: 12px;
    color: var(--color-text-dark);
  }

  /* ── Gradient bars with position marker ── */
  .bar-track {
    position: relative;
    width: 100%;
    height: 8px;
    border-radius: 4px;
    margin-bottom: 10px;
  }

  .bar-gradient {
    position: absolute;
    inset: 0;
    border-radius: 4px;
  }

  .tcd-gradient {
    background: linear-gradient(90deg, #ffec81 0%, #005f00 100%);
  }

  .con-gradient {
    background: linear-gradient(90deg, rgba(7,82,63,0.12) 0%, rgba(7,82,63,1) 100%);
  }

  .bro-gradient {
    background: linear-gradient(90deg, rgba(199,180,71,0.12) 0%, rgba(199,180,71,1) 100%);
  }

  .bar-marker {
    position: absolute;
    top: 50%;
    transform: translate(-50%, -50%);
    width: 2px;
    height: 16px;
    background: rgba(0,0,0,0.75);
    border-radius: 1px;
    box-shadow: 0 0 0 1px rgba(255,255,255,0.35);
    pointer-events: none;
    z-index: 2;
  }

  /* ── Desktop bars / mobile numbers toggle ── */
  .bars-desktop { display: block; }
  .stats-mobile { display: none; }

  @media (max-width: 600px) {
    .bars-desktop { display: none; }
    .desc-row .tree-icons { display: none; }

    .stats-mobile {
      display: flex;
      flex-direction: column;
      gap: 4px;
    }

    .mobile-stat {
      display: flex;
      justify-content: space-between;
      align-items: baseline;
    }
  }

  /* ── Source credit ─────────────────── */
  .source {
    font-family: var(--font);
    font-size: 10px;
    color: var(--color-text-light);
    line-height: 1.3;
    margin: 0;
  }

  /* ── Arrow ─────────────────────────── */
  .arrow {
    width: 0;
    height: 0;
    border-left: 7px solid transparent;
    border-right: 7px solid transparent;
    border-top: 7px solid var(--color-glass-heavy);
  }
</style>
