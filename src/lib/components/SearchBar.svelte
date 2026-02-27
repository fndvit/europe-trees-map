<script lang="ts">
  interface SearchResult {
    display_name: string;
    lat: string;
    lon: string;
  }

  interface Props {
    visible?: boolean;
    onflyto?: (coords: { lat: number; lng: number; name: string }) => void;
    dark?: boolean;
    exploration?: boolean;
  }

  let { visible = true, onflyto, dark = false, exploration = false }: Props = $props();

  let query = $state('');
  let results = $state<SearchResult[]>([]);
  let open = $state(false);
  let loading = $state(false);
  let debounceTimer: ReturnType<typeof setTimeout>;

  async function search(q: string) {
    if (!q.trim() || q.length < 2) {
      results = [];
      open = false;
      return;
    }
    loading = true;
    try {
      const url = `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(q)}&format=json&limit=5&viewbox=-25,71,45,34&bounded=0`;
      const res = await fetch(url, {
        headers: { 'Accept-Language': 'en', 'User-Agent': 'EuropeTreesMap/1.0' }
      });
      results = await res.json();
      open = results.length > 0;
    } catch {
      results = [];
    } finally {
      loading = false;
    }
  }

  function handleInput() {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => search(query), 400);
  }

  function selectResult(r: SearchResult) {
    query = r.display_name.split(',')[0];
    open = false;
    onflyto?.({ lat: parseFloat(r.lat), lng: parseFloat(r.lon), name: query });
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Escape') {
      open = false;
      query = '';
    }
  }
</script>

<div class="search-wrap" class:visible class:dark class:exploration>
  <div class="search-bar" class:active={open}>
    <svg class="search-icon" xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
      <circle cx="11" cy="11" r="8"/><path d="m21 21-4.35-4.35"/>
    </svg>
    <input
      type="text"
      placeholder="Search by location"
      bind:value={query}
      oninput={handleInput}
      onkeydown={handleKeydown}
      onfocus={() => { if (results.length) open = true; }}
      class="search-input"
    />
    {#if loading}
      <div class="spinner" aria-label="Searching..."></div>
    {/if}
    {#if query}
      <button class="clear-btn" onclick={() => { query = ''; results = []; open = false; }} aria-label="Clear">
        <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
          <path d="M18 6 6 18M6 6l12 12"/>
        </svg>
      </button>
    {/if}
  </div>

  {#if open && results.length > 0}
    <ul class="results">
      {#each results as r}
        <li>
          <button class="result-item" onclick={() => selectResult(r)}>
            <svg xmlns="http://www.w3.org/2000/svg" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="M20 10c0 6-8 12-8 12s-8-6-8-12a8 8 0 0 1 16 0Z"/><circle cx="12" cy="10" r="3"/>
            </svg>
            <span>{r.display_name}</span>
          </button>
        </li>
      {/each}
    </ul>
  {/if}
</div>

<style>
  .search-wrap {
    position: absolute;
    top: 72px;
    right: 20px;
    z-index: 20;
    opacity: 0;
    transition: opacity 0.4s ease;
    pointer-events: none;
    min-width: 220px;
  }

  .search-wrap.exploration {
    right: auto;
    left: 50%;
    transform: translateX(-50%);
    min-width: 380px;
  }

  @media (max-width: 600px) {
    .search-wrap.exploration {
      left: 16px;
      right: 16px;
      transform: none;
      min-width: 0;
      top: 60px;
    }
  }

  .search-wrap.visible {
    opacity: 1;
    pointer-events: auto;
  }

  .search-bar {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 12px;
    background: var(--color-glass-light);
    backdrop-filter: var(--blur-light);
    -webkit-backdrop-filter: var(--blur-light);
    box-shadow: var(--shadow-search);
    border-radius: var(--radius-card);
    border: 1px solid rgba(255,255,255,0.4);
    transition: background 0.2s;
  }

  .search-bar.active {
    background: rgba(255,255,255,0.85);
    border-radius: var(--radius-card) var(--radius-card) 0 0;
  }

  .dark .search-bar {
    background: rgba(255,255,255,0.9);
    box-shadow: 0px 1px 32px rgba(0,0,0,0.15);
  }

  .search-icon {
    color: var(--color-text-medium);
    flex-shrink: 0;
  }

  .search-input {
    flex: 1;
    background: none;
    border: none;
    outline: none;
    font-family: var(--font);
    font-weight: 300;
    font-size: 16px;
    color: var(--color-text-medium);
    min-width: 0;
  }

  .search-input::placeholder {
    color: var(--color-text-medium);
    opacity: 0.8;
  }

  .dark .search-input {
    font-size: 20px;
    color: var(--color-text-dark);
  }

  .dark .search-input::placeholder {
    color: var(--color-text-dark);
    opacity: 0.6;
  }

  .clear-btn {
    background: none;
    border: none;
    cursor: pointer;
    color: var(--color-text-medium);
    padding: 2px;
    display: flex;
    align-items: center;
    border-radius: 50%;
    transition: background 0.2s;
  }

  .clear-btn:hover {
    background: rgba(0,0,0,0.06);
  }

  .spinner {
    width: 14px;
    height: 14px;
    border: 2px solid rgba(0,0,0,0.15);
    border-top-color: var(--color-text-medium);
    border-radius: 50%;
    animation: spin 0.6s linear infinite;
    flex-shrink: 0;
  }

  @keyframes spin {
    to { transform: rotate(360deg); }
  }

  .results {
    list-style: none;
    background: rgba(255,255,255,0.95);
    backdrop-filter: var(--blur-light);
    border-radius: 0 0 var(--radius-card) var(--radius-card);
    overflow: hidden;
    box-shadow: 0 4px 16px rgba(0,0,0,0.15);
    max-height: 240px;
    overflow-y: auto;
  }

  .result-item {
    width: 100%;
    display: flex;
    align-items: flex-start;
    gap: 8px;
    padding: 10px 14px;
    background: none;
    border: none;
    cursor: pointer;
    font-family: var(--font);
    font-weight: 300;
    font-size: 14px;
    color: var(--color-text-dark);
    text-align: left;
    transition: background 0.15s;
  }

  .result-item:hover {
    background: rgba(0,0,0,0.04);
  }

  .result-item svg {
    flex-shrink: 0;
    margin-top: 2px;
    color: var(--color-text-medium);
  }

  .result-item span {
    overflow: hidden;
    text-overflow: ellipsis;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
  }
</style>
