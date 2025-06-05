Below is a complete list of every file (with its final contents) that you need in order to build and run the solution exactly as described. For any file names under `/wwwroot/lib/apexcharts/` (namely `apexcharts.min.js` and `apexcharts.css`), simply copy them from the ZIP you have—do not try to hand‐type the minified code. Every other file below contains the full contents you should paste.

---

## 1. `appsettings.json`

**Path:**

```
/StarTrendsDashboard/appsettings.json
```

```jsonc
{
  "ConnectionStrings": {
    "OracleDb": "User Id=MYUSER;Password=MYPASSWORD;Data Source=//myhost:1521/ORCLPDB1"
  },
  "ChartDefinitionsPath": "ChartDefinitions/chart‐definitions.json",
  "ChartQueryFolder": "ChartDefinitions/Queries"
}
```

> Replace `"User Id=MYUSER;Password=MYPASSWORD;Data Source=//myhost:1521/ORCLPDB1"` with your actual Oracle connection string.

---

## 2. `Program.cs`

**Path:**

```
/StarTrendsDashboard/Program.cs
```

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using StarTrendsDashboard.Services;

var builder = WebApplication.CreateBuilder(args);

// 1. Register ChartService & PollingService
builder.Services.AddSingleton<IChartService, ChartService>();
builder.Services.AddHostedService<ChartPollingService>();

// 2. Add Blazor services
builder.Services.AddRazorPages();
builder.Services.AddServerSideBlazor();

var app = builder.Build();

// 3. Standard middleware
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}
app.UseStaticFiles();
app.UseRouting();
app.MapBlazorHub();
app.MapFallbackToPage("/_Host");
app.Run();
```

---

## 3. Chart Definitions & SQL Files

### 3.1. `chart‐definitions.json`

**Path:**

```
/StarTrendsDashboard/ChartDefinitions/chart‐definitions.json
```

```jsonc
[
  {
    "ChartId": "ProductMarkets",
    "Page": "Product",
    "Title": "Markets Set in Last 30 Days",
    "ChartType": "Bar",
    "SqlFile": "ProductMarkets.sql",
    "RefreshIntervalSeconds": 300
  },
  {
    "ChartId": "TradeTools",
    "Page": "Trade",
    "Title": "Tools Used in Last 30 Days",
    "ChartType": "Bar",
    "SqlFile": "TradeTools.sql",
    "RefreshIntervalSeconds": 300
  }
]
```

---

### 3.2. `ProductMarkets.sql`

**Path:**

```
/StarTrendsDashboard/ChartDefinitions/Queries/ProductMarkets.sql
```

```sql
-- ProductMarkets.sql
SELECT
  r.feature AS "Markets",
  COUNT(1) AS "Times used"
FROM star_action_audit r
WHERE r.mod_dt > TRUNC(SYSDATE) - 30
  AND feature_type IN ('MARKET')
GROUP BY r.feature
ORDER BY "Times used" DESC;
```

---

### 3.3. `TradeTools.sql`

**Path:**

```
/StarTrendsDashboard/ChartDefinitions/Queries/TradeTools.sql
```

```sql
-- TradeTools.sql
SELECT
  r.feature AS "Tools",
  COUNT(1) AS "Times used"
FROM star_action_audit r
WHERE r.mod_dt > TRUNC(SYSDATE) - 30
  AND feature_type IN ('TOOLS')
