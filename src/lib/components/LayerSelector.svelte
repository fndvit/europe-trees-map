<script lang="ts">
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

</script>

<div class="selector-wrap" class:visible>
  <div class="selector-track">
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
</div>

<style>
  .selector-wrap {
    position: absolute;
    bottom: 64px;
    left: 20px;
    right: 20px;
    z-index: 20;
    opacity: 0;
    transform: translateY(12px);
    transition: opacity 0.5s ease, transform 0.5s ease;
    pointer-events: none;
    overflow: hidden;
  }

  .selector-wrap.visible {
    opacity: 1;
    transform: none;
    pointer-events: auto;
  }

  .selector-track {
    display: flex;
    flex-direction: row;
    align-items: stretch;
    gap: 0;
    overflow-x: auto;
    scroll-snap-type: x mandatory;
    -webkit-overflow-scrolling: touch;
    scrollbar-width: none;
    background: var(--color-glass-light);
    backdrop-filter: var(--blur-light);
    -webkit-backdrop-filter: var(--blur-light);
    box-shadow: 0px -2px 16px rgba(0,0,0,0.1);
    border-radius: var(--radius-card);
    padding: 10px;
    gap: 2px;
  }

  .selector-track::-webkit-scrollbar {
    display: none;
  }

  .layer-btn {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 12px;
    padding: 8px 24px 8px 8px;
    background: none;
    border: none;
    cursor: pointer;
    border-radius: 8px;
    transition: background 0.2s;
    flex-shrink: 0;
    scroll-snap-align: start;
    position: relative;
    border-right: 1px dashed rgba(160, 160, 160, 0.4);
  }

  .layer-btn:last-child {
    border-right: none;
  }

  .layer-btn.active {
    background: rgba(255,255,255,0.9);
    box-shadow: 0px 0px 16px rgba(0,0,0,0.25);
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
