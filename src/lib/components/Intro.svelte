<script lang="ts">
  interface Props {
    visible?: boolean;
    loading?: boolean;
  }

  let { visible = true, loading = true }: Props = $props();
</script>

<div class="intro" class:hidden={!visible}>
  <!-- ViT Logo top-left -->
  <img src="/assets/vit-logo-bw.svg" alt="ViT" class="vit-logo" />

  <!-- GitHub link top-right -->
  <a
    href="https://github.com/fndvit/europe-trees-map"
    target="_blank"
    rel="noopener noreferrer"
    class="github-link"
    aria-label="View on GitHub"
  >
    <img
      src="/assets/github-mark-white.svg"
      alt="GitHub"
      width="30"
      height="30"
    />
  </a>

  <!-- Main title -->
  <h1 class="title">
    The<br />
    Most Detailed<br />
    Map of<br />
    <b
      >Europe's<br />
      Trees</b
    >
  </h1>

  <!-- Bottom-center: single scroll / loading indicator -->
  <div class="scroll-indicator" aria-hidden="true">
    {#if loading}
      <div class="indicator-state loading-state">
        <img
          src="/assets/badam-tree.svg"
          alt=""
          class="indicator-icon loading-icon"
        />
        <span class="indicator-label">Loading map...</span>
      </div>
    {:else}
      <div class="indicator-state ready-state">
        <span class="indicator-label">Scroll to the map</span>
        <img
          src="/assets/pinus-tree.svg"
          alt=""
          class="indicator-icon pine-icon"
        />
      </div>
    {/if}
  </div>
</div>

<style>
  .intro {
    position: fixed;
    inset: 0;
    z-index: 100;
    background-color: #6eaf40;
    overflow: hidden;
    transition:
      opacity 0.8s ease,
      visibility 0.8s ease;
  }

  .intro.hidden {
    opacity: 0;
    visibility: hidden;
    pointer-events: none;
  }

  /* ── ViT logo ─────────────────────────── */
  .vit-logo {
    position: absolute;
    top: 10px;
    left: 10px;
    width: 108px;
    height: auto;
  }

  /* ── GitHub ──────────────────────────── */
  .github-link {
    position: absolute;
    top: 22px;
    right: 20px;
    opacity: 0.7;
    transition: opacity 0.2s;
  }

  .github-link:hover {
    opacity: 1;
  }

  /* ── Title ───────────────────────────── */
  .title {
    position: absolute;
    left: 48px;
    top: 50%;
    transform: translateY(-52%);
    font-family: var(--font-title);
    font-variation-settings:
      "opsz" 18,
      "wght" 400;
    font-size: 7rem;
    line-height: 6.5rem;
    color: #ffffff;
    letter-spacing: -0.01em;
  }

  .title b {
    font-variation-settings:
      "opsz" 12,
      "wght" 600;
  }

  /* ── Bottom-center scroll / loading indicator ── */
  .scroll-indicator {
    position: absolute;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 6px;
  }

  .indicator-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 6px;
    animation: fadeIn 0.4s ease;
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
      transform: translateY(6px);
    }
    to {
      opacity: 1;
      transform: none;
    }
  }

  .indicator-icon {
    width: 20px;
    opacity: 0.7;
  }

  /* Pine upside-down for "scroll to map" + bounce */
  .pine-icon {
    animation: bounce 2s ease-in-out infinite;
    transform-origin: center;
  }

  @keyframes bounce {
    0%,
    100% {
      transform: translateY(0);
    }
    50% {
      transform: translateY(-6px);
    }
  }

  /* Forest icon pulses while loading */
  .loading-icon {
    width: 60px !important;
    animation: pulse 1.8s ease-in-out infinite;
  }

  @keyframes pulse {
    0%,
    100% {
      opacity: 0.4;
    }
    50% {
      opacity: 0.8;
    }
  }

  .indicator-label {
    font-family: var(--font);
    font-weight: 300;
    font-size: 12px;
    line-height: 1.4;
    color: rgba(255, 255, 255, 0.65);
    text-align: center;
    letter-spacing: 0.04em;
    white-space: nowrap;
  }

  /* ── Mobile ──────────────────────────── */
  @media (max-width: 600px) {
    .title {
      left: 22px;
      top: 149px;
      padding-right: 22px;
      transform: none;
      line-height: 4.5rem;
      font-size: clamp(56px, 18vw, 90px);
    }

    .vit-logo {
      width: 90px;
    }
  }
</style>