GROUP BY r.feature
ORDER BY "Times used" DESC;
```

---

## 4. Model Classes

### 4.1. `ChartDefinition.cs`

**Path:**

```
/StarTrendsDashboard/Models/ChartDefinition.cs
```

```csharp
namespace StarTrendsDashboard.Models
{
    public class ChartDefinition
    {
        public string ChartId { get; set; } = string.Empty;
        public string Page { get; set; } = string.Empty;
        public string Title { get; set; } = string.Empty;
        public string ChartType { get; set; } = string.Empty;
        public string SqlFile { get; set; } = string.Empty;
        public int RefreshIntervalSeconds { get; set; } = 300;
    }
}
```

---

### 4.2. `ChartDataRow.cs`

**Path:**

```
/StarTrendsDashboard/Models/ChartDataRow.cs
```

```csharp
namespace StarTrendsDashboard.Models
{
    public class ChartDataRow
    {
        public string Label { get; set; } = string.Empty;
        public decimal Value { get; set; }
    }
}
```

---

### 4.3. `ChartDataCache.cs`

**Path:**

```
/StarTrendsDashboard/Models/ChartDataCache.cs
```

```csharp
using System;
using System.Collections.Generic;

namespace StarTrendsDashboard.Models
{
    public class ChartDataCache
    {
        public string ChartId { get; set; } = string.Empty;
        public DateTime LastUpdatedUtc { get; set; }
        public List<ChartDataRow> Rows { get; set; } = new List<ChartDataRow>();
    }
}
```

---

## 5. Service Layer

### 5.1. `ChartService.cs`

**Path:**

```
/StarTrendsDashboard/Services/ChartService.cs
```

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Oracle.ManagedDataAccess.Client;
using StarTrendsDashboard.Models;
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Data;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Newtonsoft.Json;

namespace StarTrendsDashboard.Services
{
    public interface IChartService
    {
        IReadOnlyList<ChartDefinition> GetAllDefinitions();
        IReadOnlyList<ChartDefinition> GetDefinitionsByPage(string pageName);
        ChartDataCache GetCachedData(string chartId);
        Task<ChartDataCache> RefreshChartAsync(string chartId);
    }

    public class ChartService : IChartService
    {
        private readonly string _definitionsJsonPath;
        private readonly string _queryFolder;
        private readonly string _connectionString;
        private readonly ILogger<ChartService> _logger;

        private readonly List<ChartDefinition> _definitions = new();
        private readonly object _definitionsLock = new();
        private readonly ConcurrentDictionary<string, ChartDataCache> _cache
            = new(StringComparer.OrdinalIgnoreCase);

        public ChartService(IConfiguration configuration, ILogger<ChartService> logger)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));

            _definitionsJsonPath = configuration["ChartDefinitionsPath"]
                ?? throw new ArgumentNullException("ChartDefinitionsPath missing in config");
            _queryFolder = configuration["ChartQueryFolder"]
                ?? throw new ArgumentNullException("ChartQueryFolder missing in config");
            _connectionString = configuration.GetConnectionString("OracleDb")
                ?? throw new ArgumentNullException("OracleDb connection string missing");

            if (!File.Exists(_definitionsJsonPath))
                throw new FileNotFoundException($"Cannot find JSON: {_definitionsJsonPath}");

            var json = File.ReadAllText(_definitionsJsonPath);
            var defs = JsonConvert.DeserializeObject<List<ChartDefinition>>(json)
                       ?? new List<ChartDefinition>();

            lock (_definitionsLock)
            {
                foreach (var def in defs)
                {
                    _definitions.Add(def);
                    _cache[def.ChartId] = new ChartDataCache
                    {
                        ChartId = def.ChartId,
                        LastUpdatedUtc = DateTime.MinValue,
                        Rows = new List<ChartDataRow>()
                    };
                }
            }
        }

        public IReadOnlyList<ChartDefinition> GetAllDefinitions()
        {
            lock (_definitionsLock)
            {
                return _definitions.Select(d => d).ToList();
            }
        }

        public IReadOnlyList<ChartDefinition> GetDefinitionsByPage(string pageName)
        {
            if (pageName is null)
                throw new ArgumentNullException(nameof(pageName));

            lock (_definitionsLock)
            {
                return _definitions
                    .Where(d => d.Page.Equals(pageName, StringComparison.OrdinalIgnoreCase))
                    .Select(d => d)
                    .ToList();
            }
        }

        public ChartDataCache GetCachedData(string chartId)
        {
            if (chartId is null)
                throw new ArgumentNullException(nameof(chartId));

            if (_cache.TryGetValue(chartId, out var existing))
                return existing!;

            var empty = new ChartDataCache
            {
                ChartId = chartId,
                LastUpdatedUtc = DateTime.MinValue,
                Rows = new List<ChartDataRow>()
            };
            _cache[chartId] = empty;
            return empty;
        }

        public async Task<ChartDataCache> RefreshChartAsync(string chartId)
        {
            if (chartId is null)
                throw new ArgumentNullException(nameof(chartId));

            ChartDefinition? def;
            lock (_definitionsLock)
            {
                def = _definitions.FirstOrDefault(d =>
                    d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
            }
            if (def is null)
            {
                _logger.LogWarning($"RefreshChartAsync: No definition for '{chartId}'.");
                return GetCachedData(chartId);
            }

            var sqlPath = Path.Combine(_queryFolder, def.SqlFile);
            if (!File.Exists(sqlPath))
            {
                _logger.LogError($"RefreshChartAsync: SQL not found: {sqlPath}");
                return GetCachedData(chartId);
            }

            var rawSql = await File.ReadAllTextAsync(sqlPath);
            var rows = new List<ChartDataRow>();

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
                    var label = reader.IsDBNull(0)
                        ? string.Empty
                        : reader.GetString(0);
                    var valObj = reader.GetValue(1);
                    decimal val = valObj == DBNull.Value
                        ? 0
                        : Convert.ToDecimal(valObj);

                    rows.Add(new ChartDataRow
                    {
                        Label = label,
                        Value = val
                    });
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Error executing SQL for '{chartId}'.");
                return GetCachedData(chartId);
            }

            var newCache = new ChartDataCache
            {
                ChartId = chartId,
                LastUpdatedUtc = DateTime.UtcNow,
                Rows = rows
            };
            _cache[chartId] = newCache;
            return newCache;
        }
    }
}
```

