<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let tempData = [];
  let names = null;
  let datevalues = null;
  let keyframes = null;
  let nameframes = null;
  let prev = null;
  let next = null;
  let color = null;

  let duration = 250;
  let n = 12;
  let k = 10;
  let barSize = 48;
  let marginTop = 16;
  let marginRight = 6;
  let marginBottom = 6;
  let marginLeft = 0;
  let height = marginTop + barSize * n + marginBottom;
  let width = 1250;

  let x = d3.scaleLinear([0, 1], [marginLeft, width - marginRight]);
  let y = d3.scaleBand()
    .domain(d3.range(n + 1))
    .rangeRound([marginTop, marginTop + barSize * (n + 1 + 0.1)])
    .padding(0.1);

  let formatDate = d3.utcFormat("%Y");
  let formatNumber = d3.format(",d");

  onMount(async () => {
    const res = await fetch('final_enrollment_data.csv'); 
    const csv = await res.text();
    tempData = d3.csvParse(csv, d3.autoType);
    names = new Set(tempData.map(d => d.race));
    datevalues = Array.from(d3.rollup(tempData, ([d]) => d.enrollment, d => +d.year, d => d.race))
      .map(([date, tempData]) => [new Date(date), tempData])
      .sort(([a], [b]) => d3.ascending(a, b));

    keyframes = create_keyframes();
    nameframes = [];

    keyframes.forEach(([date, data]) => {
      data.forEach(item => {
        const index = nameframes.findIndex(([name]) => name === item.name);
        if (index === -1) {
          nameframes.push([item.name, [item]]);
        } else {
          nameframes[index][1].push(item);
        } 
      });
    });

    prev = new Map(nameframes.flatMap(([, tempData]) => d3.pairs(tempData, (a, b) => [b, a])));
    next = new Map(nameframes.flatMap(([, tempData]) => d3.pairs(tempData)));

    color = create_color()

  });

  $: if ((keyframes !== null) && (prev !== null) && (next !== null) && (color !== null)) {
    chart();
  }

  async function chart() {
      const svg = d3.create("svg")
          .attr("viewBox", [0, 0, width, height])
          .attr("width", width)
          .attr("height", height)
          .attr("style", "max-width: 100%; height: auto;");

      const updateBars = bars(svg);
      const updateAxis = axis(svg);
      const updateLabels = labels(svg);
      const updateTicker = ticker(svg);

      document.body.appendChild(svg.node()); // Append SVG to the DOM
    
      for (const keyframe of keyframes) {
          const transition = svg.transition()
              .duration(duration)
              .ease(d3.easeLinear);

          // Extract the top bar’s value.
          x.domain([0, keyframe[1][0].value]);
          
          updateAxis(keyframe, transition);
          updateBars(keyframe, transition);
          updateLabels(keyframe, transition);
          updateTicker(keyframe, transition);

          await transition.end();
      }
  }

  function create_keyframes() {
    const keyframes = [];
    let ka, a, kb, b;
    for ([[ka, a], [kb, b]] of d3.pairs(datevalues)) {
      for (let i = 0; i < k; ++i) {
        const t = i / k;
        keyframes.push([
          new Date(ka * (1 - t) + kb * t),
          rank(name => (a.get(name) || 0) * (1 - t) + (b.get(name) || 0) * t)
        ]);
      }
    }
    keyframes.push([new Date(kb), rank(name => b.get(name) || 0)]);
    return keyframes;
  }

  

  function rank(value) {
    const data = Array.from(names, name => ({name, value: value(name)}));
    data.sort((a, b) => d3.descending(a.value, b.value));
    for (let i = 0; i < data.length; ++i) data[i].rank = Math.min(n, i);
    return data;
  }

  function bars(svg) {
    let bar = svg.append("g")
        .attr("fill-opacity", 0.6)
      .selectAll("rect");

    return ([date, tempData], transition) => bar = bar
      .data(tempData.slice(0, n), d => d.race)
      .join(
        enter => enter.append("rect")
          .attr("fill", color)
          .attr("height", y.bandwidth())
          .attr("x", x(0))
          .attr("y", d => y((prev.get(d) || d).rank))
          .attr("width", d => x((prev.get(d) || d).value) - x(0)),
        update => update,
        exit => exit.transition(transition).remove()
          .attr("y", d => y((next.get(d) || d).rank))
          .attr("width", d => x((next.get(d) || d).value) - x(0))
      )
      .call(bar => bar.transition(transition)
        .attr("y", d => y(d.rank))
        .attr("width", d => x(d.value) - x(0)));
  }

  function labels(svg) {
    let label = svg.append("g")
        .style("font", "bold 12px var(--sans-serif)")
        .style("font-variant-numeric", "tabular-nums")
        .attr("text-anchor", "end")
      .selectAll("text");

    return ([date, tempData], transition) => label = label
      .data(tempData.slice(0, n), d => d.race)
      .join(
        enter => enter.append("text")
          .attr("transform", d => `translate(${x((prev.get(d) || d).value)},${y((prev.get(d) || d).rank)})`)
          .attr("y", y.bandwidth() / 2)
          .attr("x", -6)
          .attr("dy", "-0.25em")
          .text(d => d.name)
          .call(text => text.append("tspan")
            .attr("fill-opacity", 0.7)
            .attr("font-weight", "normal")
            .attr("x", -6)
            .attr("dy", "1.15em")),
        update => update,
        exit => exit.transition(transition).remove()
          .attr("transform", d => `translate(${x((next.get(d) || d).value)},${y((next.get(d) || d).rank)})`)
          .call(g => g.select("tspan").tween("text", d => textTween(d.value, (next.get(d) || d).value)))
      )
      .call(bar => bar.transition(transition)
        .attr("transform", d => `translate(${x(d.value)},${y(d.rank)})`)
        .call(g => g.select("tspan").tween("text", d => textTween((prev.get(d) || d).value, d.value))));
  }

  function textTween(a, b) {
    const i = d3.interpolateNumber(a, b);
    return function(t) {
      this.textContent = formatNumber(i(t));
    };
  }

  function axis(svg) {
    const g = svg.append("g")
        .attr("transform", `translate(0,${marginTop})`);

    const axis = d3.axisTop(x)
        .ticks(width / 160)
        .tickSizeOuter(0)
        .tickSizeInner(-barSize * (n + y.padding()));

    return (_, transition) => {
      g.transition(transition).call(axis);
      g.select(".tick:first-of-type text").remove();
      g.selectAll(".tick:not(:first-of-type) line").attr("stroke", "white");
      g.select(".domain").remove();
    };
  }

  function ticker(svg) {
    const now = svg.append("text")
        .style("font", `bold ${barSize}px var(--sans-serif)`)
        .style("font-variant-numeric", "tabular-nums")
        .attr("text-anchor", "end")
        .attr("x", width - 6)
        .attr("y", marginTop + barSize * (n - 0.45))
        .attr("dy", "0.32em")
        .text(formatDate(keyframes[0][0]));

    return ([date], transition) => {
      transition.end().then(() => now.text(formatDate(date)));
    };
  }

  function create_color() {
    let scale = d3.scaleOrdinal(d3.schemeTableau10);
    return (d, i) => scale(i);
  }
</script>

<main id="chart-container">
  {#if height > 0}
    <svg id="chart-svg" viewBox="0 0 {width} {height}"></svg>
  {/if}
</main>

<style>
  /* Write your CSS here */
</style>
