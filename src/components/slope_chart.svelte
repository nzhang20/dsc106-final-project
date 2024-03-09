<script context="module">
  import * as d3 from 'd3';
  import { raceSlopeChart } from './race_slope_chart.svelte';

  // Function for slope chart
  export function slopeChart(data, harassData, disciplineData, sectionHeight) {
    // Chart dimensions
    const width = 1000;
    let height = 450;
    if (sectionHeight < 450){
      height = sectionHeight;
    }
    const marginTop = 40;
    const marginRight = 50;
    const marginBottom = 10;
    const marginLeft = 50;
    const padding = 3;

    const steps = [...new Set(data.map(d => d.year))];

    const x = d3.scalePoint()
      .domain(steps)
      .range([marginLeft, width - marginRight])
      .padding(0.5);

    const y = d3.scaleLinear()
      .domain(d3.extent(data.map(d => d.count)))
      .range([height - marginBottom, marginTop]);

    const line = d3.line()
      .x(d => x(d.year))
      .y(d => y(d.count));

    const formatNumber = y.tickFormat(100);

    // Create SVG container
    const svg = d3.create("svg")
        .attr("viewBox", [0, 0, width, height + 20])
        .attr("style", "max-width: 100%; height: auto; font: 8px sans-serif;");

    // Create x axis
    svg.append("g")
        .attr("text-anchor", "middle")
      .selectAll("g")
      .data(steps)
      .join("g")
        .attr("transform", (d) => `translate(${x(d)},20)`)
        .call(g => g.append("text").text((d) => d))
        .call(g => g.append("line").attr("y1", 3).attr("y2", 9).attr("stroke", "currentColor"));

    // Create a line for each name
    const lines = svg.append("g")
      .attr("fill", "none")
      .selectAll("path")
      .data(d3.group(data, d => d.type))
      .join("path")
        .attr("d", ([, values]) => line(values))
        .attr("class", "line") // Add class for easier selection
        .attr("stroke", "currentColor") // Set the initial stroke color
        .attr("stroke-width", 2) // Adjust stroke width as needed
        .style("pointer-events", "all") // Allow lines to receive mouse events
        .on("mouseenter", handleMouseEnter)
        .on("mouseleave", handleMouseLeave)
        .on("click", handleMouseClick);

    function handleMouseEnter() {
        d3.select(this)
          .attr("stroke", "#af69ee"); // Change color to purple on hover
    }

    function handleMouseClick(event) {
        const type = event.target.__data__[0]; // Assuming the type is stored as the first element of the data array associated with each line
        const container = document.getElementById('slope-chart-container');
        if (container) {
            container.innerHTML = ''; // Clear existing content
            
            if (type === "# Harassed") {
              const sc_harass = raceSlopeChart(data, harassData, disciplineData, 'harass', sectionHeight); // Generate race slope chart with harassData
              container.appendChild(sc_harass); // Append the race slope chart to the container
            } else if (type === "# Disciplined") {
              const sc_discipline = raceSlopeChart(data, harassData, disciplineData, 'discipline', sectionHeight); // Generate race slope chart with disciplineData
              container.appendChild(sc_discipline); // Append the race slope chart to the container
            }
        }
    }

    function handleMouseLeave() {
        d3.select(this)
            .attr("stroke", function() {
                return d3.select(this).classed("hovered") ? "purple" : "currentColor";
            }); // Revert color on mouse leave if not hovered
    }


    // Create a group of labels for each year
    svg.append("g")
      .selectAll("g")
      .data(d3.group(data, d => d.year))
      .join("g")
        .attr("transform", ([step]) => `translate(${x(step) + (
            step === "2017" ? padding
          : step === "2011" ? -padding
          : 0
        )},0)`)
        .attr("text-anchor", ([step]) =>
            step === "2011" ? "end"
            : step === "2017" ? "start"
            : "middle")
      .selectAll("text")
      .data(([step, values]) => d3.zip(
        values.map(
            step === "2017" ? (d) => `${formatNumber(d.count)} ${d.type}`
          : step === "2011" ? (d) => `${d.type} ${formatNumber(d.count)}`
          : (d) => `${formatNumber(d.count)}`),
        dodge(values.map(d => y(d.count)))))
      .join("text")
        .attr("y", ([, y]) => y)
        .attr("dy", "0.35em")
        .text(([text]) => text)
        .attr("fill", "currentColor")
        .attr("stroke", "white")
        .attr("stroke-width", 5)
        .attr("paint-order", "stroke");

    svg.append("g")
      .selectAll("g")
      .data(d3.group(data, d => d.type))
      .join("g")
      .append("text")
        .attr("x", d => x(d[0][d[1].length - 1].year)) // Place the text at the x-coordinate of the last data point for each type
        .attr("y", d => y(d[1][0].count)) // Place the text at the y-coordinate of the first data point for each type
        .text(d => d[0]) // Use the type as the text content
        .attr("dy", "0.35em") // Adjust the vertical position of the text
        .attr("dx", 110) // Adjust the horizontal position of the text
        .attr("text-anchor", "end") 
        .attr("fill", "currentColor")
        .style("font-size", "8px"); // You can adjust the font size as needed
    
    svg.append("g")
      .selectAll("g")
      .data(d3.group(data, d => d.type))
      .join("g")
      .append("text")
        .attr("x", d => x(d[1][d[1].length - 1].year)) // Place the text at the x-coordinate of the last data point for each type
        .attr("y", d => y(d[1][d[1].length - 1].count)) // Place the text at the y-coordinate of the first data point for each type
        .text(d => d[0]) // Use the type as the text content
        .attr("dy", "0.4em") // Adjust the vertical position of the text
        .attr("dx", 15) // Adjust the horizontal position of the text
        .attr("fill", "currentColor")
        .style("font-size", "8px"); // You can adjust the font size as needed

    // Append text beneath the chart
    svg.append("text")
        .attr("x", width / 2)
        .attr("y", height + 7) // Adjust the y-coordinate as needed
        .attr("text-anchor", "middle")
        .text("Click line to expand") 
        .attr("fill", "gray")
        .style("font-size", "10px"); 

    return svg.node();
  }

  function dodge(positions, separation = 10, maxiter = 10, maxerror = 1e-1) {
    positions = Array.from(positions);
    let n = positions.length;
    if (!positions.every(isFinite)) throw new Error("invalid position");
    if (!(n > 1)) return positions;
    let index = d3.range(positions.length);
    for (let iter = 0; iter < maxiter; ++iter) {
      index.sort((i, j) => d3.ascending(positions[i], positions[j]));
      let error = 0;
      for (let i = 1; i < n; ++i) {
        let delta = positions[index[i]] - positions[index[i - 1]];
        if (delta < separation) {
          delta = (separation - delta) / 2;
          error = Math.max(error, delta);
          positions[index[i - 1]] -= delta;
          positions[index[i]] += delta;
        }
      }
      if (error < maxerror) break;
    }
    return positions;
  }

</script>
