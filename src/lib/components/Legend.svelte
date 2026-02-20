<script lang="ts">
  type Layer = 'density' | 'type-density' | 'broadleaved';

  interface LegendItem {
    label: string;
    sublabel: string;
    gradient?: string;
    colors?: string[];
    active?: boolean;
  }

  interface Props {
    layer?: Layer;
    visible?: boolean;
    activeIndex?: number;
  }

  let { layer = 'density', visible = true, activeIndex = 1 }: Props = $props();

  const LEGENDS: Record<Layer, LegendItem[]> = {
    density: [
      {
        label: 'Tree cover density',
        sublabel: 'Sparse cover in yellow, denser in green.',
        gradient: 'linear-gradient(90deg, #ffec81 0%, #005f00 100%)'
      }
    ],
    'type-density': [
      {
        label: 'Forest type + density',
        sublabel: 'Two hues show forest type, lighter shades equal less density',
        gradient: 'linear-gradient(90deg, rgba(199,180,71,0.2) 0%, #c7b447 50%, #07523f 100%)'
      },
      {
        label: 'Tree cover density',
        sublabel: 'Sparse cover in yellow, denser in green',
        gradient: 'linear-gradient(90deg, #ffec81 0%, #005f00 100%)'
      }
    ],
    broadleaved: [
      {
        label: 'Tree cover density',
        sublabel: 'Sparse cover in yellow, denser in green',
        gradient: 'linear-gradient(90deg, #ffec81 0%, #005f00 100%)'
      }
    ]
  };

  let items = $derived(LEGENDS[layer] ?? LEGENDS.density);
</script>

<div class="legend-wrap" class:visible>
  {#each items as item, i}
    <div class="legend-item" class:active={i === activeIndex - 1 || items.length === 1}>
      <div class="swatch" style="background: {item.gradient}"></div>
      <div class="legend-text">
        <span class="legend-label">{item.label}</span>
        <span class="legend-sublabel">{item.sublabel}</span>
      </div>
    </div>
  {/each}
</div>

<style>
  .legend-wrap {
    position: absolute;
    bottom: 20px;
    right: 20px;
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 2px;
    opacity: 0;
    transform: translateY(8px);
    transition: opacity 0.5s ease, transform 0.5s ease;
    pointer-events: none;
    z-index: 10;
  }

  .legend-wrap.visible {
    opacity: 1;
    transform: none;
    pointer-events: auto;
  }

  .legend-item {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 12px;
    padding: 8px 48px 8px 8px;
    background: var(--color-glass-light);
    backdrop-filter: var(--blur-light);
    -webkit-backdrop-filter: var(--blur-light);
    box-shadow: var(--shadow-legend);
    border-radius: 8px;
    border: 0.5px solid transparent;
    transition: background 0.3s, border-color 0.3s;
    min-width: 240px;
  }

  .legend-item.active {
    background: var(--color-glass-heavy);
    border-color: rgba(160, 160, 160, 0.4);
  }

  .swatch {
    width: 120px;
    height: 90px;
    border-radius: 8px;
    flex-shrink: 0;
    box-shadow: 0px 1px 8px rgba(0,0,0,0.1);
  }

  .legend-text {
    display: flex;
    flex-direction: column;
    gap: 6px;
    width: 141px;
  }

  .legend-label {
    font-family: var(--font);
    font-weight: 700;
    font-size: 14px;
    line-height: 1.22;
    color: var(--color-text-dark);
  }

  .legend-sublabel {
    font-family: var(--font);
    font-weight: 300;
    font-size: 14px;
    line-height: 1.21;
    color: var(--color-text-dark);
  }

  /* Mobile: compact */
  @media (max-width: 600px) {
    .legend-wrap {
      bottom: 12px;
      right: 12px;
    }

    .legend-item {
      padding: 8px 12px;
      min-width: 200px;
    }

    .swatch {
      width: 60px;
      height: 45px;
    }
  }
</style>
