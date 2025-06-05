Below is a complete, end‐to‐end implementation that splits your dashboard into:

1. **MyDashboard.Shared** – a class‐library with shared models.
2. **MyDashboard.Worker** – a .NET Worker Service that polls Oracle on a schedule and writes cached JSON files.
3. **MyDashboard.Web** – an ASP .NET Core Web App (Razor Pages) that reads those JSON files and renders ApexCharts in the browser, with a “Refresh Now” button that forces the Worker logic to run on demand.

Each section shows exactly **where to place** every file (folder and filename) and the **full file contents**. When you’ve copied everything, build and run:

* **Step 1**: Start the Worker (`MyDashboard.Worker`). It will immediately write `ChartCache/ProductMarkets.json` and `ChartCache/TradeTools.json`.
* **Step 2**: Start the Web App (`MyDashboard.Web`). Navigate to `/charts/Product` or `/charts/Trade`. Clicking “Refresh Now” will POST to `/api/refresh/{chartId}`, trigger an immediate SQL run, rewrite the JSON, and redraw the chart.

---

## 1. Solution Layout

Create a new directory for your solution, e.g. `MyDashboardSolution/`. Inside it, create three projects:

```
MyDashboardSolution/
├── MyDashboard.Shared/       (Class Library for shared models)
├── MyDashboard.Worker/       (Worker Service for polling + cache)
└── MyDashboard.Web/          (Web App for rendering charts)
```

You can open this folder in Visual Studio or VS Code as a single solution. Each subfolder is its own project.

---

## 2. Shared Library: MyDashboard.Shared

This library holds the POCOs that both Worker and Web projects will use.

### 2.1. Create the project

In your solution root:

```bash
cd MyDashboardSolution
dotnet new classlib -o MyDashboard.Shared
```

You’ll get `MyDashboard.Shared/MyDashboard.Shared.csproj` and an empty `Class1.cs`. Delete `Class1.cs` and replace with the following three files:

### 2.2. File: `MyDashboard.Shared/ChartDefinition.cs`

```csharp
namespace MyDashboard.Shared
{
    public class ChartDefinition
    {
        public string ChartId { get; set; } = string.Empty;
        public string Page { get; set; } = string.Empty;           // e.g. "Product" or "Trade"
        public string Title { get; set; } = string.Empty;          // e.g. "Markets Set in Last 30 Days"
        public string ChartType { get; set; } = string.Empty;      // "Bar", "Line", "Scatter", etc.
        public string SqlFile { get; set; } = string.Empty;        // e.g. "ProductMarkets.sql"
        public int RefreshIntervalSeconds { get; set; } = 300;
    }
}
```

### 2.3. File: `MyDashboard.Shared/ChartDataRow.cs`

```csharp
namespace MyDashboard.Shared
{
    public class ChartDataRow
    {
        public string Label { get; set; } = string.Empty;   // e.g. "US", "EU"
        public decimal Value { get; set; }                  // e.g. 1234
    }
}
```

### 2.4. File: `MyDashboard.Shared/ChartDataCache.cs`

```csharp
using System;
using System.Collections.Generic;

namespace MyDashboard.Shared
{
    public class ChartDataCache
    {
        public string ChartId { get; set; } = string.Empty;
        public DateTime LastUpdatedUtc { get; set; }
        public List<ChartDataRow> Rows { get; set; } = new List<ChartDataRow>();
    }
}
```

You are done with the Shared project. The next two projects will reference this via a project reference.

---

## 3. Worker Service: MyDashboard.Worker

This project runs as a service or console, polls Oracle every *N* seconds, and writes JSON cache files under `ChartCache/`.

### 3.1. Create the Worker project

At solution root:

```bash
dotnet new worker -o MyDashboard.Worker
```

You now have `MyDashboard.Worker/Program.cs`, `MyDashboard.Worker/Worker.cs`, etc. Delete the default `Worker.cs` (we’ll replace it). Also delete `Worker.cs` from the folder.

Add a reference to Shared:

```bash
cd MyDashboard.Worker
dotnet add reference ../MyDashboard.Shared/MyDashboard.Shared.csproj
```

### 3.2. Folder structure in `MyDashboard.Worker`

```
MyDashboard.Worker/
├── appsettings.json
├── Program.cs
├── ChartDefinitions/
│   ├── chart-definitions.json
│   └── Queries/
│       ├── ProductMarkets.sql
│       └── TradeTools.sql
├── ChartCache/               ← Worker will write JSON files here
├── Services/
│   ├── IChartService.cs
│   ├── ChartService.cs
│   └── ChartPollingBackgroundService.cs
└── MyDashboard.Worker.csproj
```

