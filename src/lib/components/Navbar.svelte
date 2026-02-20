<script lang="ts">
  interface Props {
    visible?: boolean;
  }

  let { visible = false }: Props = $props();
</script>

<nav class="navbar" class:visible>
  <div class="nav-inner">
    <img src="/assets/vit-logo-color.svg" alt="ViT" class="logo" />
    <span class="nav-title">The Most Detailed Map of Europe's Trees</span>
  </div>
</nav>

<style>
  .navbar {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    z-index: 20;
    background: linear-gradient(180deg, rgba(255,255,255,1) 0%, rgba(255,255,255,0) 100%);
    backdrop-filter: var(--blur-medium);
    -webkit-backdrop-filter: var(--blur-medium);
    padding: 10px 10px 30px;
    opacity: 0;
    transform: translateY(-8px);
    transition: opacity 0.4s ease, transform 0.4s ease;
    pointer-events: none;
  }

  .navbar.visible {
    opacity: 1;
    transform: none;
    pointer-events: auto;
  }

  .nav-inner {
    display: flex;
    align-items: center;
    gap: 16px;
  }

  .logo {
    width: 72px;
    height: 48px;
    flex-shrink: 0;
  }

  .nav-title {
    font-family: var(--font);
    font-weight: 200;
    font-size: 18px;
    line-height: 1;
    color: var(--color-text-dark);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
</style>
