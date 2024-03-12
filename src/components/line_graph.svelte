<script context="module">
    import * as d3 from 'd3';

    export function multiLineGraph(data, lg_height) {
        const width = 600;
        const height = lg_height;
        const margin = ({ top: 30, right: 0, bottom: 30, left: 60 });
        const colors = d3.scaleOrdinal(d3.schemeCategory10);

        const svg = d3.create("svg")
            .attr("viewBox", [0, 0, width + margin.left + margin.right, height + margin.top + margin.bottom])
            .attr("width", width + margin.left + margin.right)
            .attr("height", height + margin.top + margin.bottom);

        const chartGroup = svg.append('g')
            .attr('class', 'chart-group');

        const xScale = d3.scaleTime()
            .domain(d3.extent(data, d => new Date(d.year)))
            .range([margin.left, width - margin.right]);

        const yScale = d3.scaleLinear()
            .domain([0, d3.max(data, d => d.enrollment)])
            .range([height - margin.bottom, margin.top]);

        const line = d3.line()
            .x(d => xScale(new Date(d.year)))
            .y(d => yScale(d.enrollment));

        const xAxis = d3.axisBottom(xScale);
        const yAxis = d3.axisLeft(yScale);

        chartGroup.append("g")
            .attr("class", "x-axis")
            .attr("transform", `translate(0, ${height - margin.bottom})`)
            .call(xAxis);

        chartGroup.append("g")
            .attr("class", "y-axis")
            .attr("transform", `translate(${margin.left}, 0)`)
            .call(yAxis);

        const races = [...new Set(data.map(d => d.race))];
        races.forEach((race, index) => {
            const filteredData = data.filter(d => d.race === race);
            chartGroup.append('path')
                .datum(filteredData)
                .attr('class', 'line')
                .attr('d', line)
                .style('stroke', colors(race))
                .style('fill', 'none');
        });

        // Create legend group
        const legendGroup = svg.append('g')
            .attr('class', 'legend-group')
            .attr('transform', `translate(${width - 100}, ${margin.top})`);

        // Add legend items
        races.forEach((race, index) => {
            // Split "Native Hawaiian or other Pacific Islander" into two rows
            if (race === "Native Hawaiian or other Pacific Islander") {
                const raceText = "Native Hawaiian or other Pacific";
                const islanderText = "Islander";
                
                // Add text for the first row
                legendGroup.append('text')
                    .attr('x', -2)
                    .attr('y', 10 * index)
                    .style('fill', colors(race))
                    .text(raceText)
                    .style('font-size', '11px');
                
                // Add text for the second row
                legendGroup.append('text')
                    .attr('x', 8)
                    .attr('y', 10 * index + 10) // Adjust y position for the second row
                    .style('fill', colors(race))
                    .text(islanderText)
                    .style('font-size', '11px');
            } else if (race === "Two or more races") {
                legendGroup.append('text')
                    .attr('x', -2)
                    .attr('y', 10 * index + 10)
                    .style('fill', colors(race))
                    .text(race)
                    .style('font-size', '11px');
            }else {
                // For other races, add a single text element
                legendGroup.append('text')
                    .attr('x', -2)
                    .attr('y', 10 * index)
                    .style('fill', colors(race))
                    .text(race)
                    .style('font-size', '11px');
            }
        });


        // Add a box around the legend
        const legendBox = legendGroup.node().getBBox();
        legendGroup.insert('rect', ':first-child')
            .attr('x', legendBox.x - 5)
            .attr('y', legendBox.y - 10)
            .attr('width', 160)
            .attr('height', 85)
            .attr('fill', 'white')
            .style('opacity', 0.75)
            .attr('stroke', 'black')
            .attr('stroke-width', 1);

        return svg.node();
    }
</script>
