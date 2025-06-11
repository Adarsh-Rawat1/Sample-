Below are the necessary file changes to support 3‑column SQL (converted to ApexCharts heatmap) and line charts with optional average lines.

---

### 1. `ChartDefinition.cs`

```csharp
namespace StarTrendsDashboard.Shared;

public class ChartDefinition
{
    public string ChartId { get; set; } = "";
    public string Title   { get; set; } = "";
    public string SqlFile { get; set; } = "";

    // Controls rendering
    public string ChartType { get; set; } = "bar";        // "bar", "line", "heatmap", etc.
    public int    RefreshIntervalSeconds { get; set; } = 21600; // default 6 hrs

    // New:
    public int    Dimensions      { get; set; } = 2;    // 2 = [label,value], 3 = [x,y,value]
    public bool   ShowAverageLine { get; set; } = false;
}
```

---

### 2. `ChartDataCache.cs`

```csharp
namespace StarTrendsDashboard.Shared;

public class ChartDataCache
{
    public string ChartId         { get; set; } = "";
    public DateTime LastUpdatedUtc { get; set; }

    // Holds 2‑ or 3‑element points: [label, value] or [x, y, value]
    public List<object[]> Data     { get; set; } = new();

    public string XLabel  { get; set; } = "";
    public string YLabel  { get; set; } = "";
    public string? ZLabel { get; set; }        // only used when Dimensions==3
    public string Category { get; set; } = "";
    public string? Summary { get; set; }

    // For line charts with an average overlay
    public decimal? AverageValue { get; set; }
}
```

---

### 3. `ChartService.cs` (inside `RefreshChartAsync`)

Replace your existing row‐reading logic with this block to populate `Data` and compute `AverageValue`:

```csharp
// after parsing headers (xLabel, yLabel, possibly zLabel) and stripping `;`
var rows3D = new List<object[]>();
while (await reader.ReadAsync())
{
    var x = reader.GetString(0);
    var yVal = Convert.ToDecimal(reader.GetValue(1));

    if (def.Dimensions == 3)
    {
        var zVal = Convert.ToDecimal(reader.GetValue(2));
        rows3D.Add(new object[] { x, reader.GetString(1), zVal });
    }
    else
    {
        rows3D.Add(new object[] { x, yVal });
    }
}

// compute average of the second numeric column if requested
decimal? avg = null;
if (def.ShowAverageLine && def.Dimensions == 2 && rows3D.Count > 0)
{
    avg = rows3D.Average(r => Convert.ToDecimal(r[1]));
}

var cache = new ChartDataCache
{
    ChartId       = chartId,
    LastUpdatedUtc = DateTime.UtcNow,
    Data          = rows3D,
    XLabel        = xLabel,
    YLabel        = yLabel,
    ZLabel        = def.Dimensions == 3 ? zLabel : null,
    Category      = category,
    Summary       = summary,
    AverageValue  = avg
};

// serialize & write JSON as before
```

---

### 4. `ChartBlock.razor` (add heatmap & average‐line builders)

In your options switch, add:

```csharp
chartOptions = Definition.ChartType.ToLower() switch
{
  "bar"     => BuildBarOptions(cache),
  "line"    => BuildLineOptions(cache),
  "heatmap" => BuildHeatmapOptions(cache),
  _          => BuildBarOptions(cache)
};
```

Then below your existing builders, append:

```csharp
private object BuildHeatmapOptions(ChartDataCache c)
{
    // extract distinct axes
    var xCats = c.Data.Select(d => (string)d[0]).Distinct().ToArray();
    var yCats = c.Data.Select(d => (string)d[1]).Distinct().ToArray();

    // series per Y category
    var series = yCats.Select(y => new
    {
        name = y,
        data = xCats.Select(x => new
        {
            x,
            y = (decimal?)c.Data
                  .FirstOrDefault(d => (string)d[0] == x && (string)d[1] == y)?[2]
                  ?? 0m
        }).ToArray()
    }).ToArray();

    return new
    {
        chart = new { type = "heatmap", height = 350 },
        plotOptions = new { heatmap = new { shadeIntensity = 0.5, radius = 2 } },
        dataLabels = new { enabled = false },
        xaxis = new { type = "category", categories = xCats },
        yaxis = new { type = "category" },
        series = series
    };
}

private object BuildLineOptions(ChartDataCache c)
{
    var labels = c.Data.Select(d => (string)d[0]).ToArray();
    var values = c.Data.Select(d => (decimal)d[1]).ToArray();

    // main series
    var list = new List<object>
    {
        new { name = Definition.Title, type = "line", data = values }
    };

    // optional average line
    if (Definition.ShowAverageLine && c.AverageValue.HasValue)
    {
        list.Add(new
        {
            name = "Average",
            type = "line",
            data = Enumerable.Repeat(c.AverageValue.Value, values.Length).ToArray(),
            stroke = new { dashArray = 4 }
        });
    }

    return new
    {
        chart = new { id = ElementId, type = "line", height = 350, toolbar = new { show = true }, zoom = new { enabled = true } },
        stroke = new { curve = "smooth" },
        dataLabels = new { enabled = false },
        xaxis = new { categories = labels, title = new { text = c.XLabel } },
        yaxis = new { title = new { text = c.YLabel } },
        series = list.ToArray()
    };
}
```

---

### 5. Sample `chart-definitions.json` for a heatmap chart

```json
{
  "ChartId": "OtcRecBreaksHeatmap",
  "Title": "OTC intra-day rec breaks per week by reason",
  "ChartType": "heatmap",
  "SqlFile": "OTC intra-day rec breaks per week by break reason (in the last 12 months).sql",
  "Dimensions": 3,
  "ShowAverageLine": false,
  "RefreshIntervalSeconds": 21600
}
```

With these changes, any SQL with 3 header columns (x,y,value) will automatically render as a heatmap, and 2‑column SQL can render as bar/line (with optional average). Let me know if you need further tweaks!
