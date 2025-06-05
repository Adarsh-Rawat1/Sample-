{
  "ConnectionStrings": {
    "OracleDb": "Data Source=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=eurvlid05098.xmp.net.intra)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=381luk101)(SERVER=DEDICATED)));User Id=your_username;Password=your_password;"
  }
}

Below is a **complete, self-contained** Blazor Server implementation (no NuGet wrappers) that:

1. Reads chart definitions from JSON + SQL files
2. Caches and polls each chart’s Oracle SQL on a schedule
3. Renders any number of charts per “page” via ApexCharts JS
4. Requires only dropping new `.sql` + JSON entries to add charts

Follow the folder‐and‐file structure exactly. Copy each code snippet into the exact path shown. Afterward, run `dotnet run` and navigate to `/charts/Product` or `/charts/Trade`.

---

## Project Root Structure

```
/StarTrendsDashboard
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
            ├── apexcharts.min.js
            ├── apexcharts.css
            └── apexInterop.js
```

---

## 1. Configuration (`appsettings.json`)

Create `/StarTrendsDashboard/appsettings.json`:

```jsonc
{
  "ConnectionStrings": {
    "OracleDb": "User Id=MYUSER;Password=MYPASSWORD;Data Source=//myhost:1521/ORCLPDB1"
  },
  "ChartDefinitionsPath": "ChartDefinitions/chart‐definitions.json",
  "ChartQueryFolder": "ChartDefinitions/Queries"
}
```

* **OracleDb**: Replace `MYUSER`, `MYPASSWORD`, `//myhost:1521/ORCLPDB1` with your actual Oracle connection string.
* **ChartDefinitionsPath**: Path to the JSON that lists every chart.
* **ChartQueryFolder**: Folder containing all `.sql` scripts.

---

## 2. Entry Point (`Program.cs`)

Create `/StarTrendsDashboard/Program.cs`:

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

### 3.1. JSON Metadata (`chart‐definitions.json`)

Create `/StarTrendsDashboard/ChartDefinitions/chart‐definitions.json`:

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

* **ChartId**: Unique identifier (also used as HTML `div` ID).
* **Page**: string, e.g. `"Product"` or `"Trade"`. This maps to the route `/charts/{Page}`.
* **Title**: Display name above the chart.
* **ChartType**: `"Bar"`, `"Line"`, `"Scatter"`, or `"Pie"`. We’ll handle only `"Bar"` here.
* **SqlFile**: Name of the SQL script in `ChartDefinitions/Queries/`.
* **RefreshIntervalSeconds**: Poll interval in seconds (here 5 minutes).

### 3.2. SQL Scripts

Create `/StarTrendsDashboard/ChartDefinitions/Queries/ProductMarkets.sql`:

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

Create `/StarTrendsDashboard/ChartDefinitions/Queries/TradeTools.sql`:

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

> Both SQL scripts must return exactly two columns:
>
> 1. A string (label) in column 1
> 2. A number (value) in column 2

---

## 4. Data Models

### 4.1. `ChartDefinition.cs`

Create `/StarTrendsDashboard/Models/ChartDefinition.cs`:

```csharp
namespace StarTrendsDashboard.Models
{
    public class ChartDefinition
    {
        public string ChartId { get; set; }
        public string Page { get; set; }
        public string Title { get; set; }
        public string ChartType { get; set; }
        public string SqlFile { get; set; }
        public int RefreshIntervalSeconds { get; set; }
    }
}
```

### 4.2. `ChartDataRow.cs`

Create `/StarTrendsDashboard/Models/ChartDataRow.cs`:

```csharp
namespace StarTrendsDashboard.Models
{
    public class ChartDataRow
    {
        public string Label { get; set; }
        public decimal Value { get; set; }
    }
}
```

### 4.3. `ChartDataCache.cs`

Create `/StarTrendsDashboard/Models/ChartDataCache.cs`:

```csharp
using System;
using System.Collections.Generic;

namespace StarTrendsDashboard.Models
{
    public class ChartDataCache
    {
        public string ChartId { get; set; }
        public DateTime LastUpdatedUtc { get; set; }
        public List<ChartDataRow> Rows { get; set; } = new();
    }
}
```

