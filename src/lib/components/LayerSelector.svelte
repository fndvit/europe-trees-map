<script lang="ts">
  import { onMount } from 'svelte';

  type Layer = 'density' | 'type-density' | 'broadleaved' | 'conifers' | 'forest-type';

  interface LayerOption {
    id: Layer;
    label: string;
    sublabel: string;
    gradient: string;
  }

  interface Props {
    visible?: boolean;
    activeLayer?: Layer;
    onchange?: (layer: Layer) => void;
  }

  let { visible = false, activeLayer = 'broadleaved', onchange }: Props = $props();

  const LAYERS: LayerOption[] = [
    {
      id: 'type-density',
      label: 'Forest type + density',
      sublabel: 'Two hues show forest type, lighter shades equal less density',
      gradient: 'linear-gradient(90deg, rgba(199,180,71,0.3) 0%, #c7b447 45%, #07523f 100%)'
    },
    {
      id: 'conifers',
      label: 'Conifers',
      sublabel: 'Known for their needle-like leaves: pines, spruces, cypresses ...',
      gradient: 'linear-gradient(90deg, rgba(7,82,63,0.1) 0%, rgba(7,82,63,1) 100%)'
    },
    {
      id: 'broadleaved',
      label: 'Broadleaves',
      sublabel: 'Also known as hardwoods: oaks, birch, beech ...',
      gradient: 'linear-gradient(90deg, rgba(199,180,71,0.1) 0%, rgba(199,180,71,1) 100%)'
    },
    {
      id: 'forest-type',
      label: 'Forest type',
      sublabel: 'Two hues show forest type: broadleaved trees and conifers.',
      gradient: 'linear-gradient(90deg, #c7b447 0%, #07523f 100%)'
    },
    {
      id: 'density',
      label: 'Tree cover density',
      sublabel: 'Sparse cover in yellow, denser in green',
      gradient: 'linear-gradient(90deg, #ffec81 0%, #005f00 100%)'
    }
  ];

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
    if (visible) requestAnimationFrame(updateScrollState);
  });

  onMount(() => {
    updateScrollState();
    window.addEventListener('resize', updateScrollState);
    return () => window.removeEventListener('resize', updateScrollState);
  });
</script>

<div class="selector-wrap" class:visible>
  <button
    class="chevron left"
    class:hidden={!canScrollLeft}
    onclick={() => scrollBy(-220)}
    aria-label="Scroll left"
  >
    <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><polyline points="15 18 9 12 15 6"/></svg>
  </button>

  <div class="selector-track" bind:this={trackEl} onscroll={updateScrollState}>
    {#each LAYERS as opt}
      <button
        class="layer-btn"
        class:active={activeLayer === opt.id}
        onclick={() => onchange?.(opt.id)}
      >
        <div class="layer-swatch" style="background: {opt.gradient}"></div>
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
    bottom: 45px;
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
    
    transition: background 0.2s;
    flex-shrink: 0;
    scroll-snap-align: start;
    position: relative;
    border-right: 1px dashed rgba(160, 160, 160, 0.4);
  }

  .layer-btn:last-child {
    border-right: none;
  }

  .layer-btn.active,
  .layer-btn:has(+ .layer-btn.active) {
    border-right: none;
  }

  .layer-btn.active {
    background: rgba(255, 255, 255, 0.90);
    box-shadow: 0 0 16px 0 rgba(0, 0, 0, 0.25);
    border-radius: 0;
    margin: 0;
  }

  .layer-swatch {
    width: 120px;
    height: 90px;
    border-radius: 8px;
    flex-shrink: 0;
    box-shadow: 0px 1px 8px rgba(0,0,0,0.1);
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
