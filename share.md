To tame the heatmap overflow and get a single‐color gradient, update your `BuildHeatmapOptions` like this:

```csharp
private object BuildHeatmapOptions(ChartDataCache c)
{
    // Extract categories
    var xCats = c.Data.Select(d => (string)d[0]).Distinct().ToArray();
    var yCats = c.Data.Select(d => (string)d[1]).Distinct().ToArray();

    // Prepare min/max for color scale
    var values = c.Data.Select(d => Convert.ToDecimal(d[2])).ToArray();
    var minVal = (double)values.Min();
    var maxVal = (double)values.Max();

    // Series per Y category
    var series = yCats.Select(y => new
    {
        name = y,
        data = xCats.Select(x =>
        {
            var match = c.Data.FirstOrDefault(d => (string)d[0] == x && (string)d[1] == y);
            var z = match != null ? Convert.ToDouble(match[2]) : 0.0;
            return new { x, y, value = z };
        }).ToArray()
    }).ToArray();

    // Dynamic height: e.g. 30px per row, min 350px
    var chartHeight = Math.Max(350, yCats.Length * 30);

    return new
    {
        chart = new
        {
            type   = "heatmap",
            height = chartHeight,
            toolbar = new { show = true }
        },
        plotOptions = new
        {
            heatmap = new
            {
                shadeIntensity      = 0.5,
                radius              = 0,
                useFillColorAsStroke = false,
                distributed         = false,
                colorScale = new
                {
                    ranges = new[] {
                        new { from = minVal, to = maxVal, name = "", color = "#008FFB" }
                    }
                }
            }
        },
        dataLabels = new { enabled = false },
        stroke     = new { width = 0 },
        xaxis = new
        {
            type       = "category",
            categories = xCats,
            labels     = new { rotate = -45, style = new { fontSize = "12px" } }
        },
        yaxis = new
        {
            type   = "category",
            labels = new { style = new { fontSize = "12px" } }
        },
        series = series
    };
}
```

### What this does

* **Dynamic height** (`chartHeight`) prevents overflow by sizing the chart to the number of rows.
* **Single‐color gradient**: the `colorScale.ranges` with one entry gives you a uniform color ramp (`#008FFB`).
* **No multi‐color**: with only one range, every cell uses a gradient of that color.
* **Compact cells**: `radius = 0` and turning off strokes keeps each cell tight.
* **Label rotation** (`rotate = -45`) makes dense category labels legible.

Drop this into your `ChartBlock.razor` in place of the old heatmap builder—and the container should stay tidy, scrollable (if you keep your CSS), and consistently colored.
