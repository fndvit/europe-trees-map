<script lang="ts">
  interface Props {
    text?: string;
    visible?: boolean;
    icon?: string;
    progress?: number; // 0–1 within current scroll step
  }

  let { text = '', visible = true, icon = '', progress = 0 }: Props = $props();

  // Card is centered vertically (top:50% + translateY(-50%)).
  // Scroll animation adds ±100vh so it enters from off-screen below and exits off-screen above.
  // -50%  → centers the card on its own height
  // ±100vh → full-screen travel driven by scroll progress
  let yOffset = $derived(
    `translateY(calc(-50% + ${(1 - progress * 2).toFixed(4)} * 100vh))`
  );
</script>

<div
  class="info-card"
  class:visible
  style="transform: {yOffset}"
>
  {#if icon}
    <img src={icon} alt="" class="card-icon" />
  {/if}
  <p class="card-text">{@html text}</p>
</div>

<style>
  .info-card {
    position: absolute;
    left: 20px;
    top: 50%;          /* vertical anchor — transform handles the rest */
    max-width: 400px;
    padding: 40px;
    background: var(--color-glass-heavy);
    backdrop-filter: var(--blur-heavy);
    -webkit-backdrop-filter: var(--blur-heavy);
    box-shadow: var(--shadow-card);
    border-radius: var(--radius-card);
    opacity: 0;
    /* transform is driven entirely by scroll progress via inline style */
    transition: opacity 0.4s ease;
    pointer-events: none;
    z-index: 10;
  }

  .info-card.visible {
    opacity: 1;
    pointer-events: auto;
  }

  .card-icon {
    width: 100%;
    max-width: 320px;
    margin-bottom: 14px;
    display: block;
  }

  .card-text {
    font-family: var(--font);
    font-weight: 300;
    font-size: 20px;
    line-height: 1.4;
    color: var(--color-text-dark);
  }

  .card-text :global(strong) {
    font-weight: 600;
  }

  /* Mobile */
  @media (max-width: 600px) {
    .info-card {
      left: 16px;
      right: 16px;
      max-width: none;
      padding: 28px 24px;
    }

    .card-text {
      font-size: 17px;
    }
  }
</style>