Create the folders `ChartDefinitions/`, under it `Queries/`, and `ChartCache/`, and also a `Services/` folder. Then add these files:

---

### 3.3. File: `MyDashboard.Worker/appsettings.json`

```jsonc
{
  "ConnectionStrings": {
    "OracleDb": "User Id=MYUSER;Password=MYPASSWORD;Data Source=//myhost:1521/ORCLPDB"
  },
  "ChartDefinitionsPath": "ChartDefinitions/chart-definitions.json",
  "ChartQueryFolder": "ChartDefinitions/Queries",
  "ChartCacheFolder": "ChartCache"
}
```

* **OracleDb**: replace `MYUSER`, `MYPASSWORD`, `//myhost:1521/ORCLPDB` with your actual connect string.
* `ChartDefinitionsPath`: relative path within this project to the JSON definitions.
* `ChartQueryFolder`: folder containing your `.sql` scripts.
* `ChartCacheFolder`: folder where JSON cache files will be written.

In Visual Studio, right‐click each of these paths/folders and set “Copy to Output Directory” → **Copy if newer** (for the SQL and the JSON). For `ChartCache/`, set “Copy Always” so the folder exists next to the built `.exe`.

---

### 3.4. File: `MyDashboard.Worker/ChartDefinitions/chart-definitions.json`

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
    "Title": "Trades Saved in Last 30 Days",
    "ChartType": "Scatter",
    "SqlFile": "TradeTools.sql",
    "RefreshIntervalSeconds": 300
  }
]
```

* **ChartId**: used as both filename for JSON (`ChartCache/ProductMarkets.json`) and as a unique key.
* **Page**: not used by Worker, but for consistency if you ever want to reload definitions.
* **Title**, **ChartType**: not used by Worker; just here for completeness.
* **SqlFile**: name of the SQL script under `Queries/`.
* **RefreshIntervalSeconds**: polling interval; 300 s = 5 minutes.

---

### 3.5. File: `MyDashboard.Worker/ChartDefinitions/Queries/ProductMarkets.sql`

```sql
SELECT
  r.feature AS "Markets",
  COUNT(1)  AS "Times used"
FROM star_action_audit r
WHERE r.mod_dt > TRUNC(SYSDATE) - 30
  AND feature_type = 'MARKET'
GROUP BY r.feature
ORDER BY "Times used" DESC
```

– No trailing semicolon; only the raw `SELECT …`.

---

### 3.6. File: `MyDashboard.Worker/ChartDefinitions/Queries/TradeTools.sql`

```sql
SELECT
  r.feature AS "Tools",
  COUNT(1)  AS "Times used"
FROM star_action_audit r
WHERE r.mod_dt > TRUNC(SYSDATE) - 30
  AND feature_type = 'TOOLS'
