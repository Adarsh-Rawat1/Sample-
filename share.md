using StarTrendsDashboard.Components;
using StarTrendsDashboard.Services;

var builder = WebApplication.CreateBuilder(args);

// 1) Add Razor Components + server‐side interactivity + HeadOutlet
builder.Services
    .AddRazorComponents()
    .AddInteractiveServerComponents()
    .AddHeadOutlet();

// 2) Register your chart‐caching service + optional background worker
builder.Services.AddSingleton<IChartService, ChartService>();
builder.Services.AddHostedService<ChartPollingBackgroundService>();

// 3) Controllers for your API endpoints
builder.Services.AddControllers();

// 4) (Optional) CORS policy
builder.Services.AddCors(o => o.AddDefaultPolicy(p =>
{
    p.AllowAnyOrigin()
     .AllowAnyHeader()
     .AllowAnyMethod();
}));

var app = builder.Build();

// 5) Production exception / HSTS
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error", createScopeForErrors: true);
    app.UseHsts();
}

// 6) Security & static files
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseAntiforgery();

// 7) Map your API controllers
app.MapControllers();

// 8) Map your Blazor root component (<Routes/> in App.razor)
app.MapRazorComponents
 



 <link rel="icon" type="image/png" href="favicon.png" />

+ <!-- ApexCharts CSS -->
+ <link rel="stylesheet" href="lib/apexcharts/apexcharts.css" />

  <HeadOutlet />
</head>
<body>
  <Routes />

  <script src="_framework/blazor.web.js"></script>

+ <!-- ApexCharts JS + interop -->
+ <script src="lib/apexcharts/apexcharts.min.js"></script>
+ <script src="lib/apexcharts/apexInterop.js"></script>







### Services/ChartService.cs

```csharp
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

            _definitionsJsonPath = config["ChartDefinitionsPath"]
                ?? throw new ArgumentNullException("ChartDefinitionsPath missing");
            _queryFolder = config["ChartQueryFolder"]
                ?? throw new ArgumentNullException("ChartQueryFolder missing");
            _cacheFolder = config["ChartCacheFolder"]
                ?? throw new ArgumentNullException("ChartCacheFolder missing");
            _connectionString = config.GetConnectionString("OracleDb")
                ?? throw new ArgumentNullException("OracleDb missing");

            // Ensure ChartCache directory exists
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
                return _definitions.ToList();
            }
        }

        public async Task<ChartDataCache> RefreshChartAsync(string chartId)
        {
            LoadDefinitions();

            ChartDefinition? def;
            lock (_lock)
            {
                def = _definitions
                    .FirstOrDefault(d => d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
            }
            if (def == null)
            {
                _logger.LogWarning("No definition for '{chartId}'", chartId);
                return new ChartDataCache
                {
                    ChartId = chartId,
                    LastUpdatedUtc = DateTime.UtcNow
                };
            }

            var sqlPath = Path.Combine(_queryFolder, def.SqlFile);
            if (!File.Exists(sqlPath))
            {
                _logger.LogError("SQL file not found: {sqlPath}", sqlPath);
                return new ChartDataCache
                {
                    ChartId = chartId,
                    LastUpdatedUtc = DateTime.UtcNow
                };
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

---

### Controllers/ChartDataController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using StarTrendsDashboard.Services;
using StarTrendsDashboard.Shared;
using System;
using System.IO;
using System.Text.Json;

namespace StarTrendsDashboard.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ChartDataController : ControllerBase
    {
        private readonly IChartService _chartService;
        private readonly string _cacheFolder;

        public ChartDataController(IChartService chartService, IConfiguration config)
        {
            _chartService = chartService;
            _cacheFolder = config["ChartCacheFolder"] ?? "ChartCache";
        }

        // GET /api/chartdata/{chartId}
        [HttpGet("{chartId}")]
        public IActionResult Get(string chartId)
        {
            var jsonPath = Path.Combine(Directory.GetCurrentDirectory(),
                                        _cacheFolder,
                                        $"{chartId}.json");
            if (System.IO.File.Exists(jsonPath))
            {
                var json = System.IO.File.ReadAllText(jsonPath);
                var cache = JsonSerializer.Deserialize<ChartDataCache>(json)
                            ?? new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };
                return Ok(cache);
            }

            // If no cached JSON, run SQL now
            var fresh = _chartService.RefreshChartAsync(chartId).Result;
            return Ok(fresh);
        }

        // POST /api/chartdata/refresh/{chartId}
        [HttpPost("refresh/{chartId}")]
        public IActionResult RefreshNow(string chartId)
        {
            var fresh = _chartService.RefreshChartAsync(chartId).Result;
            return Ok(fresh);
        }
    }
}
```

