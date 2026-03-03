<script lang="ts">
  import { onMount } from 'svelte';
  import { fly } from 'svelte/transition';

  type Layer = 'density' | 'type-density' | 'broadleaved' | 'conifers' | 'forest-type';

  interface LayerOption {
    id: Layer;
    label: string;
    sublabel: string;
    image: string;
  }

  interface Props {
    visible?: boolean;
    activeLayer?: Layer;
    layers?: Layer[];                      // when provided, only show these layers (story mode)
    onchange?: (layer: Layer) => void;
  }

  let { visible = false, activeLayer = 'broadleaved', layers = undefined, onchange }: Props = $props();

  // Order matches the story build-up: density → forest-type → broadleaved → conifers → type-density
  // so the newly introduced (active) layer is always on the right
  const ALL_LAYERS: LayerOption[] = [
    {
      id: 'density',
      label: 'Tree cover density',
      sublabel: 'Sparse cover in yellow, denser in green',
      image: '/assets/layer-density.png'
    },
    {
      id: 'forest-type',
      label: 'Forest type',
      sublabel: 'Two hues show forest type: broadleaved trees and conifers.',
      image: '/assets/layer-forest-type.png'
    },
    {
      id: 'broadleaved',
      label: 'Broadleaves',
      sublabel: 'Also known as hardwoods: oaks, birch, beech ...',
      image: '/assets/layer-broadleaved.png'
    },
    {
      id: 'conifers',
      label: 'Conifers',
      sublabel: 'Known for their needle-like leaves: pines, spruces, cypresses ...',
      image: '/assets/layer-conifers.png'
    },
    {
      id: 'type-density',
      label: 'Forest type + density',
      sublabel: 'Two hues show forest type, lighter shades equal less density',
      image: '/assets/layer-type-density..png'
    }
  ];

  // Filter to provided layers (preserving ALL_LAYERS order), or show all
  let visibleLayers = $derived(
    layers != null ? ALL_LAYERS.filter(l => layers!.includes(l.id)) : ALL_LAYERS
  );

  let isInteractive = $derived(!!onchange);

  let trackEl: HTMLDivElement;
  let canScrollLeft = $state(false);
  let canScrollRight = $state(false);

  function updateScrollState() {
    if (!trackEl) return;
    canScrollLeft = trackEl.scrollLeft > 4;
    canScrollRight = trackEl.scrollLeft + trackEl.clientWidth < trackEl.scrollWidth - 4;
  }

  function scrollBy(delta: number) {
    trackEl?.scrollBy({ left: delta, behavior: 'smooth' });
  }

  $effect(() => {
    visible; isInteractive; // re-run on visibility or mode change
    requestAnimationFrame(updateScrollState);
  });

  // Scroll to the rightmost card whenever the layer list changes
  $effect(() => {
    visibleLayers;
    requestAnimationFrame(() => {
      if (trackEl) trackEl.scrollTo({ left: trackEl.scrollWidth, behavior: 'smooth' });
    });
  });

  onMount(() => {
    updateScrollState();
    window.addEventListener('resize', updateScrollState);
    return () => window.removeEventListener('resize', updateScrollState);
  });
</script>

