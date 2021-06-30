---
title: "mot millage"
summary: ""
authors: ["thomas"]
tags: ["car", "graph"]
categories: []
date: 2021-07-30 23:29:00
---
## Mot Millage

<script type="text/javascript" src="https://www.gstatic.com/charts/loader.js"></script>
<script type="text/javascript">
        google.charts.load('current', {'packages':['corechart']});
        google.charts.setOnLoadCallback(drawChart);

        function drawChart() {
        var data = google.visualization.arrayToDataTable([
                ['Date', 'Distance'],
                [ new Date('2008-02-21'), 8651],
                [ new Date('2009-02-27'), 8776],
                [ new Date('2010-02-17'), 10263],
                [ new Date('2011-02-25'), 9285],
                [ new Date('2012-02-17'), 8908],
                [ new Date('2013-02-13'), 8465],
                [ new Date('2014-02-21'), 8780],
                [ new Date('2015-03-26'), 12834],
                [ new Date('2016-03-15'), 9928],
                [ new Date('2017-03-22'), 11385],
                [ new Date('2018-03-21'), 10665],
                [ new Date('2019-03-20'), 11831],
                [ new Date('2020-03-17'), 8644],
                [ new Date('2021-03-27'), 4928]
        ]);

        var options = {
                title: 'Difference between annual MOT millage',
                hAxis: {title: 'Year'},
                vAxis: {title: 'Distance (miles)'},
                legend: 'none'
        };

        var chart = new google.visualization.ScatterChart(document.getElementById('chart_div'));
        chart.draw(data, options);
      }

</script>
<div id="chart_div" style="width: 900px; height: 500px;"></div>

