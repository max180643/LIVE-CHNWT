<script lang="ts">
  import { onMount } from "svelte";
  import Live from "./components/Live.svelte";
  import Boardcast from "./components/Boardcast.svelte";

  let page = ``;

  onMount(async () => {
    let url = new URL(window.location.href);
    page = url.searchParams.get("p") ?? "";
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
