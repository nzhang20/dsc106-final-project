<script>
  import { onMount } from 'svelte';
  import { slopeChart } from './slope_chart.svelte';
  import { stackedBarChart } from './exam_stacked_bar.svelte';
  import { multiLineGraph } from './line_graph.svelte';
  import Scroller from "@sveltejs/svelte-scroller"
  import * as d3 from 'd3';

  let count, offset, progress;
  let index = 0;

  let scrollerWidth, scrollerHeight;
  let num_sections = 5; // # of sections in scrollyteller

  let tempData = [];
  let harass_bully_data = [];
  let harass_data = [];
  let discipline_data = [];
  let ap_data = [];

  let names = null;
  let datevalues = null;
  let keyframes = null;
  let nameframes = null;
  let prev = null;
  let next = null;
  let color = null;

  let duration = 150;
  let n = 7; // number of categories
  let k = 10;
  let barSize = 48;
  let marginTop = 80;
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
    // console.log(tempData);

    // Data for slope_chart
    const res2 = await fetch('final_harass_bully.csv'); 
    const csv2 = await res2.text();
    harass_bully_data = d3.csvParse(csv2, d3.autoType);
    // console.log(harass_bully_data);

    const res3 = await fetch('final_harass_df.csv'); 
    const csv3 = await res3.text();
    harass_data = d3.csvParse(csv3, d3.autoType);
    // console.log(harass_data);

    const res4 = await fetch('final_discipline_df.csv'); 
    const csv4 = await res4.text();
    discipline_data = d3.csvParse(csv4, d3.autoType);

    // Data for stacked bar chart
    const res5 = await fetch('final_ap_df.csv');
    const csv5 = await res5.text();
    ap_data = d3.csvParse(csv5, d3.autoType);
    // console.log(ap_data);

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
    
    window.addEventListener('scroll', updateIndex);
    return () => window.removeEventListener('scroll', updateIndex);

  });

  $: if ((keyframes !== null) && (prev !== null) && (next !== null) && (color !== null)) {
    chart();
    // console.log(tempData);
    
    let lg_height = (document.body.clientHeight / count) / 2;
    console.log(lg_height);
    const line_graph = multiLineGraph(tempData, lg_height); // Call your multiLineGraph function
    // console.log(line_graph);

    const container = document.getElementById('multiline-graph');
    if (container) {
      container.innerHTML = ''; // Clear existing content
      container.style.textAlign = "center";
      container.appendChild(line_graph); // Append the line graph to the container
    }
  }

  $: if ((harass_bully_data.length > 0) && (harass_data.length > 0) && (discipline_data.length > 0)) {
    let sc_height = document.body.clientHeight / count;
    const sc_harass_bully = slopeChart(harass_bully_data, harass_data, discipline_data, sc_height);

    const container = document.getElementById('slope-chart-container');
    if (container) {
      container.innerHTML = '';
      
      container.appendChild(sc_harass_bully);
    }
  }

  $: if (ap_data.length > 0) {
    let ap_height = (document.body.clientHeight / count) / 2;
    const ap_stacked_bar = stackedBarChart(ap_data, ap_height);
    // const container = document.getElementById('stackedbar-chart-container');
    // if (container) {
    //   container.innerHTML = '';

    //   container.appendChild(ap_stacked_bar);
    // }

    const container = document.getElementById("link-container");
    const link = document.createElement("a");
    link.href = "https://www.youtube.com/watch?v=sllqLQLRlJo"
    link.textContent = 'Demo Video';
  
    container.appendChild(link);
  }

  



  function updateIndex() {
    const scrollPosition = window.scrollY;
    const windowHeight = window.innerHeight;
    const totalHeight = document.body.clientHeight;
    const sectionHeight = totalHeight / count;

    // Calculate the index based on scroll position and section height
    let pos = Math.round(((scrollPosition + windowHeight / 2) / sectionHeight) * 100) / 100; // get 2 decimal places
    let new_index = Math.floor(pos);
    // console.log(pos);

    if (new_index !== index) {
      index = new_index;
    } else if (pos === 3.50){
      // only show bar chart race once it gets to index 2
      replay();
    }
  }

  async function chart() {
      const svg = d3.create("svg")
          .attr("viewBox", [0, 0, width, height])
          .attr("width", width)
          .attr("height", height - 65)
          .attr("style", "max-width: 100%; height: auto;");
       
      // add title
      svg.append("text")
        .attr("x", width / 2)
        .attr("y", marginTop / 2)
        .attr("text-anchor", "middle")
        .style("font-size", "1.5em")
        .text("Student Enrollment from 2000-2021");
      
      // add subtitle
      svg.append("text")
        .attr("x", width / 2)
        .attr("y", marginTop / 2 + 17)
        .attr("text-anchor", "middle")
        .style("font-size", "1em")
        .text("K-12th Grades + Undergrad");
      

      const updateBars = bars(svg);
      const updateAxis = axis(svg);
      const updateLabels = labels(svg);
      const updateTicker = ticker(svg);

      // document.body.appendChild(svg.node()); // Append SVG to the DOM
      const replayButton = document.createElement('button');
      replayButton.textContent = 'Replay';
      replayButton.addEventListener('click', replay);


      const container = document.getElementById('chart-container');
      if (container) {
        container.innerHTML = ''; // Clear existing content
        
        // Append the replay button to the container
        const replayButton = document.createElement('button');
        replayButton.textContent = 'Replay';
        replayButton.className = 'replay-button';
        replayButton.addEventListener('click', replay);
        container.appendChild(replayButton);
        
        replayButton.style.position = 'absolute';
        replayButton.style.left = '47.5%';
        replayButton.style.top = (100 * 3 + 72) + 'vh'; // * x = section

        container.appendChild(svg.node());
      }


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
        .attr("text-anchor", "start")
      .selectAll("text");

    return ([date, tempData], transition) => label = label
      .data(tempData.slice(0, n), d => d.race)
      .join(
        enter => enter.append("text")
          .attr("transform", d => `translate(${x((prev.get(d) || d).value)},${y((prev.get(d) || d).rank)})`)
          .attr("y", y.bandwidth() / 2)
          .attr("x", 5)
          .attr("dy", "-0.25em")
          .text(d => d.name)
          .call(text => text.append("tspan")
            .attr("fill-opacity", 0.7)
            .attr("font-weight", "normal")
            .attr("x", 5)
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

  // for chart replay
  async function replay() {
    const svg = d3.select("#chart-svg");

    // Clear existing content of the SVG
    svg.selectAll("*").remove();

    // Call the chart function to redraw the chart
    await chart();
  }

</script>


<Scroller
    top={0.0}
    bottom={1}
    threshold={0.5}
    bind:count
    bind:index
    bind:offset
    bind:progress
  >
    <div 
      class="background" 
      slot="background"
      bind:clientWidth={scrollerWidth}
      bind:clientHeight={scrollerHeight}
      >
    </div>

    <div class="foreground" slot="foreground">
      <section> <!-- hook -->
        <p id="hook">Imagine a world where every student, regardless of race, has equal opportunities and access to education. But the reality is far from this ideal...</p>
      </section>
      <section>
        <h1>
          <!-- Understanding Education: How Has Schooling Changed Over the Years? -->
          Critical Race Theory (CRT) and Education
        </h1>
        <h2>
          Diversity in Education: How Has Schooling Changed Over the Years?
        </h2>
        <p id="authors"> By: Charisse Hao and Nicole Zhang</p>
        <main id="link-container"></main>
      </section>
      <section>
        <p id="quote_setup"> 
          In the late 1970s to early 1980s, the Critical Race Theory (CRT) was developed, encompassing:
        </p>
        <p id="quote">
“a set of ideas holding that racial bias is inherent in many parts of western society, especially in its legal and social institutions, on the basis of their having been primarily designed for...white people” - Oxford Languages
        </p>
        <p>
How has this affected education institutions since then?

          <!-- Critical Race Theory (CRT): the theory that racial bias is inherent in many parts of society, 
          including educational institutions  -->
        </p>
        <!-- on the basis of their having been primarily designed for and implemented by white people. -->
      </section>
      <section>
        <main id="chart-container">
          <button class="replay-button" on:click={replay}>Replay</button>
        </main>
      </section>
      <section>
        <main id="multiline-graph"></main>
        <p id="multiline-graph-description">
          While white enrollment has historically been predominant, the gap between white and minority enrollment has gradually narrowed over time.
        </p>
      </section>
      <section>
<<<<<<< HEAD
        <div class="column-container">
          <div class="column">
            <!-- Content for the first column -->
            <h3>Enrollment in 2000:</h3>
            <p>
              58% White<br>
              42% Minorities<br>
              &nbsp;&nbsp;&nbsp;19% Hispanic<br>
              &nbsp;&nbsp;&nbsp;18% Black<br>
              &nbsp;&nbsp;&nbsp;4% Asian<br>
              &nbsp;&nbsp;&nbsp;1% American Indian or Alaska Native<br>
            </p>
          </div>
          <div class="column">
            <!-- Content for the second column -->
            <p id="arrow"> ➡ </p>
          </div>
          <div class="column">
            <!-- Content for the third column -->
            <h3>In 2017:</h3>
            <p>
              47% White<br>
              53% Minorities<br>
              &nbsp;&nbsp;&nbsp;27% Hispanic<br>
              &nbsp;&nbsp;&nbsp;15% Black<br>
              &nbsp;&nbsp;&nbsp;5% Asian<br>
              &nbsp;&nbsp;&nbsp;4% Two or more races<br>
              &nbsp;&nbsp;&nbsp;1% American Indian or Alaska Native<br>
              &nbsp;&nbsp;&nbsp;0.4% Native Hawaiian or other Pacific Islander<br>
            </p>
          </div>
        </div>
        <!-- <p>  58% Whites vs 41% Minorities</p>
=======
        <h2>Enrollment:</h2>
        <p> In 2000: 58% Whites vs 41% Minorities</p>
>>>>>>> b07b8ef7705864ca784fcb5b384337154f1c4037
        <p> ↓ </p>
        <p> In 2021: 47% White vs 52% Minorities</p> -->
      </section>
      <section>
        <main id="slope-chart-container">
        </main>
        <p>
          This reveals a significant decrease in harassment and bullying over the years. However, among racial demographics, black students stand out with notably high rates of both harassment and disciplinary actions. These findings suggest a potential bias within school systems, indicating a disproportionate tendency to discipline black students and potentially implying an increased likelihood of being targeted for harassment compared to other racial groups.
        </p>
      </section>
      <section>
        <main id="stackedbar-chart-container"></main>
        <p>
          This data illustrate a concerning trend where minorities, specifically black students, tend to have lower participation rates in AP exams, which are crucial for advancing in higher education. This discrepancy potentially places them at a significant disadvantage in accessing advanced educational opportunities.
        </p>
      </section>
      <!-- temp section for video -->
      <section>
        <h1>
          Takeaways 
        </h1>
        <p>
          These visualizations reveal progress in diversity and decreased harassment in educational institutions. However, black students are still more likely to be harassed and have lower AP exam enrollment rates. Acknowledging that societal change takes time, we must continue striving for equality for racial justice.
        </p>
      </section>
      
      
    </div>
</Scroller>

<style>
  /* Title style */
  h1 {
      font-size: 2.5rem;
      margin-left: 200px;
      margin-right: 200px;
      color: #000; 
      font-family: "Times New Roman"; 
      /* text-align: left; Left align the title */
  }
  h2 {
      font-size: 1.25rem;
      margin-top: -20px;
      margin-left: 200px;
      margin-right: 200px;
      color: #000; 
      font-family: "Times New Roman"; 
      /* text-align: left; Left align the title */
  }
  p {
      font-size: 1.25em;
      margin-top: 50px;
      margin-bottom: 50px;
      margin-left: 250px;
      margin-right: 250px;
      color: #000; 
      font-family: "Times New Roman"; 
      text-align: justify; /* Left align the title */
  }

  #hook {
      font-size: 1.75em;
      margin-left: 300px;
      margin-right: 300px;
      color: #000; 
      font-family: "Times New Roman"; 
      text-align: center;
  }

  #authors {
      font-size: 1.1em;
      margin-top: 0px;
      margin-bottom: 7.5px;
      margin-left: 250px;
      margin-right: 250px;
      color: #000; 
      font-family: "Times New Roman"; 
      text-align: center;
  }

  #quote_setup {
      font-size: 1.25em;
      margin-top: 50px;
      margin-left: 250px;
      margin-right: 250px;
      color: #000; 
      font-family: "Times New Roman"; 
      text-align: justify; /* Left align the title */
  }

  #quote {
    font-style: italic; /* Make the text italic */
    color: #555; /* Change the text color */
    font-size: 1.2em; /* Adjust the font size */
    margin-left: 200px; /* Adjust the margin */
    margin-right: 200px;
    padding: 10px; /* Add padding */
    background-color: #f9f9f9; /* Add a background color */
    border-left: 5px solid #ccc; /* Add a left border */
  }

  #multiline-graph-description {
    font-size: 1.25em;
      margin-left: 250px;
      margin-right: 250px;
      margin-top: -10px;
      color: #000; 
      font-family: "Times New Roman"; 
      text-align: center;
  }

  .background {
    width: 100%;
    height: 100vh;
    position: relative;
  }

  .foreground {
    width: 100%;
    margin: 0 auto;
    height: auto;
    position: relative;
    /* outline: red solid 3px; */
  }

  section {
    display: flex;
    flex-direction: column;
    justify-content: center; /* center horizontally */
    align-items: center; /* center vertically */

    width: 95%;
    height: 90vh;
    /* background-color: rgba(0, 0, 0, 0.2); 20% opaque */
    /* color: white; */
    /* outline: magenta solid 3px; */
    text-align: center;
    /* max-width: 1000px; adjust at will */
    color: black;
    padding: 1em;
    margin: 0 auto 2em auto;
    /* overflow-x: hidden; Hide horizontal scrollbar */
  }

  #slope-chart-container {
    width: 100%; /* Adjust width as needed */
    margin-top: 5vh;
    height: 90vh; /* Adjust height as needed */
  }

  #stackedbar-chart-container {
    width: 100%;
    height: 90vh; /* or any other height */
  }

  .replay-button {
    position: absolute;
    left: 50%;
    transform: translateX(-50%);
    bottom: 20px;
    z-index: 1;
  }
  
  .column-container {
    margin-top: -150px;
    display: flex;
    width: 80%;
    /* margin: 0 auto; Center the columns horizontally */
    justify-content: space-between; /* Distribute space between columns */
    /* outline: blue solid 3px; */
  }

  .column {
    flex: 1 1 0; /* Each column takes one-third of the page's width */
    max-width: 40%;
    margin: 0 ; /* Adjust margin as needed */
    /* text-align: right; Center align content within columns */
    /* outline: red solid 3px; */
  }

  .column h3 {
    font-size: 2em;
    color: #333; /* Adjust text color */
    text-align: justify;
  }

  .column p {
    font-size: 1.5em;
    color: #666; /* Adjust text color */
    text-align: left;
    margin: 15px;
  }

  #arrow {
    display: flex;
    flex-direction: column; /* Ensure content stacks vertically */
    justify-content: center; /* Center vertically */
    align-items: center; /* Center horizontally */
    font-size: 2em; /* Adjust font size as needed */
    padding: 1em;
    margin: 0 auto 2em auto;
    height: 100%; /* Set height to 100% */
  }
</style>