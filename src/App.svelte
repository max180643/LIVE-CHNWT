<script lang="ts">
  import { onMount } from "svelte";
  import Live from "./components/Live.svelte";
  import Boardcast from "./components/Boardcast.svelte";

  let page = ``;

  onMount(async () => {
    const path = window.location.pathname;
    page = path.substring(1).toLowerCase();
  });
</script>

<div>
  {#if page === ""}
    <Live />
  {:else if page === "boardcast"}
    <Boardcast />
  {:else}
    <h1>404 NOT FOUND</h1>
  {/if}
</div>
