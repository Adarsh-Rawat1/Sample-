Here’s the **updated** `BuildLineOptions` showing **small, colored markers** on your main series and **no markers** on the EMA line:

```csharp
private object BuildLineOptions(ChartDataCache c)
{
    // Prepare timestamps and values
    var timestamps = c.Data
        .Select(d => DateTime.Parse((string)d[0])
                         .ToUniversalTime()
                         .ToUnixTimeMilliseconds())
        .ToArray();
    var values = c.Data.Select(d => Convert.ToDecimal(d[1])).ToArray();

    // Main series with small blue markers
    var seriesList = new List<object>
    {
        new
        {
            name    = Definition.Title,
            type    = "line",
            data    = timestamps
                      .Zip(values, (x, y) => new object[] { x, y })
                      .ToArray(),
            stroke  = new { width = 2, curve = "smooth" },
            markers = new
            {
                size   = 4,              // small dots
                colors = new[] { "#008FFB" } // custom color
            }
        }
    };

    // EMA series, no markers
    if (c.EmaSeries is { Count: > 0 } ema)
    {
        seriesList.Add(new
        {
            name    = "EMA (6)",
            type    = "line",
            data    = timestamps
                      .Zip(ema, (x, e) => new object[] { x, e })
                      .ToArray(),
            stroke  = new { width = 1, dashArray = 4 },
            markers = new { size = 0 }  // hide markers
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
                style                 = new { fontSize = "12px" }
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

**Highlights:**

* **Main line markers**: `size = 4`, `colors = ["#008FFB"]`
* **EMA markers**: `size = 0` (hidden)
* **Stroke widths**: `2` for main, `1` dashed for EMA

Drop this into your `ChartBlock.razor` in place of the old builder, and you’ll get crisp, colored dots on your data points—and no dots crowding the EMA.
