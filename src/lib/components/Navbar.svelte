<script lang="ts">
  interface Props {
    visible?: boolean;
  }

  let { visible = false }: Props = $props();
</script>

<nav class="navbar" class:visible>
  <div class="nav-inner">
    <img src="/assets/badam_tree.png" alt="ViT" class="logo" />
    <span class="nav-title">The Most Detailed Map of <b>Europe's Trees</b></span
    >
  </div>
</nav>

<style>
  .navbar {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    z-index: 20;
    background: linear-gradient(
      180deg,
      rgba(255, 255, 255, 1) 0%,
      rgba(255, 255, 255, 0.2) 100%
    );
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
    padding: 10px;
    padding-left: 35px;
    opacity: 0;
    transform: translateY(-8px);
    transition:
      opacity 0.4s ease,
      transform 0.4s ease;
    pointer-events: none;
  }

  .navbar.visible {
    opacity: 1;
    transform: none;
    pointer-events: auto;
  }

  .nav-inner {
    display: flex;
    align-items: flex-end;
    gap: 5px;
  }

  .logo {
    width: 24px;
    height: 24px;
    flex-shrink: 0;
  }

  .nav-title {
    font-family: var(--font-title);
    font-weight: 200;
    font-size: 18px;
    line-height: 1;
    color: var(--color-text-dark);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .nav-title b {
    font-weight: 700;
  }

  /* ── Mobile ──────────────────────────── */
  @media (max-width: 600px) {
    .navbar {
      padding-left: 0px;
    }
  }
</style>
