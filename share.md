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







```

using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using StarTrendsDashboard.Shared;
using Oracle.ManagedDataAccess.Client;
using System;
using System.Collections.Generic;
using System.Data;
using System.IO;
using System.Linq;
using System.Text.Json;
using System.Threading.Tasks;

namespace StarTrendsDashboard.Services
{
    public class ChartService : IChartService
    {
        private readonly string _definitionsJsonPath;
        private readonly string _queryFolder;
        private readonly string _connectionString;
        private readonly string _cacheFolder;
        private readonly ILogger<ChartService> _logger;

        private readonly List<ChartDefinition> _definitions = new();
        private DateTime _defsLastWriteTimeUtc;
        private readonly object _lock = new();

        public ChartService(IConfiguration config, ILogger<ChartService> logger)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            _definitionsJsonPath = config["ChartDefinitionsPath"] ?? throw new ArgumentNullException("ChartDefinitionsPath missing");
            _queryFolder = config["ChartQueryFolder"] ?? throw new ArgumentNullException("ChartQueryFolder missing");
            _cacheFolder = config["ChartCacheFolder"] ?? throw new ArgumentNullException("ChartCacheFolder missing");
            _connectionString = config.GetConnectionString("OracleDb") ?? throw new ArgumentNullException("OracleDb missing");

            Directory.CreateDirectory(_cacheFolder);
            LoadDefinitions();
        }

        private void LoadDefinitions()
        {
            var fi = new FileInfo(_definitionsJsonPath);
            if (!fi.Exists)
                throw new FileNotFoundException($"Cannot find {_definitionsJsonPath}");

            if (fi.LastWriteTimeUtc <= _defsLastWriteTimeUtc && _definitions.Count > 0)
                return;

            var json = File.ReadAllText(_definitionsJsonPath);
            var defs = JsonSerializer.Deserialize<List<ChartDefinition>>(json) ?? new List<ChartDefinition>();

            lock (_lock)
            {
                _definitions.Clear();
                _definitions.AddRange(defs);
                _defsLastWriteTimeUtc = fi.LastWriteTimeUtc;
            }

            // extract category from SQL header if present
            foreach (var def in _definitions)
            {
                var sqlPath = Path.Combine(_queryFolder, def.SqlFile);
                if (File.Exists(sqlPath))
                {
                    var parts = File.ReadLines(sqlPath).FirstOrDefault()?.Split(',') ?? Array.Empty<string>();
                    // category is always the last element in header
                    def.Category = parts.Length > (def.Dimensions == 3 ? 3 : 2)
                        ? parts[def.Dimensions].Trim()
                        : string.Empty;
                }
            }

            _logger.LogInformation("Loaded {Count} chart definitions", _definitions.Count);
        }

        public IReadOnlyList<ChartDefinition> GetAllDefinitions()
        {
            LoadDefinitions();
            lock (_lock) { return _definitions.ToList(); }
        }

        public async Task<ChartDataCache> RefreshChartIfNeededAsync(string chartId)
        {
            LoadDefinitions();
            ChartDefinition? def;
            lock (_lock)
            {
                def = _definitions.FirstOrDefault(d => d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
            }
            if (def == null)
                return await RefreshChartAsync(chartId);

            var outPath = Path.Combine(_cacheFolder, $"{chartId}.json");
            if (File.Exists(outPath))
            {
                try
                {
                    var existing = JsonSerializer.Deserialize<ChartDataCache>(await File.ReadAllTextAsync(outPath))!;
                    if ((DateTime.UtcNow - existing.LastUpdatedUtc).TotalSeconds < def.RefreshIntervalSeconds)
                    {
                        _logger.LogInformation("Using cached data for '{chartId}'", chartId);
                        return existing;
                    }
                }
                catch (Exception ex)
                {
                    _logger.LogWarning(ex, "Failed reading cache for '{chartId}', regenerating", chartId);
                }
            }
            return await RefreshChartAsync(chartId);
        }

        public async Task<ChartDataCache> RefreshChartAsync(string chartId)
        {
            LoadDefinitions();
            ChartDefinition? def;
            lock (_lock)
            {
                def = _definitions.FirstOrDefault(d => d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
            }
            if (def == null)
            {
                _logger.LogWarning("No definition for '{chartId}'", chartId);
                return new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };
            }

            var sqlPath = Path.Combine(_queryFolder, def.SqlFile);
            if (!File.Exists(sqlPath))
            {
                _logger.LogError("SQL file not found: {sqlPath}", sqlPath);
                return new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };
            }

            var content = await File.ReadAllTextAsync(sqlPath);
            var lines = content.Split(new[] { "\r\n", "\n" }, StringSplitOptions.None);

            // header: xLabel, yLabel, (zLabel if 3D), (category)
            var header = lines[0].Split(',');
            var xLabel = header.ElementAtOrDefault(0)?.Trim() ?? string.Empty;
            var yLabel = header.ElementAtOrDefault(1)?.Trim() ?? string.Empty;
            string? zLabel = def.Dimensions == 3
                ? header.ElementAtOrDefault(2)?.Trim()
                : null;
            var category = header.ElementAtOrDefault(def.Dimensions)?.Trim() ?? string.Empty;

            // optional summary
            string? summary = null;
            var sqlStart = 1;
            if (lines.Length > 1)
            {
                var line2 = lines[1].Trim();
                if (line2.StartsWith("{") && line2.EndsWith("}"))
                {
                    summary = line2.Trim('{', '}').Trim();
                    sqlStart = 2;
                }
            }

            // rebuild SQL and trim trailing semicolon
            var rawSql = string.Join("\n", lines.Skip(sqlStart)).TrimEnd();
            if (rawSql.EndsWith(";"))
                rawSql = rawSql[..^1].TrimEnd();

            var data = new List<object[]>();
            try
            {
                using var conn = new OracleConnection(_connectionString);
                await conn.OpenAsync();
                using var cmd = conn.CreateCommand();
                cmd.CommandText = rawSql;
                cmd.CommandType = CommandType.Text;

                using var reader = await cmd.ExecuteReaderAsync();
                while (await reader.ReadAsync())
                {
                    if (def.Dimensions == 3)
                    {
                        var xVal = reader.IsDBNull(0) ? string.Empty : reader.GetString(0);
                        var yVal = reader.IsDBNull(1) ? string.Empty : reader.GetString(1);
                        var zVal = reader.IsDBNull(2) ? 0m : Convert.ToDecimal(reader.GetValue(2));
                        data.Add(new object[] { xVal, yVal, zVal });
                    }
                    else
                    {
                        var xVal = reader.IsDBNull(0) ? string.Empty : reader.GetString(0);
                        var yNum = reader.IsDBNull(1) ? 0m : Convert.ToDecimal(reader.GetValue(1));
                        data.Add(new object[] { xVal, yNum });
                    }
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error executing SQL for '{chartId}'", chartId);
            }

            decimal? avg = null;
            if (def.ShowAverageLine && def.Dimensions == 2 && data.Count > 0)
                avg = data.Average(r => Convert.ToDecimal(r[1]));

            var cache = new ChartDataCache
            {
                ChartId        = chartId,
                LastUpdatedUtc = DateTime.UtcNow,
                Data           = data,
                XLabel         = xLabel,
                YLabel         = yLabel,
                ZLabel         = zLabel,
                Category       = category,
                Summary        = summary,
                AverageValue   = avg
            };

            var outJson = JsonSerializer.Serialize(cache, new JsonSerializerOptions { WriteIndented = true });
            await File.WriteAllTextAsync(Path.Combine(_cacheFolder, $"{chartId}.json"), outJson);

            _logger.LogInformation("Refreshed '{chartId}' → {Count} data points, wrote cache", chartId, data.Count);
            return cache;
        }
    }
}

```
