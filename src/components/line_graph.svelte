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
        // const svg = d3.create("svg")
            // .attr("viewBox", [0, 0, width, height])
            // .attr("style", "max-width: 100%; height: auto; font: 8px sans-serif;");

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

        svg.append("g")
            .attr("class", "x-axis")
            .attr("transform", `translate(0, ${height - margin.bottom})`)
            .call(xAxis);

        svg.append("g")
            .attr("class", "y-axis")
            .attr("transform", `translate(${margin.left}, 0)`)
            .call(yAxis);

        const races = [...new Set(data.map(d => d.race))];
        races.forEach(race => {
            const filteredData = data.filter(d => d.race === race);
            svg.append('path')
                .datum(filteredData)
                .attr('class', 'line')
                .attr('d', line)
                .style('stroke', colors(race))
                .style('fill', 'none');
        });

        // const container = document.createElement('div');
        // container.appendChild(svg.node());

        return svg.node();
    }
</script>
