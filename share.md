```

private object BuildLineOptions(ChartDataCache c)
{
    // Base series (your actual data)
    var mainDates  = c.Data.Select(d => DateTime.Parse((string)d[0]).ToUniversalTime().ToUnixTimeMilliseconds()).ToArray();
    var mainValues = c.Data.Select(d => Convert.ToDecimal(d[1])).ToArray();

    // Build the series list
    var seriesList = new List<object>
    {
        new
        {
            name = Definition.Title,
            type = "line",
            data = mainDates
                   .Zip(mainValues, (x, y) => new object[] { x, y })
                   .ToArray()
        }
    };

    // If AverageValue is present, append a flat average‐line series
    if (c.AverageValue.HasValue)
    {
        var avgVal   = c.AverageValue.Value;
        var avgSeries = mainDates
                        .Select(x => new object[] { x, avgVal })
                        .ToArray();

        seriesList.Add(new
        {
            name       = "Average",
            type       = "line",
            data       = avgSeries,
            stroke     = new { width = 1, dashArray = 4 },
            markers    = new { size = 0 }
        });
    }

    return new
    {
        chart = new
        {
            id     = ElementId,
            type   = "line",
            height = 350,
            toolbar = new { show = true },
            zoom    = new { enabled = true }
        },
        stroke = new { width = 2, curve = "smooth" },
        markers = new { size = 4 },
        dataLabels = new { enabled = false },

        xaxis = new
        {
            type       = "datetime",
            tickAmount = 8,
            labels     = new
            {
                format                = "dd MMM HH:mm",
                rotate                = -45,
                hideOverlappingLabels = true,
                datetimeUTC           = false,
                style = new { fontSize = "12px" }
            },
            title = new { text = c.XLabel, style = new { fontSize = "14px" } }
        },
        yaxis = new
        {
            title  = new { text = c.YLabel, style = new { fontSize = "14px" } },
            labels = new { style = new { fontSize = "12px" } }
        },

        series = seriesList.ToArray()
    };
}
```
