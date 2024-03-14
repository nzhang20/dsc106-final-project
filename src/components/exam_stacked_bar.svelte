<script context="module">
    import * as d3 from 'd3';

    export function stackedBarChart(ap_data, apHeight) {
        function wrap(text, width) {
            text.each(function () {
                var text = d3.select(this),
                    words = text.text().split(/\s+/).reverse(),
                    word,
                    line = [],
                    lineNumber = 0,
                    lineHeight = 1.1, // ems
                    y = text.attr("y"),
                    dy = parseFloat(text.attr("dy")),
                    tspan = text.text(null).append("tspan").attr("x", 0).attr("y", y).attr("dy", dy + "em");
                while (word = words.pop()) {
                    line.push(word);
                    tspan.text(line.join(" "));
                    if (tspan.node().getComputedTextLength() > width) {
                        line.pop();
                        tspan.text(line.join(" "));
                        line = [word];
                        tspan = text.append("tspan").attr("x", 0).attr("y", y).attr("dy", ++lineNumber * lineHeight + dy + "em").text(word);
                    }
                }
            });
        }

        const dataset = ap_data;

        const margin = { top: 100, right: 20, bottom: 80, left: 60 };
        const width = 800 - margin.left - margin.right; 
        const height = apHeight - 20; // 700 - margin.top - margin.bottom;

        const x = d3.scaleBand()
            .range([0, width])
            .padding(0.1);

        const y = d3.scaleLinear()
            .range([height, 0]);

        const colorRange = d3.scaleOrdinal(d3.schemeCategory10);
        const color = d3.scaleOrdinal()
            .domain(Object.keys(dataset[0]).filter(function (key, index) { return index > 0; }))
            .range(["#1f77b4", "#ff7f0e", "#2ca02c"]);

        const xAxis = d3.axisBottom(x);

        const yAxis = d3.axisLeft(y)
            .tickFormat(d3.format(".2s"));

        const svgContainer = d3.select("#stackedbar-chart-container");

        const svg = svgContainer.append("svg")
            .attr("width", width + margin.left + margin.right)
            .attr("height", height + margin.top + margin.bottom)
            .append("g")
            .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

        // Add y-axis title
        svg.append("text")
            .attr("transform", "rotate(-90)")
            .attr("y", 0 - margin.left)
            .attr("x", 0 - height / 2)
            .attr("dy", "1em")
            .style("text-anchor", "middle")
            .text("Number of Students");

        // Add x-axis title
        svg.append("text")
            .attr("y", height + margin.bottom / 1.5 - 30)
            .attr("x", width / 2)
            .attr("dy", "1em")
            .style("text-anchor", "middle")
            .text("Race");


        color.domain(Object.keys(dataset[0]).filter(function (key) { return key !== "Race"; }));

        dataset.forEach(function (d) {
            let y0 = 0;
            d.total = d3.sum(Object.keys(d).slice(1).map(key => d[key])); // Calculate total value for each bar
            d.values = color.domain().map(function (name) {
                const percentage = (d[name] / d.total) * 100; // Calculate percentage for each segment
                return {
                    name: name,
                    percentage: percentage, 
                    y0: y0,
                    y1: y0 += +d[name]
                };
            });
        });

        dataset.sort((a, b) => b.total - a.total);

        x.domain(dataset.map(function (d) { return d.Race; }));
        y.domain([0, d3.max(dataset, function (d) { return d.total; })]);

        svg.append("g")
            .attr("class", "x axis")
            .attr("transform", "translate(0," + height + ")")
            .call(xAxis);

        svg.append("g")
            .attr("class", "y axis")
            .call(yAxis);

        const bar = svg.selectAll(".Race")
            .data(dataset)
            .enter().append("g")
            .attr("class", "g")
            .attr("transform", function (d) { return "translate(" + x(d.Race) + ",0)"; });

        svg.selectAll(".x.axis .tick text")
            .call(wrap, x.bandwidth());

        const bar_enter = bar.selectAll("rect")
            .data(function (d) { return d.values; })
            .enter();

        // bar_enter.append("rect")
        //     .attr("width", x.bandwidth())
        //     .attr("y", function (d) { return y(d.y1); })
        //     .attr("height", function (d) { return y(d.y0) - y(d.y1); })
        //     .style("fill", function (d) { return color(d.name); });

                
        // bar_enter.append("text")
        //     .text(function (d) { return d3.format(".2f")(d.percentage) + "%"; }) // Display the percentage
        //     .attr("y", function (d) { return y(d.y1) + (y(d.y0) - y(d.y1)) / 2; })
        //     .attr("x", x.bandwidth() / 3)
        //     .style("fill", '#ffffff');

        // tooltip
        const tooltip = d3.select("#stackedbar-chart-container").append("div")
            .attr("class", "tooltip")
            .style("opacity", 0);

        bar_enter.append("rect")
            .attr("width", x.bandwidth())
            .attr("y", function (d) { return y(d.y1); })
            .attr("height", function (d) { return y(d.y0) - y(d.y1); })
            .style("fill", function (d) { return color(d.name); })
            // Show tooltip on mouseover
            .on("mouseover", function (event, d) {
                tooltip.transition()
                    .duration(200)
                    .style("opacity", .9);
                const percentage = d3.format(".2f")(d.percentage) + "%";
                const race = d3.select(this.parentNode).datum().Race;
                tooltip.html("Race: " + race + "<br/>" + "Segment: " + d.name + "<br/>" + "Value: " + (d.y1 - d.y0) + "<br/>" + "Percentage: " + percentage)
                    .style("left", (event.pageX) + "px")
                    .style("top", (event.pageY + 10) + "px");
            })
            // Hide tooltip on mouseout
            .on("mouseout", function (d) {
                tooltip.transition()
                    .duration(500)
                    .style("opacity", 0);
            });




        // legend
        const legend = svg.selectAll(".legend")
            .data(color.domain().slice().reverse())
            .enter().append("g")
            .attr("class", "legend")
            .attr("transform", function (d, i) { return "translate(0," + i * 20 + ")"; });

        legend.append("rect")
            .attr("x", width - 18)
            .attr("width", 18)
            .attr("height", 18)
            .style("fill", color);

        legend.append("text")
            .attr("x", width - 24)
            .attr("y", 9)
            .attr("dy", ".35em")
            .style("text-anchor", "end")
            .text(function (d) { return d; });

        // svg.select(".selectAll")
        // .call();

        return svg.node();
    }

</script>