GROUP BY r.feature
ORDER BY "Times used" DESC
```

– Again, no semicolon.

---

### 3.7. File: `MyDashboard.Worker/Services/IChartService.cs`

```csharp
using MyDashboard.Shared;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace MyDashboard.Worker.Services
{
    public interface IChartService
    {
        /// <summary>
        /// Returns all chart definitions from chart-definitions.json.
        /// </summary>
        IReadOnlyList<ChartDefinition> GetAllDefinitions();

        /// <summary>
        /// Runs the SQL for chartId, builds ChartDataCache,
        /// writes JSON to ChartCache/{chartId}.json, and returns the cache.
        /// </summary>
        Task<ChartDataCache> RefreshChartAsync(string chartId);
    }
}
```

---

### 3.8. File: `MyDashboard.Worker/Services/ChartService.cs`

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using MyDashboard.Shared;
using Oracle.ManagedDataAccess.Client;
using System;
using System.Collections.Generic;
using System.Data;
using System.IO;
using System.Linq;
using System.Text.Json;
using System.Threading.Tasks;

namespace MyDashboard.Worker.Services
{
    public class ChartService : IChartService
    {
        private readonly string _definitionsJsonPath;
        private readonly string _queryFolder;
        private readonly string _connectionString;
        private readonly string _cacheFolder;
        private readonly ILogger<ChartService> _logger;

        // In-memory list of definitions & last write timestamp
        private readonly List<ChartDefinition> _definitions = new();
        private DateTime _defsLastWriteTimeUtc;
        private readonly object _lock = new();

        public ChartService(IConfiguration config, ILogger<ChartService> logger)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));

            _definitionsJsonPath = config["ChartDefinitionsPath"]
                ?? throw new ArgumentNullException("ChartDefinitionsPath missing");
            _queryFolder = config["ChartQueryFolder"]
                ?? throw new ArgumentNullException("ChartQueryFolder missing");
            _cacheFolder = config["ChartCacheFolder"]
                ?? throw new ArgumentNullException("ChartCacheFolder missing");
            _connectionString = config.GetConnectionString("OracleDb")
                ?? throw new ArgumentNullException("OracleDb connection string missing");

            // Ensure cache folder exists
            Directory.CreateDirectory(_cacheFolder);

            // Load definitions initially
            LoadDefinitions();
        }

        private void LoadDefinitions()
        {
            var fi = new FileInfo(_definitionsJsonPath);
            if (!fi.Exists)
                throw new FileNotFoundException($"Cannot find {_definitionsJsonPath}");

            if (fi.LastWriteTimeUtc <= _defsLastWriteTimeUtc && _definitions.Count > 0)
                return; // no changes

            var json = File.ReadAllText(_definitionsJsonPath);
            var defs = JsonSerializer.Deserialize<List<ChartDefinition>>(json)
                        ?? new List<ChartDefinition>();

            lock (_lock)
            {
                _definitions.Clear();
                _definitions.AddRange(defs);
                _defsLastWriteTimeUtc = fi.LastWriteTimeUtc;
            }

            _logger.LogInformation("Loaded {Count} chart definitions", _definitions.Count);
        }

        public IReadOnlyList<ChartDefinition> GetAllDefinitions()
        {
            LoadDefinitions();
            lock (_lock)
            {
                // return a copy to protect our list
                return _definitions.ToList();
            }
        }

        public async Task<ChartDataCache> RefreshChartAsync(string chartId)
        {
            LoadDefinitions();

            ChartDefinition? def;
            lock (_lock)
            {
                def = _definitions.FirstOrDefault(d => 
                    d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
            }
            if (def == null)
            {
                _logger.LogWarning("RefreshChartAsync: no definition for '{chartId}'", chartId);
                return new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };
            }

            var sqlPath = Path.Combine(_queryFolder, def.SqlFile);
            if (!File.Exists(sqlPath))
            {
                _logger.LogError("SQL file not found: {sqlPath}", sqlPath);
                return new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };
            }

            string rawSql = await File.ReadAllTextAsync(sqlPath);
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
                    string label = reader.IsDBNull(0)
                        ? string.Empty
                        : reader.GetString(0);
                    object valObj = reader.GetValue(1);
                    decimal value = valObj == DBNull.Value
                        ? 0
                        : Convert.ToDecimal(valObj);

                    rows.Add(new ChartDataRow
                    {
                        Label = label,
                        Value = value
                    });
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error executing SQL for '{chartId}'", chartId);
                // Return partial or empty rows
                return new ChartDataCache
                {
                    ChartId = chartId,
                    LastUpdatedUtc = DateTime.UtcNow,
                    Rows = rows
                };
            }

            var cache = new ChartDataCache
            {
                ChartId = chartId,
                LastUpdatedUtc = DateTime.UtcNow,
                Rows = rows
            };

            // Serialize to JSON file
            var outPath = Path.Combine(_cacheFolder, $"{chartId}.json");
            var jsonOut = JsonSerializer.Serialize(cache, new JsonSerializerOptions
            {
                WriteIndented = true
            });
            await File.WriteAllTextAsync(outPath, jsonOut);

            _logger.LogInformation("Refreshed '{chartId}' → {Count} rows, wrote {outPath}",
                chartId, rows.Count, outPath);

            return cache;
        }
    }
}
```

* **LoadDefinitions()** re‐reads `chart-definitions.json` only if it’s changed on disk.
* **RefreshChartAsync** runs the SQL, builds a `ChartDataCache`, writes it to `ChartCache/{chartId}.json`, and returns it.

---

### 3.9. File: `MyDashboard.Worker/Services/ChartPollingBackgroundService.cs`

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using MyDashboard.Shared;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

namespace MyDashboard.Worker.Services
{
    public class ChartPollingBackgroundService : BackgroundService
    {
        private readonly IChartService _chartService;
        private readonly ILogger<ChartPollingBackgroundService> _logger;