---

### 5.2. `ChartPollingService.cs`

**Path:**

```
/StarTrendsDashboard/Services/ChartPollingService.cs
```

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using StarTrendsDashboard.Models;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

namespace StarTrendsDashboard.Services
{
    public class ChartPollingService : BackgroundService
    {
        private readonly IChartService _chartService;
        private readonly ILogger<ChartPollingService> _logger;

        private readonly Dictionary<string, DateTime> _nextRefreshUtc
            = new(StringComparer.OrdinalIgnoreCase);

        public ChartPollingService(IChartService chartService, ILogger<ChartPollingService> logger)
        {
            _chartService = chartService;
            _logger = logger;

            var initialDefs = _chartService.GetAllDefinitions();
            foreach (var def in initialDefs)
            {
                _nextRefreshUtc[def.ChartId] = DateTime.UtcNow;
            }
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation("ChartPollingService started.");

            while (!stoppingToken.IsCancellationRequested)
            {
                var now = DateTime.UtcNow;
                var allDefs = _chartService.GetAllDefinitions();

                // Schedule any new definitions discovered (on restart)
                foreach (var def in allDefs)
                {
                    if (!_nextRefreshUtc.ContainsKey(def.ChartId))
                    {
                        _logger.LogInformation($"Scheduling new chart '{def.ChartId}'.");
                        _nextRefreshUtc[def.ChartId] = now;
                    }
                }

                // Refresh each chart when due
                foreach (var def in allDefs)
                {
                    if (_nextRefreshUtc.TryGetValue(def.ChartId, out var nextTime))
                    {
                        if (now >= nextTime)
                        {
                            _logger.LogInformation($"[PollingService] Refreshing '{def.ChartId}'");
                            try
                            {
                                await _chartService.RefreshChartAsync(def.ChartId);
                            }
                            catch (Exception ex)
                            {
                                _logger.LogError(ex, $"[PollingService] Error refreshing '{def.ChartId}'");
                            }
                            _nextRefreshUtc[def.ChartId] = now.AddSeconds(def.RefreshIntervalSeconds);
                        }
                    }
                }

                await Task.Delay(TimeSpan.FromSeconds(60), stoppingToken);
            }

            _logger.LogInformation("ChartPollingService stopping.");
        }
    }
}
```

---

## 6. JavaScript & ApexCharts (in wwwroot)

### 6.1. `apexcharts.min.js`

**Path:**

```
/StarTrendsDashboard/wwwroot/lib/apexcharts/apexcharts.min.js
```

> **Content:** Copy the exact minified `apexcharts.min.js` from your ZIP. (Do not retype—just copy/paste.)

---

### 6.2. `apexcharts.css`

**Path:**

```
/StarTrendsDashboard/wwwroot/lib/apexcharts/apexcharts.css
```

> **Content:** If your ZIP contains `apexcharts.css`, copy that entire file here. Otherwise omit this file and remove the CSS link in `_Host.cshtml`.

---

### 6.3. `apexInterop.js`

**Path:**

```
/StarTrendsDashboard/wwwroot/lib/apexcharts/apexInterop.js
```

```js
window.apexInterop = {
  renderChart: function (elementId, config) {
    if (!elementId || !config) return;
    const elem = document.querySelector(`#${elementId}`);
    if (!elem) return;
    // If a chart already exists on that ID, destroy it first
    if (ApexCharts.getChartByID(elementId)) {
      ApexCharts.getChartByID(elementId).destroy();
    }
    const chart = new ApexCharts(elem, config);
    chart.render();
  },
  updateSeries: function (elementId, newSeries) {
    const chart = ApexCharts.getChartByID(elementId);
    if (chart) {
      chart.updateSeries(newSeries, true);
    }
  }
};
```

---

## 7. Blazor Component Wrapping ApexCharts

### 7.1. `RawApexChart.razor`

**Path:**

```
/StarTrendsDashboard/Components/RawApexChart.razor
```

```razor
@inject IJSRuntime JS

