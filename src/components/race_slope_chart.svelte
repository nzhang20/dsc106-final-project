<script context="module">
  import * as d3 from 'd3';
  import { slopeChart } from '../components/slope_chart.svelte';

  // Function for slope chart
  export function raceSlopeChart(originalData, harassData, disciplineData, dataType, sectionHeight) {
    let data = [];
    let originalChart = slopeChart(originalData, harassData, disciplineData);
    let chartTitle = '';

    if (dataType === 'harass') {
        data = harassData;
        chartTitle = 'Number of Students Harassed by Race';
    } else {
        data = disciplineData;
        chartTitle = 'Number of Students Disciplined by Race';
    }

    // Chart dimensions
    const width = 1000;
    let height = 450;
    if (sectionHeight < height) {
      height = sectionHeight - 50;
    }
    height -= 85;
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
        .attr("viewBox", [0, 0, width, height + 10])
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
      .data(d3.group(data, d => d.race))
      .join("path")
        .attr("d", ([, values]) => line(values))
        .attr("class", "line") // Add class for easier selection
        .attr("stroke", d => getColor(d[0])) // Set the stroke color based on race
        .attr("stroke-width", 2) // Adjust stroke width as needed
        .style("pointer-events", "all") // Allow lines to receive mouse events
        .on("mouseenter", handleMouseEnter)
        .on("mouseleave", handleMouseLeave);

    function handleMouseEnter() {
        d3.select(this)
          .attr("stroke", "#af69ee"); // Change color to purple on hover
    }

    function handleMouseLeave() {
        d3.select(this)
            .attr("stroke", function(d) {
                return getColor(d[0]); // Retain the original color on mouse leave
            });
    }

    // Function to get color based on race
    function getColor(race) {
        // Define a color scale based on race
        const colorScale = d3.scaleOrdinal()
            .domain([...new Set(data.map(d => d.race))])
            .range(d3.schemeCategory10);

        return colorScale(race);
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
        .data(([step, values]) => {
            const raceCounts = {};
            values.forEach(d => raceCounts[d.race] = d.count);
            return Object.entries(raceCounts).map(([race, count]) => ({ race, count }));
        })
        .join("text")
            .attr("y", d => y(d.count))
            .attr("dy", "0.35em")
            .text(d => `${formatNumber(d.count)}`)
            .attr("fill", "currentColor")
            .attr("stroke", "white")
            .attr("stroke-width", 5)
            .attr("paint-order", "stroke")
            .attr("dx", 5); // Adjust the horizontal position of the text

    // Add race labels on the sides
    let startYPositions = {};
    svg.append("g")
        .selectAll("g")
        .data(d3.group(data, d => d.race))
        .join("g")
            .append("text")
            .attr("x", d => x(d[0][d[1].length - 1].year)) 
            .attr("y", d => {
                const yPosition = Math.round(y(d[1][0].count) / 10) * 10;
                if (startYPositions[yPosition] !== undefined) {
                    const lastY = startYPositions[yPosition];
                    const lastHeight = parseFloat(svg.select(`text[y="${lastY}"]`).node().getBBox().height);
                    let newY = lastY + lastHeight + 10;
                    while (startYPositions[newY] !== undefined) {
                        newY = newY + 10
                    }
                    startYPositions[newY] = newY;
                    return newY - 3;
                } else {
                    startYPositions[yPosition] = yPosition;
                    return yPosition;
                }
            })
            .text(d => d[0])
            .attr("dx", 150)
            .attr("text-anchor", "end") 
            .attr("fill", "currentColor")
            .style("font-size", "7px");

    // Add race labels on the sides
    let lastYPositions = {};
    svg.append("g")
        .selectAll("g")
        .data(d3.group(data, d => d.race))
        .join("g")
            .append("text")
            .attr("x", d => x(d[1][d[1].length - 1].year)) 
            .attr("y", d => {
                const yPosition = Math.round(y(d[1][d[1].length - 1].count) / 10) * 10;
                if (lastYPositions[yPosition] !== undefined) {
                    const lastY = lastYPositions[yPosition];
                    const lastHeight = parseFloat(svg.select(`text[y="${lastY}"]`).node().getBBox().height);
                    let newY = lastY + lastHeight + 10;
                    while (lastYPositions[newY] !== undefined) {
                        newY = newY + 10
                    }
                    lastYPositions[newY] = newY;
                    return newY - 3;
                } else {
                    lastYPositions[yPosition] = yPosition;
                    return yPosition;
                }
            })
            .text(d => d[0])
            .attr("dx", 15)
            .attr("text-anchor", "start") 
            .attr("fill", "currentColor")
            .style("font-size", "7px");

    // Add title
    svg.append("text")
       .attr("x", (width + marginLeft - marginRight) / 2)
       .attr("y", marginTop / 2)
       .attr("text-anchor", "middle")
       .text(chartTitle)
       .attr("fill", "currentColor")
       .style("font-size", "10px")
       .attr("dy", "-1.25em");

    // Return back to original graph
    svg.on("click", () => {
        const container = document.getElementById('slope-chart-container');
        if (container) {
            container.innerHTML = ''; // Clear existing content
            container.append(originalChart);
        }
    });

    // Append text beneath the chart
    svg.append("text")
        .attr("x", width / 2)
        .attr("y", height + 7) // Adjust the y-coordinate as needed
        .attr("text-anchor", "middle")
        .text("Click anywhere to return") 
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
