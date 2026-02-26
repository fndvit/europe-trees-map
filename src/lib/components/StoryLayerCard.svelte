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
    layers?: Layer[];
  }

  let { visible = false, layers = [] }: Props = $props();

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

  const visibleLayers = $derived(LAYERS.filter(l => layers.includes(l.id)));
</script>

<div class="story-card-wrap" class:visible>
  <div class="story-card-track">
    {#each visibleLayers as opt, i}
      {#if i > 0}<div class="story-card-divider"></div>{/if}
      <div class="story-card">
        <div class="card-swatch" style="background: {opt.gradient}"></div>
        <div class="card-text">
          <span class="card-label">{opt.label}</span>
          <span class="card-sublabel">{opt.sublabel}</span>
        </div>
      </div>
    {/each}
  </div>
</div>

<style>
  .story-card-wrap {
    position: absolute;
    bottom: 20px;
    right: 20px;
    z-index: 20;
    opacity: 0;
    transform: translateY(12px);
    transition: opacity 0.5s ease, transform 0.5s ease;
    pointer-events: none;
  }

  .story-card-wrap.visible {
    opacity: 1;
    transform: none;
  }

  .story-card-track {
    display: flex;
    flex-direction: row;
    align-items: stretch;
    background: var(--color-glass-light);
    backdrop-filter: var(--blur-light);
    -webkit-backdrop-filter: var(--blur-light);
    box-shadow: 0px -2px 16px rgba(0,0,0,0.1);
    border-radius: var(--radius-card);
    padding: 8px;
    gap: 0;
  }

  .story-card-divider {
    width: 1px;
    background: rgba(160, 160, 160, 0.4);
    flex-shrink: 0;
    margin: 4px 0;
  }

  .story-card {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 10px;
    padding: 6px 14px 6px 6px;
    cursor: default;
  }

  .card-swatch {
    width: 80px;
    height: 60px;
    border-radius: 6px;
    flex-shrink: 0;
    box-shadow: 0px 1px 8px rgba(0,0,0,0.1);
  }

  .card-text {
    display: flex;
    flex-direction: column;
    gap: 4px;
    width: 110px;
    text-align: left;
  }

  .card-label {
    font-family: var(--font);
    font-weight: 700;
    font-size: 12px;
    line-height: 1.22;
    color: var(--color-text-dark);
  }

  .card-sublabel {
    font-family: var(--font);
    font-weight: 300;
    font-size: 12px;
    line-height: 1.21;
    color: var(--color-text-dark);
    opacity: 0.7;
  }

  @media (max-width: 600px) {
    .story-card-wrap {
      right: 12px;
      bottom: 16px;
    }

    .card-swatch {
      width: 60px;
      height: 46px;
    }

    .card-text {
      width: 90px;
    }
  }
</style>