<div class="selector-wrap" class:visible class:compact={!isInteractive}>
    <button
      class="chevron left"
      class:hidden={!canScrollLeft}
      onclick={() => scrollBy(-220)}
      aria-label="Scroll left"
    >
      <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><polyline points="15 18 9 12 15 6"/></svg>
    </button>

  <div class="selector-track" bind:this={trackEl} onscroll={updateScrollState}>
    {#each visibleLayers as opt (opt.id)}
      <button
        class="layer-btn"
        class:active={activeLayer === opt.id}
        class:dimmed={!isInteractive && activeLayer !== opt.id}
        onclick={() => onchange?.(opt.id)}
        in:fly={{ x: 20, duration: 350 }}
        out:fly={{ x: 20, duration: 200 }}
      >
        <div class="layer-swatch" style="background-image: url({opt.image})"></div>
        <div class="layer-text">
          <span class="layer-label">{opt.label}</span>
          <span class="layer-sublabel">{opt.sublabel}</span>
        </div>
      </button>
    {/each}
  </div>

    <button
      class="chevron right"
      class:hidden={!canScrollRight}
      onclick={() => scrollBy(220)}
      aria-label="Scroll right"
    >
      <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><polyline points="9 18 15 12 9 6"/></svg>
    </button>
</div>

<style>
  .selector-wrap {
    position: absolute;
    bottom: 68px;
    left: 50%;
    transform: translateX(-50%);
    width: calc(100% - 40px);
    max-width: 1100px;
    z-index: 20;
    border-radius: 12px;
    background: rgba(255, 255, 255, 0.35);
    box-shadow: 0 -2px 16px 0 rgba(0, 0, 0, 0.10);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    overflow: hidden;
    opacity: 0;
    transition: opacity 0.5s ease;
    pointer-events: none;
  }

  /* Story mode: right-anchored, shrinks to fit content */
  .selector-wrap.compact {
    left: auto;
    right: max(20px, calc((100% - 1100px) / 2));
    transform: none;
    width: fit-content;
  }

  .selector-wrap.visible {
    opacity: 1;
    pointer-events: auto;
  }

  .selector-track {
    display: flex;
    flex-direction: row;
    align-items: stretch;
    overflow-x: auto;
    scroll-snap-type: x mandatory;
    -webkit-overflow-scrolling: touch;
    scrollbar-width: none;
    gap: 2px;
  }

  .selector-track::-webkit-scrollbar {
    display: none;
  }

  .chevron {
    position: absolute;
    top: 0;
    bottom: 0;
    z-index: 2;
    width: 72px;
    border: none;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    color: rgba(0, 0, 0, 0.75);
    transition: opacity 0.3s ease, color 0.2s ease;
  }

  .chevron.left {
    left: 0;
    background: linear-gradient(90deg, rgba(255, 255, 255, 1) 10%, rgba(255, 255, 255, 0) 100%);
    justify-content: flex-start;
    padding-left: 6px;
  }

  .chevron.right {
    right: 0;
    background: linear-gradient(270deg, rgba(255, 255, 255, 1) 10%, rgba(255, 255, 255, 0.00) 100%);
    justify-content: flex-end;
    padding-right: 6px;
  }

  .chevron.hidden {
    opacity: 0;
    pointer-events: none;
  }

  .chevron:hover {
    color: rgba(0, 0, 0, 0.65);
  }

  .layer-btn {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 12px;
    padding: 12px 24px 12px 12px;
    background: none;
    border: none;
    cursor: pointer;
    transition: background 0.2s, opacity 0.3s ease;
    flex-shrink: 0;
    scroll-snap-align: start;
    position: relative;
  }

  .layer-btn::after {
    content: '';
    position: absolute;
    right: 0;
    top: 0;
    width: 1px;
    height: 100%;
    background: linear-gradient(
      to bottom,
      transparent 0%,
      transparent 10%,
      rgba(160, 160, 160, 0.4) 15%,
      rgba(160, 160, 160, 0.4) 85%,
      transparent 90%,
      transparent 100%
    );
    background-size: 1px 100%;
    background-repeat: repeat-y;
    mask-image: repeating-linear-gradient(
      to bottom,
      black 0px,
      black 4px,
      transparent 4px,
      transparent 8px
    );
  }

  .layer-btn:last-child::after {
    display: none;
  }

  .layer-btn.active::after,
  .layer-btn:has(+ .layer-btn.active)::after {
    display: none;
  }

  .layer-btn.active {
    background: rgba(255, 255, 255, 0.90);
    box-shadow: 0 0 16px 0 rgba(0, 0, 0, 0.25);
    border-radius: 0;
    margin: 0;
  }

  /* Story mode: dim non-active layers */
  .layer-btn.dimmed {
    opacity: 0.45;
    cursor: default;
  }

  .layer-swatch {
    width: 120px;
    height: 90px;
    border-radius: 8px;
    flex-shrink: 0;
    box-shadow: 0px 1px 8px rgba(0,0,0,0.1);
    background-size: cover;
    background-position: center;
  }

  .layer-text {
    display: flex;
    flex-direction: column;
    gap: 6px;
    width: 141px;
    text-align: left;
  }

  .layer-label {
    font-family: var(--font);
    font-weight: 700;
    font-size: 14px;
    line-height: 1.22;
    color: var(--color-text-dark);
  }

  .layer-sublabel {
    font-family: var(--font);
    font-weight: 300;
    font-size: 14px;
    line-height: 1.21;
    color: var(--color-text-dark);
  }

  .layer-btn:not(.active) .layer-label {
    color: #aaa;
    opacity: 0.8;
  }
</style>
