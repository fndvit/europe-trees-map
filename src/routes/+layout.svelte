<script lang="ts">
  import '../app.css';
  import 'mapbox-gl/dist/mapbox-gl.css';

  let { children } = $props();
</script>

<svelte:head>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <link rel="preconnect" href="https://api.mapbox.com" />
</svelte:head>

{@render children()}