<div id="@_chartDivId" style="min-height: 350px;"></div>

@code {
    [Parameter] public string ChartId { get; set; } = string.Empty;
    [Parameter] public object Options { get; set; } = new { };
    [Parameter] public IEnumerable<object> Series { get; set; } = Array.Empty<object>();

    private string _chartDivId => ChartId;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // Convert Options to a dictionary if it’s an anonymous object
            var configDict = Options as IDictionary<string, object>
                             ?? new Dictionary<string, object>();

            if (!(Options is IDictionary<string, object>))
            {
                configDict = Options
                    .GetType()
                    .GetProperties()
                    .ToDictionary(prop => prop.Name, prop => prop.GetValue(Options, null)!);
            }

            configDict["series"] = Series;
            await JS.InvokeVoidAsync("apexInterop.renderChart", _chartDivId, configDict);
        }
    }

    public async Task RefreshAsync(IEnumerable<object> newSeries)
    {
        await JS.InvokeVoidAsync("apexInterop.updateSeries", _chartDivId, newSeries);
    }
}
```

---

## 8. Razor Pages

### 8.1. `_Host.cshtml`

**Path:**

```
/StarTrendsDashboard/Pages/_Host.cshtml
```

```html
@page "/"
@namespace StarTrendsDashboard.Pages
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@{
    Layout = null;
}

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>StarTrends Dashboard</title>
    <base href="~/" />
    <link href="css/bootstrap/bootstrap.min.css" rel="stylesheet" />
    <link href="css/site.css" rel="stylesheet" />

    <!-- ApexCharts CSS (if present) -->
    <link href="lib/apexcharts/apexcharts.css" rel="stylesheet" />