Just drop these two files into your project (replacing any stale versions), rebuild, and all of your chart‐caching and API endpoints will be back in place.




Below is a **from-scratch, step-by-step** guide to get your Blazor Server “StarTrendsDashboard” up and running with **category pages** (one page per `Page`, each showing all its charts in sequence). After this, **adding a new chart** is still just: drop a `.sql` → update JSON → rebuild.

---

## 0. Prerequisites

* **Visual Studio 2022/2023** with “ASP.NET & web development” + “.NET desktop development.”
* **.NET 8.0 SDK** installed.
* **Oracle.ManagedDataAccess.Core** NuGet (we’ll add it).
* A working Oracle connection string.

---

## 1. Create the Solution & Projects

1. **File → New → Project → Blank Solution**

   * Name: `StarTrendsDashboard`
   * Location: `C:\Projects\StarTrendsDashboard`
   * Click **Create**.

2. **Add Shared Class Library**

   * Right-click solution → Add → New Project → Class Library (.NET)
   * Name: `StarTrendsDashboard.Shared`
   * Delete the default `Class1.cs`.
   * Add three files under `StarTrendsDashboard.Shared`:

     ```csharp
     // ChartDefinition.cs
     namespace StarTrendsDashboard.Shared {
       public class ChartDefinition {
         public string ChartId { get; set; } = "";
         public string Page { get; set; } = "";
         public string Title { get; set; } = "";
         public string ChartType { get; set; } = "";
         public string SqlFile { get; set; } = "";
         public int RefreshIntervalSeconds { get; set; } = 300;
       }
     }
     ```

     ```csharp
     // ChartDataRow.cs
     namespace StarTrendsDashboard.Shared {
       public class ChartDataRow {
         public string Label { get; set; } = "";
         public decimal Value { get; set; }
       }
     }
     ```

     ```csharp
     // ChartDataCache.cs
     using System;
     using System.Collections.Generic;
     namespace StarTrendsDashboard.Shared {
       public class ChartDataCache {
         public string ChartId { get; set; } = "";
         public DateTime LastUpdatedUtc { get; set; }
         public List<ChartDataRow> Rows { get; set; } = new();
       }
     }
     ```
   * Build `StarTrendsDashboard.Shared`.

3. **Add Blazor Server App**

   * Right-click solution → Add → New Project → Blazor Web App → Next
   * Name: `StarTrendsDashboard`
   * Target: **.NET 8.0**, Authentication **None**, HTTPS **On**, Render Mode **Server** → Create.
   * Delete the sample `Data/` folder if present.
   * Add Project Reference → select `StarTrendsDashboard.Shared`.

---

## 2. Folder Setup & Static Assets

Inside **StarTrendsDashboard** (the server project):

