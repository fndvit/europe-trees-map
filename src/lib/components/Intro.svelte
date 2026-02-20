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
    href="https://github.com"
    target="_blank"
    rel="noopener noreferrer"
    class="github-link"
    aria-label="View on GitHub"
  >
    <img src="/assets/github-mark-white.svg" alt="GitHub" width="30" height="30" />
  </a>

  <!-- Main title -->
  <h1 class="title">
    The<br />
    Most Detailed<br />
    Map of<br />
    Europe's Trees
  </h1>

  <!-- Bottom-right: tree + scroll CTA -->
  <div class="corner">
    {#if loading}
      <!-- Loading state: animated tree + "Loading map" label -->
      <div class="state loading-state">
        <img src="/assets/badam-tree.svg" alt="" class="badam-tree" />
        <span class="label">Loading map</span>
      </div>
    {:else}
      <!-- Ready state: pinus tree + "Scroll to the map" -->
      <div class="state ready-state">
        <img src="/assets/pinus-tree.svg" alt="" class="pinus-tree" />
        <span class="label">Scroll to<br />the map</span>
      </div>
    {/if}
  </div>

  <!-- Scroll arrow — always visible so user knows they can scroll -->
  <div class="scroll-arrow" class:dimmed={loading} aria-hidden="true">
    <svg
      xmlns="http://www.w3.org/2000/svg"
      width="28"
      height="28"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      stroke-width="1.5"
      stroke-linecap="round"
      stroke-linejoin="round"
    >
      <path d="M12 5v14M5 12l7 7 7-7" />
    </svg>
  </div>
</div>

<style>
  .intro {
    position: fixed;
    inset: 0;
    z-index: 100;
    background-color: var(--color-bg-intro);
    overflow: hidden;
    transition: opacity 0.8s ease, visibility 0.8s ease;
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

  .github-link:hover { opacity: 1; }

  /* ── Title ───────────────────────────── */
  .title {
    position: absolute;
    left: 48px;
    top: 50%;
    transform: translateY(-52%);
    font-family: var(--font);
    font-weight: 200;
    font-size: clamp(52px, 7.5vw, 108px);
    line-height: 0.93;
    color: var(--color-white);
    letter-spacing: -0.01em;
  }

  /* ── Bottom-right corner ─────────────── */
  .corner {
    position: absolute;
    bottom: 72px;
    right: 52px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
  }

  .state {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    animation: fadeIn 0.4s ease;
  }

  @keyframes fadeIn {
    from { opacity: 0; transform: translateY(6px); }
    to   { opacity: 1; transform: none; }
  }

  .badam-tree {
    width: 56px;
    opacity: 0.65;
    animation: pulse 1.8s ease-in-out infinite;
  }

  @keyframes pulse {
    0%, 100% { opacity: 0.4; }
    50%       { opacity: 0.8; }
  }

  .pinus-tree {
    width: 24px;
    opacity: 0.85;
  }

  .label {
    font-family: var(--font);
    font-weight: 300;
    font-size: 13px;
    line-height: 1.5;
    color: rgba(255, 255, 255, 0.65);
    text-align: center;
    letter-spacing: 0.02em;
  }

  /* ── Scroll arrow ────────────────────── */
  .scroll-arrow {
    position: absolute;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    color: rgba(255, 255, 255, 0.5);
    animation: bounce 2s ease-in-out infinite;
    transition: color 0.4s;
  }

  .scroll-arrow.dimmed {
    color: rgba(255, 255, 255, 0.2);
  }

  @keyframes bounce {
    0%, 100% { transform: translateX(-50%) translateY(0); }
    50%       { transform: translateX(-50%) translateY(7px); }
  }

  /* ── Mobile ──────────────────────────── */
  @media (max-width: 600px) {
    .title {
      left: 22px;
      top: 149px;
      transform: none;
      font-size: clamp(72px, 20vw, 90px);
    }

    .vit-logo { width: 90px; }

    .corner {
      right: auto;
      bottom: 80px;
      left: 50%;
      transform: translateX(-50%);
    }
  }
</style>