        // Keeps track of the next run time for each chart
        private readonly Dictionary<string, DateTime> _nextRunUtc
            = new(StringComparer.OrdinalIgnoreCase);

        public ChartPollingBackgroundService(IChartService chartService, ILogger<ChartPollingBackgroundService> logger)
        {
            _chartService = chartService;
            _logger = logger;

            // At startup, schedule an immediate run for all existing definitions
            var defs = _chartService.GetAllDefinitions();
            foreach (var d in defs)
            {
                _nextRunUtc[d.ChartId] = DateTime.UtcNow;
            }
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation("ChartPollingBackgroundService started.");

            while (!stoppingToken.IsCancellationRequested)
            {
                var now = DateTime.UtcNow;
                var defs = _chartService.GetAllDefinitions();

                // If new definitions appeared on disk, schedule them immediately
                foreach (var d in defs)
                {
                    if (!_nextRunUtc.ContainsKey(d.ChartId))
                    {
                        _logger.LogInformation("Scheduling new chart '{chartId}' for immediate run", d.ChartId);
                        _nextRunUtc[d.ChartId] = now;
                    }
                }

                // For each chart, check if it's time to refresh
                foreach (var def in defs)
                {
                    if (_nextRunUtc.TryGetValue(def.ChartId, out var nextTime) && now >= nextTime)
                    {
                        _logger.LogInformation("Background refreshing '{chartId}'", def.ChartId);
                        try
                        {
                            await _chartService.RefreshChartAsync(def.ChartId);
                        }
                        catch (Exception ex)
                        {
                            _logger.LogError(ex, "Error in background refresh for '{chartId}'", def.ChartId);
                        }
                        // Schedule next run = now + interval
                        _nextRunUtc[def.ChartId] = now.AddSeconds(def.RefreshIntervalSeconds);
                    }
                }

                // Sleep for a short period before re-checking (e.g. 30 seconds)
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }

            _logger.LogInformation("ChartPollingBackgroundService stopping.");
        }
    }
}
```

* Every 30 seconds, it loops through definitions and calls `RefreshChartAsync` on any chart whose scheduled time has arrived.

---

### 3.10. File: `MyDashboard.Worker/Program.cs`

Replace the default with:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using MyDashboard.Worker.Services;

IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        // Register our ChartService and background polling
        services.AddSingleton<IChartService, ChartService>();
        services.AddHostedService<ChartPollingBackgroundService>();
    })
    .Build();

await host.RunAsync();
```

* This wires up DI so `ChartService` is a singleton; the `ChartPollingBackgroundService` starts automatically.

---

#### 3.11. Ensure files “Copy to Output Directory”

In your project’s file‐properties (or directly in the `.csproj`), make sure:

```xml
<ItemGroup>
  <None Update="ChartDefinitions\chart-definitions.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
  <None Update="ChartDefinitions\Queries\*.sql">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
  <Folder Include="ChartCache\">
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </Folder>
</ItemGroup>
```

(That snippet goes inside `<Project>…</Project>` in `MyDashboard.Worker.csproj`.) This ensures the JSON, SQL, and cache‐folder exist next to the Worker executable.

---

### 3.12. Build and run the Worker

From the `MyDashboard.Worker` folder:

```bash
dotnet build
dotnet run
```

You should see logs like:

```
info: MyDashboard.Worker.Services.ChartService[0]
      Loaded 2 chart definitions
info: MyDashboard.Worker.Services.ChartPollingBackgroundService[0]
      ChartPollingBackgroundService started.
info: MyDashboard.Worker.Services.ChartPollingBackgroundService[0]
      Background refreshing 'ProductMarkets'
info: MyDashboard.Worker.Services.ChartService[0]
      Refreshed 'ProductMarkets' → 5 rows, wrote ChartCache/ProductMarkets.json
info: MyDashboard.Worker.Services.ChartPollingBackgroundService[0]
      Background refreshing 'TradeTools'
info: MyDashboard.Worker.Services.ChartService[0]
      Refreshed 'TradeTools' → 8 rows, wrote ChartCache/TradeTools.json
…
```

A `ChartCache/` folder should now contain `ProductMarkets.json` and `TradeTools.json`. Each JSON looks like:

```jsonc
{
  "ChartId": "ProductMarkets",
  "LastUpdatedUtc": "2025-06-06T08:30:00Z",
  "Rows": [
    { "Label": "US", "Value": 123 },
    { "Label": "EU", "Value": 98 },
    …
  ]
}
```

That completes the Worker side.

---

## 4. Web App: MyDashboard.Web

