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
    density: number;    // 0–100 from Copernicus TCD; -1 = loading
    leafType: number;   // 1=broadleaved, 2=coniferous, 0=unknown
    loading?: boolean;
    x: number;
    y: number;
  }

  interface Props {
    data?: TooltipData | null;
    onclose?: () => void;
  }

  let { data = null, onclose }: Props = $props();

  const W = 240;
  const H = 160;

  let cx = $derived(
    data
      ? Math.min(Math.max(data.x, W / 2 + 12), (typeof window !== 'undefined' ? window.innerWidth : 1200) - W / 2 - 12)
      : 0
  );
  let cy = $derived(data ? Math.max(data.y - H - 16, 60) : 0);

  function densityLabel(d: number): string {
    if (d >= 70) return 'Dense forest';
    if (d >= 30) return 'Moderate forest';
    if (d >= 10) return 'Sparse trees';
    return 'No significant tree cover';
  }

  function densityCategory(d: number): string {
    if (d === 0) return 'no';
    if (d <= 10) return 'very sparse';
    if (d <= 35) return 'sparse';
    if (d <= 70) return 'moderate';
    if (d <= 80) return 'dense';
    return 'very dense';
  }

  function leafTypeLabel(leafType: number, density: number): string {
    if (density === 0) return 'No tree cover.';
    if (leafType === 1) return `A ${densityCategory(density)} canopy, mostly broadleaved trees.`;
    if (leafType === 2) return `A ${densityCategory(density)} canopy, mostly coniferous trees.`;
    return `A ${densityCategory(density)} canopy.`;
  }

  let fillPct = $derived(data ? Math.max(1, data.density) : 0);

  // ── Tree icon drawing ──────────────────────────────────────────────────────
  // Pick a seeded-random icon from a pool based on tree type
  function pickIcon(type: 'broadleaved' | 'coniferous', seed: number): string {
    const pool = type === 'coniferous' ? [con1, con2, con3] : [bro1, bro2, bro3];
    return pool[seed % 3];
  }

  // Build a list of tree types to draw based on density and leafType
  function buildTreeRow(density: number, leafType: number): Array<{ type: 'broadleaved' | 'coniferous'; seed: number }> {
    if (density === 0) return [];
    const count = Math.max(1, Math.min(Math.round(density / 10), 10));
    let types: Array<'broadleaved' | 'coniferous'>;
    if (leafType === 2) {
      types = Array(count).fill('coniferous');
    } else if (leafType === 1) {
      types = Array(count).fill('broadleaved');
    } else {
      // unknown type: mix
      const half = Math.round(count / 2);
      types = [...Array(half).fill('coniferous'), ...Array(count - half).fill('broadleaved')];
    }
    return types.map((type, i) => ({ type, seed: i }));
  }

  let treeRow = $derived(
    data && !data.loading ? buildTreeRow(data.density, data.leafType) : []
  );
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

      {#if data.loading}
        <div class="skeletons">
          <div class="skeleton tree-skel"></div>
          <div class="skeleton wide"></div>
          <div class="skeleton narrow"></div>
        </div>
      {:else}
        <!-- Tree icon row -->
        {#if treeRow.length > 0}
          <div class="tree-row" aria-hidden="true">
            {#each treeRow as tree}
              <span class="tree-icon">
                {@html pickIcon(tree.type, tree.seed)}
              </span>
            {/each}
          </div>
        {/if}

        <!-- Natural language sentence -->
        <p class="nl-sentence">{leafTypeLabel(data.leafType, data.density)}</p>

        <div class="stat-row">
          <span class="stat-lbl">Tree cover density</span>
          <span class="stat-val">{data.density}%</span>
        </div>

        <div class="bar-track">
          <div class="bar-fill" style="width:{fillPct}%"></div>
        </div>

        <p class="source">Copernicus HRL TCD/DLT 2018 · © EEA</p>
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
    width: 240px;
    box-shadow: var(--shadow-tooltip);
    position: relative;
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

  /* ── Loading skeletons ─────────────── */
  .skeletons { display: flex; flex-direction: column; gap: 8px; }

  .skeleton {
    height: 10px;
    border-radius: 5px;
    background: linear-gradient(90deg, rgba(0,0,0,0.06) 25%, rgba(0,0,0,0.12) 50%, rgba(0,0,0,0.06) 75%);
    background-size: 200% 100%;
    animation: shimmer 1.2s ease-in-out infinite;
  }
  .skeleton.tree-skel { height: 32px; width: 80px; }
  .skeleton.wide    { width: 100%; }
  .skeleton.narrow  { width: 60%; }

  @keyframes shimmer {
    0%   { background-position: 200% 0; }
    100% { background-position: -200% 0; }
  }

  /* ── Tree icon row ─────────────────── */
  .tree-row {
    display: flex;
    align-items: flex-end;
    gap: 2px;
    margin-bottom: 8px;
    height: 32px;
  }

  .tree-icon {
    display: flex;
    align-items: flex-end;
    flex: 1;
    max-width: 24px;
  }

  /* Scale SVG to fill its container, and invert colors for dark background */
  .tree-icon :global(svg) {
    width: 100%;
    height: 32px;
    filter: invert(1);
    overflow: visible;
  }

  /* ── NL sentence ───────────────────── */
  .nl-sentence {
    font-family: var(--font);
    font-size: 12px;
    font-style: italic;
    color: var(--color-text-medium);
    margin-bottom: 10px;
    line-height: 1.4;
  }

  /* ── Data ──────────────────────────── */
  .stat-row {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    margin-bottom: 6px;
  }

  .stat-lbl {
    font-family: var(--font);
    font-weight: 300;
    font-size: 12px;
    color: var(--color-text-dark);
  }

  .stat-val {
    font-family: var(--font);
    font-weight: 700;
    font-size: 13px;
    color: var(--color-text-dark);
  }

  .bar-track {
    width: 100%;
    height: 8px;
    border-radius: 4px;
    background: rgba(0,0,0,0.08);
    overflow: hidden;
    margin-bottom: 8px;
  }

  .bar-fill {
    height: 100%;
    border-radius: 4px;
    background: linear-gradient(90deg, #ffec81, #4a9a20, #005f00);
    transition: width 0.4s ease;
  }

  .source {
    font-family: var(--font);
    font-size: 10px;
    color: var(--color-text-light);
    line-height: 1.3;
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
