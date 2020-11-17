---
title: Central limit theorem
author: Emerson Harkin
---

<style>
    rect {
        fill: blue;
        rx: 3pt;
    }

    rect:hover {
        stroke: black;
        stroke-width: 3.5;
    }

    .flex-row {
        display: flex;
    }

    .flex-col {
        flex: 1;
    }
</style>

<p>There is a <span id="tailprob"></span>% probability of getting <span id="totalroll"></span> or lower. <span id="reaction"></span></p>
<div id="distplot"></div>

<!-- Sliders for entering the dice to roll. -->

<!-- First row of sliders. -->
<div class="flex-row">
    <div id="dfour" class="slidercontainer flex-col">
        <p><span class="slidervalue"></span>D4</p>
        <input type="range" min="0" max="8" value="0" class="slider" data-numsides="4">
    </div>
    <div id="dsix" class="slidercontainer flex-col">
        <p><span class="slidervalue"></span>D6</p>
        <input type="range" min="0" max="8" value="0" class="slider" data-numsides="6">
    </div>
    <div id="deight" class="slidercontainer flex-col">
        <p><span class="slidervalue"></span>D8</p>
        <input type="range" min="0" max="8" value="0" class="slider" data-numsides="8">
    </div>
</div>

<!-- Second row of sliders. -->
<div class="flex-row">
    <div id="dten" class="slidercontainer flex-col">
        <p><span class="slidervalue"></span>D10</p>
        <input type="range" min="0" max="8" value="0" class="slider" data-numsides="10">
    </div>
    <div id="dtwelve" class="slidercontainer flex-col">
        <p><span class="slidervalue"></span>D12</p>
        <input type="range" min="0" max="8" value="0" class="slider" data-numsides="12">
    </div>
    <div id="dtwenty" class="slidercontainer flex-col">
        <p><span class="slidervalue"></span>D20</p>
        <input type="range" min="0" max="8" value="0" class="slider" data-numsides="20">
    </div>
</div>


<script>
const chartSize = {
    height: 400,
    width: 600,
    margin: 50,
}


var chart = d3.selectAll("#distplot")
    .append("svg");
chart
    .attr("width", chartSize.width)
    .attr("height", chartSize.height);

xAxis = function(g) {
    g.attr("transform", `translate(0, ${chartSize.height - chartSize.margin/2})`)
        .call(d3.axisBottom(x))
}
yAxis = function(g) {
    g.attr("transform", `translate(${chartSize.margin/2}, 0)`)
        .call(d3.axisLeft(y))
}


const formatFloat = d3.format(".1f");

updateChart = function() {
    let pmf = calculatePMF();

    let indexedPMF = [];
    for (i=0; i<pmf.probabilities.length; i++) {
        indexedPMF.push({
            x: i + pmf.baseValue,
            prob: pmf.probabilities[i],
            cumProb: pmf.cumProbabilities[i],
        })
    }

    let x = d3.scaleLinear()
        .domain([0, pmf.probabilities.length + pmf.baseValue])
        .range([chartSize.margin, chartSize.width - chartSize.margin]);
    let y = d3.scaleLinear()
        .domain([0, Math.max(...pmf.probabilities) + 0.03])
        .range([chartSize.height-chartSize.margin, chartSize.margin]);

    chart.selectAll("g").remove();

    chart.append("g").call(function(g) {
        g.attr("transform", `translate(${chartSize.margin/2}, 0)`)
        .call(d3.axisLeft(y))
    });
    chart.append("g").call(function(g) {
        g.attr("transform", `translate(0, ${chartSize.height - chartSize.margin/2})`)
        .call(d3.axisBottom(x))
    });

    let barWidth = chartSize.width / (indexedPMF.length + 8) - 5;
    let mygroup = chart.append("g");
    mygroup.selectAll("rect").data(indexedPMF).enter()
        .append("rect")
        .attr("x", (d) => {return x(d.x) - 0.5 * barWidth})
        .attr("y", (d) => {return y(d.prob)})
        .attr("width", barWidth)
        .attr("height", (d) => {return y(0) - y(d.prob)})
        .on("mouseover", (d) => {
            d3.select("#tailprob").text(formatFloat(100 * d.cumProb));
            d3.select("#totalroll").text(d.x);
            if (Math.abs(d.cumProb - 0.5) < 0.001) {
                d3.select("#reaction").text("Glass half empty?");
            } else if (d.cumProb < 0.5) {
                d3.select("#reaction").text("Unlucky!");
            } else {
                d3.select("#reaction").text("Lucky!");
            }
        });
}

calculatePMF = function() {
    // Get an array of dice from sliders on page.
    // Each entry in the array represents one die, and the vlaue in the array is
    // the number of sides on the die.
    let dice = d3.selectAll(".slider")
        .nodes()
        .map(function(d){
            let theseDice = [];
            theseDice.length = +d.value;
            theseDice.fill(+d.dataset.numsides);
            return theseDice;
        });
    dice = dice.flat();

    let rollPMF;
    if (dice.length == 0) {
        rollPMF = {
            probabilities: [],
            cumProbabilities: [],
            baseValue: 0
        }
    } else {
        rollPMF = pmfConvolve(diePMF(dice.pop()), dice, 1);
    }
    return rollPMF;
}


pmfConvolve = function(probabilities, dice, baseValue) {
    // Return the probability vector in the base case.
    if (dice.length == 0) {
        const cumulativeSum = (sum => value => sum += value)(0);
        let rollPMF = {
            probabilities: probabilities,
            cumProbabilities: probabilities.map(cumulativeSum),
            baseValue: baseValue
        }
        return rollPMF;
    }

    let thisDiePMF = diePMF(dice.pop());
    let convolvedPMF = [];
    convolvedPMF.length = probabilities.length + thisDiePMF.length - 1;

    let shorterPMF = [];
    let longerPMF = [];
    if (probabilities.length < thisDiePMF.length) {
        shorterPMF = probabilities;
        longerPMF = thisDiePMF;
    } else {
        shorterPMF = thisDiePMF;
        longerPMF = probabilities;
    }

    // Convolve the uniform distribution from the next die with the existing
    // probability vector.
    for (i = 0; i < convolvedPMF.length; i++) {
        let probMass = 0.0;
        for (j = shorterPMF.length - 1; j >= 0; j--) {
            if ((i - j < longerPMF.length) & (i - j >= 0)) {
                probMass += longerPMF[i-j] * shorterPMF[j];
            }
        }
        convolvedPMF[i] = probMass;
    }

    return pmfConvolve(convolvedPMF, dice, baseValue + 1);
}

// Get the probability mass function for a die.
// Returns an array representing a uniform probability distribution.
diePMF = function(numSides) {
    let pmf = [];
    pmf.length = numSides;
    pmf.fill(1.0 / numSides);
    return pmf;
}

d3.selectAll(".slidercontainer")
    .each(function() {
        var sliderValue = d3.select(this).select("span.slidervalue");
        d3.select(this)
            .select("input")
            .each(function() {sliderValue.text(this.value)})
            .on("input", function() {
                sliderValue.text(this.value);
            })
            .on("mouseup", updateChart)
        ;
});

updateChart();
</script>