This project hosts the front‐end. It will read `ChartCache/{chartId}.json` (written by the Worker) and render ApexCharts in the browser. It also exposes an API endpoint to force‐refresh a chart on demand.

### 4.1. Create the Web project

At solution root:

```bash
dotnet new webapp -o MyDashboard.Web
```

You now have a Razor Pages‐based ASP .NET Core Web App in `MyDashboard.Web`. Delete the sample `Pages/Privacy.cshtml` and `Pages/Error.cshtml` if you want (optional).

Add references:

```bash
cd MyDashboard.Web
dotnet add reference ../MyDashboard.Shared/MyDashboard.Shared.csproj
dotnet add reference ../MyDashboard.Worker/MyDashboard.Worker.csproj
```

This ensures the Web App can use `IChartService` and the models.

### 4.2. Folder structure in `MyDashboard.Web`

```
MyDashboard.Web/
├── appsettings.json
├── Program.cs
├── Controllers/
│   └── RefreshController.cs
├── Pages/
│   ├── Index.cshtml
│   ├── Index.cshtml.cs
│   ├── Product.cshtml
│   ├── Product.cshtml.cs
│   ├── Trade.cshtml
│   └── Trade.cshtml.cs
├── wwwroot/
│   ├── css/
│   │   └── site.css
│   └── lib/
│       └── apexcharts/
│           ├── apexcharts.min.js
│           ├── apexcharts.css
│           └── apexInterop.js
└── MyDashboard.Web.csproj
```

Create the folders `Controllers/`, `Pages/`, and `wwwroot/lib/apexcharts/`. Copy the three ApexCharts files into `wwwroot/lib/apexcharts/`.

---

### 4.3. File: `MyDashboard.Web/appsettings.json`

```jsonc
{
  "CacheFolder": "ChartCache"
}
```

* This folder path (relative to the Web App’s content root) should point to the same `ChartCache/` that the Worker uses. When you deploy, you’ll place `ChartCache/` at the same level as the Web App’s published files so it can read and write there.

---

### 4.4. File: `MyDashboard.Web/Controllers/RefreshController.cs`

```csharp
using Microsoft.AspNetCore.Mvc;
using MyDashboard.Shared;
using System.Threading.Tasks;

namespace MyDashboard.Web.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class RefreshController : ControllerBase
    {
        private readonly IChartService _chartService;

        public RefreshController(IChartService chartService)
        {
            _chartService = chartService;
        }

        // POST /api/refresh/ProductMarkets
        [HttpPost("{chartId}")]
        public async Task<IActionResult> Post(string chartId)
        {
            // Force an immediate SQL run and JSON rewrite
            await _chartService.RefreshChartAsync(chartId);
            return Ok(new { chartId, refreshed = true });
        }
    }
}
```

* This lets the browser call `POST /api/refresh/ProductMarkets` to force‐refresh that chart.

---

### 4.5. File: `MyDashboard.Web/Program.cs`

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using MyDashboard.Worker.Services; // needs to see ChartService, IChartService

var builder = WebApplication.CreateBuilder(args);

// 1) Register ChartService so Web App can call RefreshChartAsync
builder.Services.AddSingleton<IChartService, ChartService>();

// 2) Add controllers (for RefreshController)
builder.Services.AddControllers();

// 3) Add Razor Pages
builder.Services.AddRazorPages();

var app = builder.Build();

// Serve static files (wwwroot)
app.UseStaticFiles();

// Map controllers (RefreshController)
app.MapControllers();

// Map Razor Pages
app.MapRazorPages();