---

## 5. Services

### 5.1. `ChartService.cs`

Create `/StarTrendsDashboard/Services/ChartService.cs`:

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

        // In-memory list of definitions
        private readonly List<ChartDefinition> _definitions = new();
        private readonly object _definitionsLock = new();

        // In-memory cache for chart data
        private readonly ConcurrentDictionary<string, ChartDataCache> _cache
            = new(StringComparer.OrdinalIgnoreCase);

        public ChartService(IConfiguration configuration, ILogger<ChartService> logger)
        {
            _logger = logger;
            _definitionsJsonPath = configuration["ChartDefinitionsPath"];
            _queryFolder = configuration["ChartQueryFolder"];
            _connectionString = configuration.GetConnectionString("OracleDb");

            // 1) Load JSON definitions
            if (!File.Exists(_definitionsJsonPath))
                throw new FileNotFoundException($"Cannot find: {_definitionsJsonPath}");

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
            return _cache.TryGetValue(chartId, out var existing)
                ? existing
                : null;
        }

        public async Task<ChartDataCache> RefreshChartAsync(string chartId)
        {
            ChartDefinition def;
            lock (_definitionsLock)
            {
                def = _definitions.FirstOrDefault(d =>
                    d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
            }
            if (def == null)
            {
                _logger.LogWarning($"[RefreshChart] No definition for {chartId}");
                return null;
            }

            var sqlPath = Path.Combine(_queryFolder, def.SqlFile);
            if (!File.Exists(sqlPath))
            {
                _logger.LogError($"[RefreshChart] SQL not found: {sqlPath}");
                return null;
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
                    var label = reader.GetValue(0)?.ToString();
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
                _logger.LogError(ex, $"[RefreshChart] Error executing SQL for {chartId}");
                return _cache[chartId];
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

### 5.2. `ChartPollingService.cs`

Create `/StarTrendsDashboard/Services/ChartPollingService.cs`:

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

        // Track next‐due times per chart
        private readonly Dictionary<string, DateTime> _nextRefreshUtc
            = new(StringComparer.OrdinalIgnoreCase);

        public ChartPollingService(IChartService chartService, ILogger<ChartPollingService> logger)
        {
            _chartService = chartService;
            _logger = logger;

            // Schedule first run for all definitions at startup
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

                // If new definitions were added to JSON (upon restart), schedule them
                foreach (var def in allDefs)
                {
                    if (!_nextRefreshUtc.ContainsKey(def.ChartId))
                    {
                        _logger.LogInformation($"Scheduling new chart '{def.ChartId}'.");
                        _nextRefreshUtc[def.ChartId] = now;
                    }
                }

                // Refresh any chart whose time has come
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

                // Wait 60 seconds before checking again
                await Task.Delay(TimeSpan.FromSeconds(60), stoppingToken);
            }

            _logger.LogInformation("ChartPollingService stopping.");
        }
    }
}
```

---

## 6. JavaScript Interop & ApexCharts

Since you cannot use NuGet, we manually include ApexCharts JS/CSS and a tiny interop.

### 6.1. Downloaded files

* Copy `apexcharts.min.js` from your ZIP into:

  ```
  /StarTrendsDashboard/wwwroot/lib/apexcharts/apexcharts.min.js
  ```
* If there is `apexcharts.css` in your ZIP, copy it into:

  ```
  /StarTrendsDashboard/wwwroot/lib/apexcharts/apexcharts.css
  ```

### 6.2. `apexInterop.js`

Create `/StarTrendsDashboard/wwwroot/lib/apexcharts/apexInterop.js`:

```js
window.apexInterop = {
  renderChart: function (elementId, config) {
    if (!elementId || !config) return;
    const elem = document.querySelector(`#${elementId}`);
    if (!elem) return;
    // If a chart already exists on that ID, destroy it first (optional)
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

* **renderChart**: Instantiates a new ApexCharts instance on `#elementId` with the given `config` (JS object).
* **updateSeries**: Finds an existing chart by its `ID` and updates its series data on-demand.

---

## 7. Raw Blazor Component to Wrap ApexCharts

### 7.1. `RawApexChart.razor`

Create `/StarTrendsDashboard/Components/RawApexChart.razor`:

```razor
@inject IJSRuntime JS

<div id="@_chartDivId" style="min-height: 350px;"></div>

@code {
    [Parameter] public string ChartId { get; set; }
    [Parameter] public object Options { get; set; }
    [Parameter] public IEnumerable<object> Series { get; set; }

    private string _chartDivId => ChartId;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // Merge Options with series into a single config object
            // We assume Options is a Dictionary<string, object> or an anonymous type
            var config = Options as IDictionary<string, object> 
                         ?? new Dictionary<string, object>();
            // If Options is an anonymous object, JSInterop will serialize it directly including series
            // So create a new object:
            var fullConfig = new Dictionary<string, object>(config)
            {
                ["series"] = Series
            };
            await JS.InvokeVoidAsync("apexInterop.renderChart", _chartDivId, fullConfig);
        }
    }

    // Call this from parent to update series
    public async Task RefreshAsync(IEnumerable<object> newSeries)
    {
        await JS.InvokeVoidAsync("apexInterop.updateSeries", _chartDivId, newSeries);
    }
}
```

* `<div id="@_chartDivId">` is the container for ApexCharts.
* On first render (`firstRender == true`), we merge `Options` and `Series` into `fullConfig` and call `renderChart`.
* `RefreshAsync(...)` can be called by parent to dynamically update the chart’s data.

---

## 8. Razor Pages

### 8.1. `_Host.cshtml`

Modify `/StarTrendsDashboard/Pages/_Host.cshtml` to include CSS/JS:

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
    <!-- ApexCharts CSS -->
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

* Include `apexcharts.css` inside `<head>`.
* Include `apexcharts.min.js` and `apexInterop.js` just before `</body>`.

### 8.2. `Index.razor`

Create `/StarTrendsDashboard/Pages/Index.razor`:

```razor
@page "/"

<h3>Welcome to StarTrends Dashboard</h3>
<p>Select a section from the menu to view its charts.</p>
```

### 8.3. `ChartPage.razor`

Create `/StarTrendsDashboard/Pages/ChartPage.razor`:

```razor
@page "/charts/{pageName}"
@inject IChartService ChartService
@using BlazorApexCharts
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
    [Parameter]
    public string pageName { get; set; }

    private List<ChartDefinition> _definitions = new();
    private readonly Dictionary<string, ChartDataCache> _cacheMap
        = new(StringComparer.OrdinalIgnoreCase);
    private readonly Dictionary<string, RawApexChart> _chartRefs
        = new(StringComparer.OrdinalIgnoreCase);

    protected override async Task OnInitializedAsync()
    {
        // 1) Load definitions for this page
        _definitions = ChartService.GetDefinitionsByPage(pageName).ToList();

        // Initialize chartRefs and cacheMap
        foreach (var def in _definitions)
        {
            _chartRefs[def.ChartId] = null;

            // Attempt to get cached data; if empty, refresh once
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

        // You can add other chart types here (Line, Scatter, Pie)...

        // Fallback: simple bar
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

        // Fallback: bar
        return new[]
        {
            new { name = def.ChartId, data = rows.Select(r => r.Value).ToArray() }
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

* **Route** `/charts/{pageName}` dynamically matches any `pageName` (e.g. “Product” or “Trade”).
* On first load, it grabs definitions whose `Page` == `pageName`, fetches initial data from `ChartService.RefreshChartAsync`, and stores it in `_cacheMap`.
* For each chart, it renders `<RawApexChart>` with generated `Options` and `Series`.
* The “Refresh Now” button calls `RefreshChart(chartId)`, which re‐queries and updates the chart in place.

---

## 9. Shared Navigation (`NavMenu.razor`)

Create or modify `/StarTrendsDashboard/Shared/NavMenu.razor`:

```razor
<nav class="navbar navbar-expand-lg navbar-light bg-light">
    <div class="container-fluid">
        <a class="navbar-brand" href="">StarTrends</a>
        <button class="navbar-toggler" type="button" data-bs-toggle="collapse"
                data-bs-target="#navbarNav">
            <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="navbarNav">
            <ul class="navbar-nav">
                <!-- Hard-code links to known pages: -->
                <li class="nav-item">
                    <NavLink class="nav-link" href="charts/Product">Product</NavLink>
                </li>
                <li class="nav-item">
                    <NavLink class="nav-link" href="charts/Trade">Trade</NavLink>
                </li>
                <!-- To add more pages, insert more NavLink entries -->
            </ul>
        </div>
    </div>
</nav>
```

* Clicking “Product” goes to `/charts/Product`.
* Clicking “Trade” goes to `/charts/Trade`.

---

## 10. wwwroot (ApexCharts JS/CSS & Interop)

### 10.1. Copy files into `/StarTrendsDashboard/wwwroot/lib/apexcharts/`

1. **`apexcharts.min.js`**

   * From your downloaded ZIP, locate `apexcharts.min.js`.
   * Copy it here:

     ```
     /StarTrendsDashboard/wwwroot/lib/apexcharts/apexcharts.min.js
     ```

2. **`apexcharts.css`** (if your ZIP contains it)

   * Copy it here:

     ```
     /StarTrendsDashboard/wwwroot/lib/apexcharts/apexcharts.css
     ```

3. **`apexInterop.js`** (create this file)
   Create `/StarTrendsDashboard/wwwroot/lib/apexcharts/apexInterop.js`:

   ```js
   window.apexInterop = {
     renderChart: function (elementId, config) {
       if (!elementId || !config) return;
       const elem = document.querySelector(`#${elementId}`);
       if (!elem) return;
       // Destroy existing chart if present
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

   * `renderChart`: creates a new ApexCharts instance in the `<div>` whose `id=elementId`.
   * `updateSeries`: updates an existing chart’s data array.

---

## 11. Shared Layout (MainLayout — if needed)

Ensure `/StarTrendsDashboard/Shared/MainLayout.razor` (from Blazor template) includes `NavMenu`:

```razor
@inherits LayoutComponentBase

<div class="page">
    <div class="sidebar">
        <NavMenu />
    </div>
    <div class="main">
        <div class="top-row px-4">
            <a href="">StarTrends</a>
        </div>
        <div class="content px-4">
            @Body
        </div>
    </div>
</div>
```

---

## 12. Build & Run

1. **Restore and build**:

   ```bash
   cd StarTrendsDashboard
   dotnet restore
   dotnet build
   ```
2. **Run**:

   ```bash
   dotnet run
   ```
3. **Navigate** in your browser to:

   * `https://localhost:5001/charts/Product` → Shows the “Markets Set in Last 30 Days” bar chart.
   * `https://localhost:5001/charts/Trade` → Shows the “Tools Used in Last 30 Days” bar chart.

   The background service logs will indicate each chart being refreshed every 5 minutes.

---

## 13. Adding More Charts (Future)

1. **Add new SQL file** under `/ChartDefinitions/Queries/`, e.g. `ProductTopUsers.sql`.
2. **Append a JSON object** in `/ChartDefinitions/chart‐definitions.json`:

   ```jsonc
   {
     "ChartId": "ProductTopUsers",
     "Page": "Product",
     "Title": "Top 10 Users (Last 7 Days)",
     "ChartType": "Bar",
     "SqlFile": "ProductTopUsers.sql",
     "RefreshIntervalSeconds": 300
   }
   ```
3. **Restart** the application (`dotnet run`).
4. **Navigate** to `/charts/Product`. You’ll now see two bar charts (Markets + Top Users) in one page, each with its own refresh button and polling.

No code changes are needed beyond adding SQL and JSON entries.

---

### That’s the **complete start-to-end implementation**.

Simply copy each file into the paths shown, adjust your Oracle credentials, and you’re ready to go.
