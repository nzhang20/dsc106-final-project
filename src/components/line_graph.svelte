<script context="module">
    import * as d3 from 'd3';

    export function multiLineGraph(data, lg_height) {
        const width = 800;
        let height = 600;
        if (lg_height < height) {
            height = lg_height;
        }
        const margin = ({ top: 30, right: 0, bottom: 30, left: 90 });
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

        // add title
        svg.append("text")
            .attr("x", width / 2 + 20)
            .attr("y", margin.top / 2 - 2)
            .attr("text-anchor", "middle")
            .style("font-size", "1.2em")
            .text("Student Enrollment Trends by Race");
        
        // Adjust font size of y-axis label
        svg.append("text")
            .attr("transform", "rotate(-90)")
            .attr("y", margin.left / 2 - 40) // Adjust position of the label
            .attr("x", 0 - (height / 2))
            .attr("dy", "1em")
            .style("text-anchor", "middle")
            .style("font-size", "1em") // Adjust font size here
            .text("Number of Enrolled Students");

        // Adjust font size of x-axis label
        svg.append("text")
            .attr("transform", `translate(${width / 2 + 35}, ${height + margin.top - 20})`) // Adjust position of the label
            .style("font-size", "1em") // Adjust font size here
            .style("text-anchor", "middle")
            .text("Year");


        const races = [...new Set(data.map(d => d.race))];
        races.forEach((race, index) => {
            const filteredData = data.filter(d => d.race === race);
                chartGroup.append('path')
                    .datum(filteredData)
                    .attr('class', 'line')
                    .attr('d', line)
                    .style('stroke', colors(race))
                    .style('stroke-width', 3) // make line thicker
                    .style('fill', 'none');

                const area = d3.area()
                    .x(d => xScale(new Date(d.year)))
                    .y0(height - margin.bottom)
                    .y1(d => yScale(d.enrollment));

                chartGroup.append('path')
                    .datum(filteredData)
                    .attr('class', 'area')
                    .attr('d', area)
                    .style('fill', colors(race))
                    .style('opacity', 0.2); // Adjust opacity as needed


        });

        // Calculate the height of the legend group based on the number of legend items and font size
        const legendItemHeight = 20; // Adjust as needed
        const legendGroupHeight = races.length * legendItemHeight;

        // Create legend group
        const legendGroup = svg.append('g')
            .attr('class', 'legend-group')
            .attr('transform', `translate(${width - 200}, ${margin.top})`);

        // Add legend items
        races.forEach((race, index) => {
            // Add text for the legend items
            legendGroup.append('text')
                .attr('x', 0)
                .attr('y', index * legendItemHeight)
                .style('fill', colors(race))
                .text(race)
                .style('font-size', '14px'); // Adjust font size as needed
        });

        // Calculate the width and height of the legend box
        const legendBoxWidth = 180; // Adjust as needed
        const legendBoxHeight = legendGroupHeight + 10; // Add padding

        // Add a rectangle around the legend items
        legendGroup.insert('rect', ':first-child')
            .attr('x', -5)
            .attr('y', -20)
            .attr('width', legendBoxWidth+65)
            .attr('height', legendBoxHeight)
            .attr('fill', 'white')
            .style('opacity', 0.75)
            .attr('stroke', 'black')
            .attr('stroke-width', 1);


        return svg.node();
    }
</script>