app.Run();
```

* Note: We reference `ChartService` from the Worker project directly. You could refactor it into a separate library instead, but referencing `MyDashboard.Worker.csproj` works too.

---

### 4.6. Copy ApexCharts into `wwwroot/lib/apexcharts/`

From the ZIP you downloaded:

1. Copy **`apexcharts.min.js`** → `MyDashboard.Web/wwwroot/lib/apexcharts/apexcharts.min.js`
2. Copy **`apexcharts.css`** → `MyDashboard.Web/wwwroot/lib/apexcharts/apexcharts.css`
3. Create **`apexInterop.js`** with contents below and save as `MyDashboard.Web/wwwroot/lib/apexcharts/apexInterop.js`.

#### 4.6.1. File: `MyDashboard.Web/wwwroot/lib/apexcharts/apexInterop.js`

```js
window.apexInterop = {
  renderChart: function (elementId, config) {
    console.log(`[apexInterop] renderChart for '#${elementId}', config=`, config);
    const elem = document.querySelector(`#${elementId}`);
    if (!elem) {
      console.warn(`[apexInterop] Container '#${elementId}' not found.`);
      return;
    }
    // Destroy existing chart if present (optional)
    if (ApexCharts.getChartByID(elementId)) {
      ApexCharts.getChartByID(elementId).destroy();
    }
    const chart = new ApexCharts(elem, config);
    chart.render();
  },
  updateSeries: function (elementId, newSeries) {
    console.log(`[apexInterop] updateSeries for '#${elementId}', newSeries=`, newSeries);
    const chart = ApexCharts.getChartByID(elementId);
    if (chart) {
      chart.updateSeries(newSeries, true);
    } else {
      console.warn(`[apexInterop] No chart instance found for '#${elementId}'.`);
    }
  }
};
```

* This interop exposes `apexInterop.renderChart(divId, config)` and `apexInterop.updateSeries(divId, newSeries)` to your Blazor or Razor‐Page JS.

---

### 4.7. File: `MyDashboard.Web/wwwroot/css/site.css`

You can leave it empty or put minimal styling. For example:

```css
body {
  font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
  margin: 20px;
}
```

---

### 4.8. Razor Pages

We’ll create two pages:

* `/charts/Product` → shows the “Markets Set in Last 30 Days” bar chart.
* `/charts/Trade`   → shows the “Trades Saved in Last 30 Days” scatter chart.

#### 4.8.1. File: `MyDashboard.Web/Pages/Product.cshtml`

```html
@page "/charts/Product"
@model MyDashboard.Web.Pages.ProductModel
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers

@{
    ViewData["Title"] = "Product Charts";
}

<h3>Charts for “Product”</h3>

<h4>@Model.DefinitionTitle</h4>
<button class="btn btn-primary" id="refreshBtn">Refresh Now</button>

<div id="productChart" style="min-height: 350px;"></div>
<p id="lastUpdated" class="text-muted"></p>

<!-- Reference static ApexCharts and interop -->
<script src="~/lib/apexcharts/apexcharts.min.js"></script>
<script src="~/lib/apexcharts/apexInterop.js"></script>

<script>
  async function loadProductChart() {
    try {
      // 1) Trigger immediate SQL run and JSON rewrite
      await fetch(`/api/refresh/ProductMarkets`, { method: 'POST' });

      // 2) Fetch the updated JSON file
      const resp = await fetch(`/${"@Model.CacheFolder"}/ProductMarkets.json`);
      if (!resp.ok) {
        console.error("Failed to fetch ProductMarkets.json:", resp.statusText);
        return;
      }
      const data = await resp.json();

      // 3) Extract labels & values
      const labels = data.Rows.map(r => r.Label);
      const values = data.Rows.map(r => r.Value);

      // 4) Build ApexCharts config
      const options = {
        chart: {
          id: "productChart",
          type: "bar",
          toolbar: { show: true },
          zoom: { enabled: false }
        },
        xaxis: {
          categories: labels
        },
        title: {
          text: data.ChartId === "" ? "@Model.DefinitionTitle" : data.ChartId,
          align: "left"
        },
        series: [{
          name: data.ChartId,
          data: values
        }]
      };

      // 5) Clear any existing chart and re-render
      document.getElementById("productChart").innerHTML = "";
      apexInterop.renderChart("productChart", options);

      // 6) Update “Last updated”
      const dt = new Date(data.LastUpdatedUtc);
      document.getElementById("lastUpdated").innerText =
        "Last updated: " + dt.toLocaleString();
    }
    catch (err) {
      console.error("Error in loadProductChart:", err);
    }
  }

  document.getElementById("refreshBtn").addEventListener("click", loadProductChart);
  window.addEventListener("DOMContentLoaded", loadProductChart);