1. **ChartDefinitions/**

   * Add Folder `ChartDefinitions` → inside it, JSON file `chart-definitions.json` (see below).
   * Add Folder `ChartDefinitions/Queries` → .sql files live here.

2. **ChartCache/**

   * Add Folder `ChartCache` → add an empty file `.gitkeep` → set its **Copy to Output Directory** → **Always**.

3. **wwwroot/lib/apexcharts/**

   * Copy your downloaded `apexcharts.min.js` + `apexcharts.css` + create `apexInterop.js` (see code below).
   * On each file’s Properties: **Copy to Output Directory** → **Copy if newer**.

---

## 3. chart-definitions.json

```jsonc
[
  {
    "ChartId": "ProductMarkets",
    "Page": "Product",
    "Title": "Markets Set in Last 30 Days",
    "ChartType": "bar",
    "SqlFile": "ProductMarkets.sql",
    "RefreshIntervalSeconds": 300
  },
  {
    "ChartId": "TradeTools",
    "Page": "Trade",
    "Title": "Trades Saved in Last 30 Days",
    "ChartType": "scatter",
    "SqlFile": "TradeTools.sql",
    "RefreshIntervalSeconds": 300
  }
  // ← append more here
]
```

* **Properties** of `chart-definitions.json`: **Copy if newer**.

---

## 4. Sample SQL Files

**ChartDefinitions/Queries/ProductMarkets.sql**

```sql
SELECT r.feature AS "Markets",
       COUNT(1)    AS "Times used"
FROM star_action_audit r
WHERE r.mod_dt > TRUNC(SYSDATE) - 30
  AND feature_type = 'MARKET'
GROUP BY r.feature
ORDER BY "Times used" DESC;
```

**ChartDefinitions/Queries/TradeTools.sql**

```sql
SELECT r.feature AS "Tools",
       COUNT(1)    AS "Times used"
FROM star_action_audit r
WHERE r.mod_dt > TRUNC(SYSDATE) - 30
  AND feature_type = 'TOOLS'
GROUP BY r.feature
ORDER BY "Times used" DESC;
```

* Each .sql: **Copy if newer**.

---

## 5. appsettings.json

```jsonc
{
  "ConnectionStrings": {
    "OracleDb": "User Id=MYUSER;Password=MYPASS;Data Source=//myhost:1521/ORCLPDB"
  },
  "ChartDefinitionsPath": "ChartDefinitions/chart-definitions.json",
  "ChartQueryFolder": "ChartDefinitions/Queries",
  "ChartCacheFolder": "ChartCache"
}
```

* Replace credentials.
* **Copy if newer**.

---

## 6. Services & Controller

1. **Install NuGet**

   * Oracle.ManagedDataAccess.Core

2. **Services/IChartService.cs**

   ```csharp
   using StarTrendsDashboard.Shared;
   using System.Collections.Generic;
   using System.Threading.Tasks;

   namespace StarTrendsDashboard.Services {
     public interface IChartService {
       IReadOnlyList<ChartDefinition> GetAllDefinitions();
       Task<ChartDataCache> RefreshChartAsync(string chartId);
     }
   }
   ```

3. **Services/ChartService.cs**
   *(Use exactly the implementation you already have; it reads JSON, runs SQL, writes cache).*

4. **Controllers/ChartDataController.cs**
   *(Exactly as before: GET reads `{chartId}.json` or calls `RefreshChartAsync`; POST forces refresh.)*

---

## 7. Program.cs

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using StarTrendsDashboard.Services;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IChartService, ChartService>();
builder.Services.AddHostedService<ChartPollingBackgroundService>(); // optional
builder.Services.AddControllers();
builder.Services.AddRazorPages();
builder.Services.AddServerSideBlazor();
builder.Services.AddCors(o => o.AddDefaultPolicy(p => p.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod()));

var app = builder.Build();
app.UseStaticFiles();
app.UseRouting();
app.UseCors();
app.MapControllers();
app.MapBlazorHub();
app.MapFallbackToPage("/_Host");
app.Run();
```

---

## 8. \_Imports.razor

```razor
@using System.Net.Http.Json
@using StarTrendsDashboard.Shared
@using Microsoft.JSInterop
```

---

## 9. ApexCharts Interop

**wwwroot/lib/apexcharts/apexInterop.js**

```js
window.apexInterop = {
  renderChart: (id, cfg) => {
    const el = document.querySelector(`#${id}`);
    if (!el) return;
    ApexCharts.getChartByID(id)?.destroy();
    new ApexCharts(el, cfg).render();
  }
};
```

In **Pages/\_Host.cshtml**, add inside `<head>`/`<body>`:

```html
<link href="lib/apexcharts/apexcharts.css" rel="stylesheet" />
<script src="lib/apexcharts/apexcharts.min.js"></script>
<script src="lib/apexcharts/apexInterop.js"></script>
```

---

## 10. Reusable ChartBlock Component

**Pages/ChartBlock.razor**

```razor
@using StarTrendsDashboard.Shared
@inject HttpClient Http
@inject IJSRuntime JS

<div class="mb-5">
  <h4>@Definition.Title</h4>
  <button class="btn btn-outline-primary mb-2"
          @onclick="Load" disabled="@isLoading">
    @if (isLoading) { <span>Loading…</span> }
    else           { <span>Refresh</span> }
  </button>

  @if (isLoading)
    <div class="spinner-border text-primary" role="status"></div>
  else if (loadError)
    <div class="alert alert-danger">Error loading data.</div>
  else if (hasNoData)
    <div class="alert alert-warning">No data available.</div>
  else
  {
    <div id="@elementId" style="min-height:300px"></div>
    <p class="text-muted">@LastUpdatedText</p>
  }
</div>

@code {
  [Parameter] public ChartDefinition Definition { get; set; } = default!;
  bool isLoading, hasNoData, loadError;
  string LastUpdatedText = "";
  string elementId => $"chart_{Definition.ChartId}";

  protected override async Task OnInitializedAsync() => await Load();

  async Task Load() {
    isLoading = true; hasNoData = false; loadError = false; StateHasChanged();
    try {
      var c = await Http.GetFromJsonAsync<ChartDataCache>($"api/chartdata/{Definition.ChartId}");
      if (c?.Rows.Count == 0) hasNoData = true;
      else {
        object opts = Definition.ChartType.ToLower() switch {
          "bar"     => BuildBar(c),
          "scatter" => BuildScatter(c),
          "line"    => BuildLine(c),
          _         => BuildBar(c)
        };
        await JS.InvokeVoidAsync("apexInterop.renderChart", elementId, opts);
        LastUpdatedText = $"Last updated: {c.LastUpdatedUtc.ToLocalTime():g}";
      }
    }
    catch { loadError = true; }
    finally { isLoading = false; StateHasChanged(); }
  }

  object BuildBar(ChartDataCache c) => new {
    chart = new { id = elementId, type = "bar", toolbar = new{show=true}, zoom = new{enabled=false} },
    xaxis = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    title = new { text = Definition.Title, align = "left" },
    series = new[]{ new { name = Definition.ChartId, data = c.Rows.Select(r=>r.Value).ToArray() } }
  };

  object BuildLine(ChartDataCache c) => new {
    chart = new { id = elementId, type = "line", toolbar = new{show=true}, zoom = new{enabled=true} },
    xaxis = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    title = new { text = Definition.Title, align = "left" },
    series = new[]{ new { name = Definition.ChartId, data = c.Rows.Select(r=>r.Value).ToArray() } }
  };

  object BuildScatter(ChartDataCache c) => new {
    chart = new { id = elementId, type = "scatter", toolbar=new{show=true}, zoom=new{enabled=true} },
    xaxis = new { type="datetime", labels=new{format="dd MMM HH:mm"} },
    title = new { text = Definition.Title, align = "left" },
    series = new[]{ new {
      name = Definition.ChartId,
      data = c.Rows.Select(r=>new object[]{ DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(), r.Value }).ToArray()
    } }
  };
}
```

---

## 11. Category Viewer Page

**Pages/CategoryViewer.razor**

```razor
@page "/charts/{page}"
@using StarTrendsDashboard.Shared
@inject IChartService ChartService

<h3>@page Charts</h3>

@code {
  [Parameter] public string page { get; set; } = "";
  List<ChartDefinition> defs = new();
  protected override void OnParametersSet() {
    defs = ChartService.GetAllDefinitions()
            .Where(d => d.Page.Equals(page, StringComparison.OrdinalIgnoreCase))
            .ToList();
  }
}

@if (!defs.Any())
{
  <div class="alert alert-secondary">No charts for “@page”.</div>
}
else
{
  @foreach (var d in defs)
  {
    <ChartBlock Definition="d" />
  }
}
```

---

## 12. Dynamic NavMenu

**Shared/NavMenu.razor**

```razor
@inject IChartService ChartService

<nav class="nav flex-column">
  @foreach (var cat in ChartService.GetAllDefinitions().Select(d=>d.Page).Distinct())
  {
    <NavLink class="nav-link"
             href="@($"/charts/{cat}")"
             Match="NavLinkMatch.Prefix">
      @cat
    </NavLink>
  }
</nav>
```

---

## 13. Run & Test

1. Set **StarTrendsDashboard** as Startup project.
2. Press **F5** → browse to **/charts/Product** and **/charts/Trade**.
3. Each page shows all charts in that category, with spinners, refresh buttons, empty/data/error states.

---

## 14. Adding a New Chart

1. **Drop** `MyNewChart.sql` into `ChartDefinitions/Queries/`.
2. **Append** to `chart-definitions.json`:

   ```jsonc
   {
     "ChartId":"MyNewChart",
     "Page":"Product",       // or "Trade", etc.
     "Title":"My New Chart",
     "ChartType":"line",
     "SqlFile":"MyNewChart.sql",
     "RefreshIntervalSeconds":300
   }
   ```
3. **Rebuild & run** → visit `/charts/Product` → you’ll see “My New Chart” appear automatically.

---

That’s it—your **zero-Razor-edits** extension model with **category pages** and **multiple charts** per page, complete from scratch. Enjoy!
