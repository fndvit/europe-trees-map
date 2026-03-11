<script lang="ts">
  interface Props {
    visible?: boolean;
  }

  let { visible = false }: Props = $props();
</script>

<nav class="navbar" class:visible>
  <div class="nav-inner">
    <img src="/assets/badam_tree.png" alt="ViT" class="logo" />
    <span class="nav-title">
      <span class="title-full">The Most Detailed Map of <b>Europe's Trees</b></span>
      <span class="title-short"><b>Europe Trees Map</b></span>
    </span>
  </div>
  <a
    href="https://github.com/ViT-Vit/europe-trees-map"
    target="_blank"
    rel="noopener noreferrer"
    class="github-link"
    aria-label="View on GitHub"
  >
    <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor" aria-hidden="true">
      <path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0 0 24 12c0-6.63-5.37-12-12-12z"/>
    </svg>
  </a>
</nav>

<style>
  .navbar {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    z-index: 20;
    padding: 10px;
    padding-left: 35px;
    padding-right: 20px;
    opacity: 0;
    transform: translateY(-8px);
    transition:
      opacity 0.4s ease,
      transform 0.4s ease;
    pointer-events: none;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .github-link {
    color: var(--color-text-dark);
    opacity: 1;
    transition: opacity 0.2s ease;
    display: flex;
    align-items: center;
    flex-shrink: 0;
  }

  .github-link:hover {
    opacity: 0.5;
  }

  /* Touch screens: use tap instead of hover */
  @media (hover: none) {
    .github-link:hover {
      opacity: 1;
    }

    .github-link:active {
      opacity: 0.5;
    }
  }

  .navbar::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: -60px;
    background: linear-gradient(
      180deg,
      rgba(255, 255, 255, 1) 0%,
      rgba(255, 255, 255, 0.2) 100%
    );
    backdrop-filter: blur(1px);
    -webkit-backdrop-filter: blur(1px);
    -webkit-mask-image: linear-gradient(to bottom, black 40%, transparent 100%);
    mask-image: linear-gradient(to bottom, black 40%, transparent 100%);
    z-index: -1;
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
    flex: 1;
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

  .title-short {
    display: none;
  }

  /* ── Mobile ──────────────────────────── */
  @media (max-width: 600px) {
    .navbar {
      padding-left: 20px;
      padding-right: 14px;
    }

    .title-full {
      display: none;
    }

    .title-short {
      display: inline;
    }

    .github-link svg {
      width: 18px;
      height: 18px;
    }
  }
</style>