</head>
<body>
    <app>
        <component type="typeof(App)" render-mode="ServerPrerendered" />
    </app>

    <!-- Blazor script -->
    <script src="_framework/blazor.server.js"></script>

    <!-- ApexCharts JS -->
    <script src="lib/apexcharts/apexcharts.min.js"></script>
    <script src="lib/apexcharts/apexInterop.js"></script>
</body>
</html>
```

---

### 8.2. `Index.razor`

**Path:**

```
/StarTrendsDashboard/Pages/Index.razor
```

```razor
@page "/"

<h3>Welcome to StarTrends Dashboard</h3>
<p>Select “Product” or “Trade” from the menu to see charts.</p>
```

---

### 8.3. `ChartPage.razor`

**Path:**

```
/StarTrendsDashboard/Pages/ChartPage.razor
```

```razor
@page "/charts/{pageName}"
@inject IChartService ChartService
@using StarTrendsDashboard.Models
@using System.Collections.Generic

<h3>Charts for “@pageName”</h3>

@if (_definitions == null || !_definitions.Any())
{
    <p class="text-muted">No charts configured for “@pageName”.</p>
}
else
{
    @foreach (var def in _definitions)
    {
        <div class="mb-5">
            <h4>@def.Title</h4>
            <button class="btn btn-sm btn-outline-primary mb-2"
                    @onclick="() => RefreshChart(def.ChartId)">
                Refresh Now
            </button>

            @if (_cacheMap.TryGetValue(def.ChartId, out var cache) == false
                  || cache.Rows.Count == 0)
            {
                <p><em>Loading data…</em></p>
            }
            else
            {
                <RawApexChart @ref="_chartRefs[def.ChartId]"
                              ChartId="@def.ChartId"
                              Options="@GetOptions(def, cache)"
                              Series="@GetSeries(def, cache)" />
                <p class="text-muted">
                    Last updated: @cache.LastUpdatedUtc.ToLocalTime().ToString("g")
                </p>
            }
        </div>
    }
}

@code {
    [Parameter] public string pageName { get; set; } = string.Empty;

    private List<ChartDefinition> _definitions = new();
    private readonly Dictionary<string, ChartDataCache> _cacheMap
        = new(StringComparer.OrdinalIgnoreCase);
    private readonly Dictionary<string, RawApexChart> _chartRefs
        = new(StringComparer.OrdinalIgnoreCase);

    protected override async Task OnInitializedAsync()
    {
        _definitions = ChartService.GetDefinitionsByPage(pageName).ToList();

        foreach (var def in _definitions)
        {
            _chartRefs[def.ChartId] = null!; // placeholder; will be set by @ref

            var existing = ChartService.GetCachedData(def.ChartId);
            if (existing == null || existing.Rows.Count == 0)
            {
                var loaded = await ChartService.RefreshChartAsync(def.ChartId);
                _cacheMap[def.ChartId] = loaded;
            }
            else
            {
                _cacheMap[def.ChartId] = existing;
            }
        }
    }

    private object GetOptions(ChartDefinition def, ChartDataCache cache)
    {
        var rows = cache.Rows;
        if (def.ChartType.Equals("Bar", StringComparison.OrdinalIgnoreCase))
        {
            return new
            {
                chart = new
                {
                    type = "bar",
                    toolbar = new { show = true },
                    zoom = new { enabled = false }
                },
                xaxis = new
                {
                    categories = rows.Select(r => r.Label).ToArray()
                },
                title = new { text = def.Title, align = "left" }
            };
        }

        // Fallback to Bar if unknown type
        return new
        {
            chart = new
            {
                type = "bar",
                toolbar = new { show = true },
                zoom = new { enabled = false }
            },
            xaxis = new
            {
                categories = rows.Select(r => r.Label).ToArray()
            },
            title = new { text = def.Title, align = "left" }
        };
    }

    private IEnumerable<object> GetSeries(ChartDefinition def, ChartDataCache cache)
    {
        var rows = cache.Rows;
        if (def.ChartType.Equals("Bar", StringComparison.OrdinalIgnoreCase))
        {
            return new[]
            {
                new
                {
                    name = def.ChartId,
                    data = rows.Select(r => r.Value).ToArray()
                }
            };
        }

        // Fallback to Bar if unknown
        return new[]
        {
            new
            {
                name = def.ChartId,
                data = rows.Select(r => r.Value).ToArray()
            }
        };
    }

    private async Task RefreshChart(string chartId)
    {
        var updated = await ChartService.RefreshChartAsync(chartId);
        if (updated != null)
        {
            _cacheMap[chartId] = updated;
            var newSeries = GetSeries(
                _definitions.First(d => d.ChartId == chartId),
                updated);
            if (_chartRefs.TryGetValue(chartId, out var comp) && comp != null)
            {
                await comp.RefreshAsync(newSeries);
            }
        }
    }
}
```

---

## 9. Shared Navigation Menu

### 9.1. `NavMenu.razor`

**Path:**

```
/StarTrendsDashboard/Shared/NavMenu.razor
```

```razor
<nav class="navbar navbar-expand-lg navbar-light bg-light">
    <div class="container-fluid">
        <a class="navbar-brand" href="/">StarTrends</a>
        <button class="navbar-toggler" type="button" data-bs-toggle="collapse"
                data-bs-target="#navbarNav">
            <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="navbarNav">
            <ul class="navbar-nav">
                <li class="nav-item">
                    <NavLink class="nav-link" href="charts/Product">Product</NavLink>
                </li>
                <li class="nav-item">
                    <NavLink class="nav-link" href="charts/Trade">Trade</NavLink>
                </li>
            </ul>
        </div>
    </div>
