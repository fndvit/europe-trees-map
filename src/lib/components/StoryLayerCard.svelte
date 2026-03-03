<script lang="ts">
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
    layers?: Layer[];
    highlightedLayer?: Layer;
  }

  let { visible = false, layers = [], highlightedLayer }: Props = $props();

  const LAYER_MAP: Record<Layer, LayerOption> = {
    density: {
      id: 'density',
      label: 'Tree cover density',
      sublabel: 'Sparse cover in yellow, denser in green',
      image: '/assets/layer-density.png'
    },
    'forest-type': {
      id: 'forest-type',
      label: 'Forest type',
      sublabel: 'Two hues show forest type: broadleaved trees and conifers.',
      image: '/assets/layer-forest-type.png'
    },
    broadleaved: {
      id: 'broadleaved',
      label: 'Broadleaves',
      sublabel: 'Also known as hardwoods: oaks, birch, beech ...',
      image: '/assets/layer-broadleaved.png'
    },
    conifers: {
      id: 'conifers',
      label: 'Conifers',
      sublabel: 'Known for their needle-like leaves: pines, spruces, cypresses ...',
      image: '/assets/layer-conifers.png'
    },
    'type-density': {
      id: 'type-density',
      label: 'Forest type + density',
      sublabel: 'Two hues show forest type, lighter shades equal less density',
      image: '/assets/layer-type-density..png'
    }
  };

  // Preserve order from layers prop
  let visibleLayers = $derived(
    layers.map(id => LAYER_MAP[id]).filter(Boolean) as LayerOption[]
  );
</script>

<div class="story-card-wrap" class:visible>
  <div class="story-card-track">
    {#each visibleLayers as opt (opt.id)}
      <div
        class="story-card"
        class:active={opt.id === highlightedLayer}
        class:dimmed={!!highlightedLayer && opt.id !== highlightedLayer}
        in:fly={{ y: -18, duration: 380 }}
        out:fly={{ y: -18, duration: 220 }}
      >
        <div class="card-swatch" style="background-image: url({opt.image})"></div>
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
    bottom: 68px;
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
    flex-direction: column-reverse;
    align-items: stretch;
    gap: 2px;
  }

  .story-card {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 10px;
    padding: 6px 14px 6px 6px;
    background: var(--color-glass-light);
    backdrop-filter: var(--blur-light);
    -webkit-backdrop-filter: var(--blur-light);
    box-shadow: 0px -2px 16px rgba(0, 0, 0, 0.1);
    border-radius: var(--radius-card);
    border: 0.5px solid transparent;
    cursor: default;
    transition:
      opacity 0.35s ease,
      background 0.3s ease,
      border-color 0.3s ease;
  }

  .story-card.active {
    background: var(--color-glass-heavy);
    border-color: rgba(160, 160, 160, 0.4);
  }

  .story-card.dimmed {
    opacity: 0.5;
  }

  .card-swatch {
    width: 80px;
    height: 60px;
    border-radius: 6px;
    flex-shrink: 0;
    box-shadow: 0px 1px 8px rgba(0, 0, 0, 0.1);
    background-size: cover;
    background-position: center;
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
