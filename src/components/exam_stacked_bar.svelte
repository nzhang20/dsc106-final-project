<script context="module">
    import * as d3 from 'd3';

    export function stackedBarChart(ap_data) {
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

        const margin = { top: 20, right: 20, bottom: 30, left: 40 };
        const width = parseInt(d3.select('body').style('width'), 10) - margin.left - margin.right;
        const height = parseInt(d3.select('body').style('height'), 10) - margin.top - margin.bottom;

        const x = d3.scaleBand()
            .range([0, width])
            .padding(0.1);

        const y = d3.scaleLinear()
            .range([height, 0]);

        const colorRange = d3.scaleOrdinal(d3.schemeCategory10);
        const color = d3.scaleBand()
            .range(colorRange.range());

        const xAxis = d3.axisBottom(x);

        const yAxis = d3.axisLeft(y)
            .tickFormat(d3.format(".2s"));


        const svg = d3.select("body").append("svg")
            .attr("width", width + margin.left + margin.right)
            .attr("height", height + margin.top + margin.bottom)
            .append("g")
            .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

        const divTooltip = d3.select("body").append("div").attr("class", "toolTip");

        color.domain(Object.keys(dataset[0]).filter(function (key) { return key !== "label"; }));

        dataset.forEach(function (d) {
            let y0 = 0;
            d.values = color.domain().map(function (name) { return { name: name, y0: y0, y1: y0 += +d[name] }; });
            d.total = d.values[d.values.length - 1].y1;
        });

        x.domain(dataset.map(function (d) { return d.label; }));
        y.domain([0, d3.max(dataset, function (d) { return d.total; })]);

        svg.append("g")
            .attr("class", "x axis")
            .attr("transform", "translate(0," + height + ")")
            .call(xAxis);

        svg.append("g")
            .attr("class", "y axis")
            .call(yAxis)
            .append("text")
            .attr("transform", "rotate(-90)")
            .attr("y", 9)
            .attr("dy", ".71em")
            .style("text-anchor", "end")
            .text("Satisfaction %");

        const bar = svg.selectAll(".label")
            .data(dataset)
            .enter().append("g")
            .attr("class", "g")
            .attr("transform", function (d) { return "translate(" + x(d.label) + ",0)"; });

        svg.selectAll(".x.axis .tick text")
            .call(wrap, x.bandwidth());

        const bar_enter = bar.selectAll("rect")
            .data(function (d) { return d.values; })
            .enter();

        bar_enter.append("rect")
            .attr("width", x.bandwidth())
            .attr("y", function (d) { return y(d.y1); })
            .attr("height", function (d) { return y(d.y0) - y(d.y1); })
            .style("fill", function (d) { return color(d.name); });

        bar_enter.append("text")
            .text(function (d) { return d3.format(".2s")(d.y1 - d.y0) + "%"; })
            .attr("y", function (d) { return y(d.y1) + (y(d.y0) - y(d.y1)) / 2; })
            .attr("x", x.bandwidth() / 3)
            .style("fill", '#ffffff');

        bar
            .on("mousemove", function (d) {
                divTooltip.style("left", d3.event.pageX + 10 + "px");
                divTooltip.style("top", d3.event.pageY - 25 + "px");
                divTooltip.style("display", "inline-block");
                const elements = document.querySelectorAll(':hover');
                let l = elements.length;
                l = l - 1;
                const element = elements[l].__data__;
                const value = element.y1 - element.y0;
                divTooltip.html((d.label) + "<br>" + element.name + "<br>" + value + "%");
            });

        bar
            .on("mouseout", function (d) {
                divTooltip.style("display", "none");
            });

        svg.append("g")
            .attr("class", "legendLinear")
            .attr("transform", "translate(0," + (height + 30) + ")");

        const legend = d3.legend.color()
            .shapeWidth(height / 4)
            .shapePadding(10)
            .orient('horizontal')
            .scale(color);

        svg.select(".legendLinear")
            .call(legend);

        return svg.node();
    }

</script>