</nav>
```

---

## 10. Summary of wwwroot Files

Inside `/StarTrendsDashboard/wwwroot/lib/apexcharts/`:

* **`apexcharts.min.js`** (copy exactly from your downloaded ZIP)
* **`apexcharts.css`** (if present in your ZIP, copy it; otherwise omit)
* **`apexInterop.js`** (content given above in step 6.3)

Nothing else in `wwwroot` needs editing.

---

## 11. Final Folder Tree (for reference)

```
StarTrendsDashboard/
│
├── appsettings.json
├── Program.cs
│
├── ChartDefinitions/
│   ├── chart‐definitions.json
│   └── Queries/
│       ├── ProductMarkets.sql
│       └── TradeTools.sql
│
├── Models/
│   ├── ChartDefinition.cs
│   ├── ChartDataRow.cs
│   └── ChartDataCache.cs
│
├── Services/
│   ├── ChartService.cs
│   └── ChartPollingService.cs
│
├── Components/
│   └── RawApexChart.razor
│
├── Pages/
│   ├── _Host.cshtml
│   ├── Index.razor
│   └── ChartPage.razor
│
├── Shared/
│   └── NavMenu.razor
│
└── wwwroot/
    └── lib/
        └── apexcharts/
            ├── apexcharts.min.js    ← copy from your ZIP
            ├── apexcharts.css       ← copy from your ZIP (if provided)
            └── apexInterop.js
```

---

### How to proceed

1. **Create these folders/files** exactly as above.
2. **Paste** the code for each file from this list (except `apexcharts.min.js`/`apexcharts.css`, which you copy from your ZIP).
3. **Adjust** your Oracle connection string in `appsettings.json`.
4. **Build & run**:

   ```bash
   cd StarTrendsDashboard
   dotnet restore
   dotnet build
   dotnet run
   ```
5. **Browse** to:

   * `https://localhost:5001/charts/Product` → “Markets Set in Last 30 Days” bar chart.
   * `https://localhost:5001/charts/Trade` → “Tools Used in Last 30 Days” bar chart.

All other pages (“/”, or unknown `/charts/XYZ` where no JSON entry has `"Page": "XYZ"`) will simply show the “No charts configured” message.

That completes the exact, final set of files you need.