</script>
```

* We use JS `fetch("/api/refresh/ProductMarkets", { method: 'POST' })` to force‐refresh the cache.
* Then we `fetch("/ChartCache/ProductMarkets.json")`, parse it, and call `apexInterop.renderChart("productChart", options)`.

---

#### 4.8.2. File: `MyDashboard.Web/Pages/Product.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace MyDashboard.Web.Pages
{
    public class ProductModel : PageModel
    {
        private readonly IConfiguration _config;

        public string CacheFolder { get; }
        public string DefinitionTitle { get; } = "Markets Set in Last 30 Days";

        public ProductModel(IConfiguration config)
        {
            _config = config;
            // The “CacheFolder” should match the Worker’s output location
            CacheFolder = _config["CacheFolder"] ?? "ChartCache";
        }

        public void OnGet()
        {
            // Nothing needed here; JS does the actual loading
        }
    }
}
```

* We read `CacheFolder` from `appsettings.json`.
* `DefinitionTitle` is a fallback if `data.ChartId` is empty in JSON (it shouldn’t be).

---

#### 4.8.3. File: `MyDashboard.Web/Pages/Trade.cshtml`

```html
@page "/charts/Trade"
@model MyDashboard.Web.Pages.TradeModel
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers

@{
    ViewData["Title"] = "Trade Charts";
}

<h3>Charts for “Trade”</h3>

<h4>@Model.DefinitionTitle</h4>
<button class="btn btn-primary" id="refreshTradeBtn">Refresh Now</button>

<div id="tradeChart" style="min-height: 350px;"></div>
<p id="tradeUpdated" class="text-muted"></p>

<script src="~/lib/apexcharts/apexcharts.min.js"></script>
<script src="~/lib/apexcharts/apexInterop.js"></script>

<script>
  async function loadTradeChart() {
    try {
      // 1) Force immediate SQL run and rewrite
      await fetch(`/api/refresh/TradeTools`, { method: 'POST' });

      // 2) Fetch JSON
      const resp = await fetch(`/${"@Model.CacheFolder"}/TradeTools.json`);
      if (!resp.ok) {
        console.error("Failed to fetch TradeTools.json:", resp.statusText);
        return;
      }
      const data = await resp.json();

      // 3) Convert each row’s Label (string) into JS Date and value
      const points = data.Rows.map(r => ({
        x: new Date(r.Label), // Assumes Label is parseable as “dd-Mon-yyyy HH:mm”
        y: r.Value
      }));

      // 4) Build config
      const options = {
        chart: {
          id: "tradeChart",
          type: "scatter",
          toolbar: { show: true },
          zoom: { enabled: true }
        },
        xaxis: {
          type: "datetime",
          labels: { format: "dd MMM HH:mm" }
        },
        title: {
          text: data.ChartId == "" ? "@Model.DefinitionTitle" : data.ChartId,
          align: "left"
        },
        series: [{
          name: data.ChartId,
          data: points
        }]
      };

      // 5) Clear and render
      document.getElementById("tradeChart").innerHTML = "";
      apexInterop.renderChart("tradeChart", options);

      // 6) Update “Last updated”
      const dt = new Date(data.LastUpdatedUtc);
      document.getElementById("tradeUpdated").innerText =
        "Last updated: " + dt.toLocaleString();
    }
    catch (err) {
      console.error("Error in loadTradeChart:", err);
    }
  }

  document.getElementById("refreshTradeBtn").addEventListener("click", loadTradeChart);
  window.addEventListener("DOMContentLoaded", loadTradeChart);
</script>
```

* Exactly the same pattern as the Product page, but uses `TradeTools.json`.

---

#### 4.8.4. File: `MyDashboard.Web/Pages/Trade.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace MyDashboard.Web.Pages
{
    public class TradeModel : PageModel
    {
        private readonly IConfiguration _config;
        public string CacheFolder { get; }
        public string DefinitionTitle { get; } = "Trades Saved in Last 30 Days";

        public TradeModel(IConfiguration config)
        {
            _config = config;
            CacheFolder = _config["CacheFolder"] ?? "ChartCache";
        }

        public void OnGet()
        {
        }
    }
}
```

---

#### 4.8.5. Optionally update `Pages/Index.cshtml` as a landing page

You can leave it as default or add a link to your chart pages:

```html
@page
@model IndexModel

<h2>Welcome to MyDashboard</h2>
<ul>
  <li><a href="/charts/Product">Product Charts</a></li>
  <li><a href="/charts/Trade">Trade Charts</a></li>
</ul>
```

No code‐behind changes are needed (keep `Index.cshtml.cs` as generated).

---

### 4.9. Ensure `appsettings.json` is copied

In `MyDashboard.Web.csproj`, add:

```xml
<ItemGroup>
  <None Update="appsettings.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
