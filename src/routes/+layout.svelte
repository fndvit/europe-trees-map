<script lang="ts">
  import '../app.css';
  import 'maplibre-gl/dist/maplibre-gl.css';

  let { children } = $props();
</script>

<svelte:head>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <link rel="preconnect" href="https://tiles.openfreemap.org" />
</svelte:head>

{@render children()}
