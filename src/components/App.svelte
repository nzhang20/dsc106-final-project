<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let tempData = [];

  onMount(async () => {
      const res = await fetch('school_map.csv'); 
      const csv = await res.text();
      tempData = d3.csvParse(csv, d3.autoType)
      console.log(tempData);
    }
  )

</script>

<main>
  <h1>Svelte template</h1>

  <p>Write your HTML here</p>
</main>

<style>
  /* Write your CSS here */
</style>