</ItemGroup>
```

So that at runtime, `appsettings.json` sits next to the Web App’s `.exe`/`.dll`.

---

## 5. Build & Run

1. **Restore NuGet packages** at solution root:

   ```bash
   dotnet restore
   ```
2. **Build everything**:

   ```bash
   dotnet build
   ```
3. **Open two terminals** (or two VS instances).

   * **Terminal A**: run the Worker.

     ```bash
     cd MyDashboardSolution/MyDashboard.Worker
     dotnet run
     ```

     You should see logs like:

     ```
     info: MyDashboard.Worker.Services.ChartService[0]
           Loaded 2 chart definitions
     info: MyDashboard.Worker.Services.ChartPollingBackgroundService[0]
           Background refreshing 'ProductMarkets'
     info: MyDashboard.Worker.Services.ChartService[0]
           Refreshed 'ProductMarkets' → 5 rows, wrote ChartCache/ProductMarkets.json
     info: MyDashboard.Worker.Services.ChartPollingBackgroundService[0]
           Background refreshing 'TradeTools'
     info: MyDashboard.Worker.Services.ChartService[0]
           Refreshed 'TradeTools' → 10 rows, wrote ChartCache/TradeTools.json
     ```

   * **Terminal B**: run the Web App.

     ```bash
     cd MyDashboardSolution/MyDashboard.Web
     dotnet run
     ```

     The console will show something like:

     ```
     Now listening on: https://localhost:5001
     ```
4. **Browse** to `https://localhost:5001/charts/Product`. You should see a bar chart of “Markets Set in Last 30 Days.” The “Last updated” text shows the Worker’s most recent run time. Click **“Refresh Now”** to force a fresh SQL run and redraw.
5. **Browse** to `https://localhost:5001/charts/Trade`. You should see a scatter (or line) chart of “Trades Saved in Last 30 Days.” Again, “Refresh Now” re‐runs SQL immediately via the same `ChartService`.

---

## 6. How the “Refresh Now” flow works

1. **User clicks “Refresh Now”** in the browser.
2. JS calls `POST /api/refresh/{chartId}` (e.g. `POST /api/refresh/ProductMarkets`).
3. **RefreshController** in Web App receives that request, does `await _chartService.RefreshChartAsync(chartId)`.

   * `ChartService.RefreshChartAsync` opens Oracle, runs the SQL, builds a `ChartDataCache` object, **writes** it to `ChartCache/{chartId}.json`, and returns it.
4. The controller returns `200 OK`.
5. JS then calls `fetch("/ChartCache/ProductMarkets.json")` to load the newly‐written JSON.
6. JS parses the JSON into `{ ChartId, LastUpdatedUtc, Rows: [ { Label,Value }, … ] }`.
7. JS builds an ApexCharts `options` object and calls `apexInterop.renderChart("productChart", options)`, causing the chart to be drawn.

---

## 7. Folder Permissions (Production)

* **Worker Service** must have write permission to the `ChartCache/` folder.
* **Web App** must have read permission (and also write permission if you want the Web App’s refresh endpoint to work). In most setups, both processes run under the same OS account or the Web App’s Pool User also has write access.

If you host in IIS:

1. Deploy `MyDashboard.Worker` and `MyDashboard.Web` to the same folder, e.g. `D:\MyDashboard\`.
2. In `D:\MyDashboard\`, there must be a subfolder `ChartCache\`. Grant the IIS AppPool user (e.g. `IIS AppPool\MyDashboardWeb`) read/write, and grant the Worker’s service account (if different) full control.
3. Configure the Worker project as a Windows Service, pointing its working directory to `D:\MyDashboard\`.
4. Configure the Web App as an IIS Website, pointing to `D:\MyDashboard\MyDashboard.Web`.

---

## 8. Summary

You now have:

* **MyDashboard.Shared**: models that define `ChartDefinition`, `ChartDataRow`, `ChartDataCache`.
* **MyDashboard.Worker**: a background service that:

  * Reads `chart-definitions.json` and SQL files.
  * Polls Oracle on a schedule (every `RefreshIntervalSeconds`).
  * Writes JSON cache files under `ChartCache/{chartId}.json`.
* **MyDashboard.Web**: a Web App that:

  * Exposes `POST /api/refresh/{chartId}` to force‐refresh a chart from the browser.
  * Provides Razor Pages (`/charts/Product` and `/charts/Trade`) that:

    1. Call `POST /api/refresh/{chartId}`.
    2. Fetch `/ChartCache/{chartId}.json`.
    3. Render ApexCharts via `apexInterop.renderChart(divId, config)`.

By copying each code snippet into the exact file path shown, you’ll have a working Worker + Web architecture where the user can click “Refresh Now” to force a database poll and immediately see updated charts in the browser.
