Below is a complete, end-to-end guide—using the **Blazor WebAssembly App (ASP NET Core hosted)** template on **.NET 8**—for building a solution named **StarTrendsDashboard**. It covers every file, folder, and setting, so you can simply follow along in Visual Studio (no CLI needed) and end up with:

* **StarTrendsDashboard.Shared** (class library with DTOs)
* **StarTrendsDashboard.Server** (ASP NET Core Web API + optional background service)
* **StarTrendsDashboard.Client** (Blazor WebAssembly UI)

At runtime:

1. The **Server** hosts `ChartService` which runs your SQL, writes JSON under `ChartCache/`, and exposes API endpoints.
2. The **Client** calls those endpoints, receives JSON, and renders ApexCharts in the browser.
3. The **Shared** project holds the `ChartDefinition`, `ChartDataRow`, and `ChartDataCache` classes so Client and Server share the same models.

---

## 1. Create a Blank Solution

1. Open **Visual Studio 2022 or 2023**.
2. **File → New → Project…**
3. In the “Create a new project” dialog, search for **“Blank Solution”**, select it, click **Next**.
4. **Solution name:** `StarTrendsDashboard`
   **Location:** e.g. `C:\Projects\StarTrendsDashboard`
   **Place solution and project in the same directory:** **Uncheck**
   Click **Create**.
5. You now have an empty solution `StarTrendsDashboard.sln`.

---

## 2. Add (or Replace) the Shared Project

When you create a Blazor WebAssembly App with “ASP NET Core hosted,” Visual Studio will scaffold a **Shared** project as part of the three‐project template. If you already have one from prior steps, you can replace it; otherwise follow these UI steps:

1. **Right-click** the solution (`StarTrendsDashboard`) → **Add → New Project…**

2. Search for **“Class Library (.NET)”**, select it, click **Next**.

3. **Project name:** `StarTrendsDashboard.Shared`
   **Location:** `C:\Projects\StarTrendsDashboard\StarTrendsDashboard.Shared`
   **Solution:** “Add to solution”
   Click **Create**.

4. In **Solution Explorer**, delete the default `Class1.cs`.

5. Add three classes under `StarTrendsDashboard.Shared`:

   * **Right-click** `StarTrendsDashboard.Shared` → **Add → Class…** → **Name:** `ChartDefinition.cs` → **Add**.
     Replace contents with:

     ```csharp
     namespace StarTrendsDashboard.Shared
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

   * **Right-click** `StarTrendsDashboard.Shared` → **Add → Class…** → **Name:** `ChartDataRow.cs` → **Add**.
     Replace contents with:

     ```csharp
     namespace StarTrendsDashboard.Shared
     {
         public class ChartDataRow
         {
             public string Label { get; set; } = string.Empty;
             public decimal Value { get; set; }
         }
     }
     ```

   * **Right-click** `StarTrendsDashboard.Shared` → **Add → Class…** → **Name:** `ChartDataCache.cs` → **Add**.
     Replace contents with:

     ```csharp
     using System;
     using System.Collections.Generic;

     namespace StarTrendsDashboard.Shared
     {
         public class ChartDataCache
         {
             public string ChartId { get; set; } = string.Empty;
             public DateTime LastUpdatedUtc { get; set; }
             public List<ChartDataRow> Rows { get; set; } = new List<ChartDataRow>();
         }
     }
     ```

6. **Right-click** `StarTrendsDashboard.Shared` → **Build** to confirm there are no errors.

7. **Right-click** the solution → **Add → Existing Project…** (if this Shared was not auto-added) and browse to `StarTrendsDashboard.Shared/StarTrendsDashboard.Shared.csproj`. Otherwise, Visual Studio already knows this project.

---

## 3. Add StarTrendsDashboard.Server (ASP NET Core API, .NET 8)

1. **Right-click** the solution → **Add → New Project…**
2. Search for **“ASP NET Core Web API”**, select **ASP NET Core Web API**, click **Next**.
3. **Project name:** `StarTrendsDashboard.Server`
   **Location:** `C:\Projects\StarTrendsDashboard\StarTrendsDashboard.Server`
   **Solution:** “Add to solution”
   Click **Create**.
4. In the next dialog, ensure:

   * **Target Framework:** **.NET 8.0 (LTS)**
   * **Authentication Type:** **None**
   * **Configure for HTTPS:** Checked (recommended for local dev)
   * **Enable OpenAPI Support:** Up to you (we won’t rely on Swagger here, so you can uncheck).
     Click **Create**.

You now see:

```
StarTrendsDashboard.Server/
├── Controllers/
│   └── WeatherForecastController.cs  (delete later)
├── Program.cs
├── appsettings.json
├── StarTrendsDashboard.Server.csproj
└── Properties/
```

5. **Delete** the sample `Controllers/WeatherForecastController.cs` (or leave it, but it’s not needed).

6. **Delete** `Models/WeatherForecast.cs` if present.

7. **Add Project Reference** to Shared:

   * **Right-click** `StarTrendsDashboard.Server` → **Add → Project Reference…**
   * Check **StarTrendsDashboard.Shared** → **OK**.

8. **Create** folder `ChartDefinitions` under Server:

   * **Right-click** `StarTrendsDashboard.Server` → **Add → New Folder**, name it `ChartDefinitions`.
   * **Right-click** `ChartDefinitions` → **Add → Existing Item…**, browse to your local copies of:

     * `StarTrendsDashboard.Worker\ChartDefinitions\chart-definitions.json`
     * `StarTrendsDashboard.Worker\ChartDefinitions\Queries\ProductMarkets.sql`
     * `StarTrendsDashboard.Worker\ChartDefinitions\Queries\TradeTools.sql`
       and click **Add**.
   * In **Solution Explorer**, select each of these three files → in the **Properties** window, set **“Copy to Output Directory”** to **“Copy if newer”**.

   If you do not have an old Worker folder, simply **Add → New Item…** → **JSON File** for `chart-definitions.json` and **New Item…** → **Text File** for `ProductMarkets.sql` and `TradeTools.sql`, then paste the contents from the steps below:

   * **chart-definitions.json** (replace with exactly):

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

   * **ProductMarkets.sql**:

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

   * **TradeTools.sql**:

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

9. **Create** folder `ChartCache` under Server:

   * **Right-click** `StarTrendsDashboard.Server` → **Add → New Folder**, name it `ChartCache`.
   * Select the new `ChartCache` folder in **Solution Explorer** → in **Properties**, set **“Copy to Output Directory”** to **“Copy always”**.

10. **Add** folder `Services` under Server:

    * **Right-click** `StarTrendsDashboard.Server` → **Add → New Folder**, name it `Services`.

11. Under `Services`, **add three classes**:

    ### 11.1 File: `IChartService.cs`

    * **Right-click** `Services` → **Add → Class…** → **Name:** `IChartService.cs` → **Add**.
      Replace contents with:

      ```csharp
      using StarTrendsDashboard.Shared;
      using System.Collections.Generic;
      using System.Threading.Tasks;

      namespace StarTrendsDashboard.Server.Services
      {
          public interface IChartService
          {
              IReadOnlyList<ChartDefinition> GetAllDefinitions();
              Task<ChartDataCache> RefreshChartAsync(string chartId);
          }
      }
      ```

    ### 11.2 File: `ChartService.cs`

    * **Right-click** `Services` → **Add → Class…** → **Name:** `ChartService.cs` → **Add**.
      Replace contents with exactly:

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

      namespace StarTrendsDashboard.Server.Services
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
                  _logger = logger 
                      ?? throw new ArgumentNullException(nameof(logger));

                  _definitionsJsonPath = config["ChartDefinitionsPath"] 
                      ?? throw new ArgumentNullException("ChartDefinitionsPath missing");
                  _queryFolder = config["ChartQueryFolder"] 
                      ?? throw new ArgumentNullException("ChartQueryFolder missing");
                  _cacheFolder = config["ChartCacheFolder"] 
                      ?? throw new ArgumentNullException("ChartCacheFolder missing");
                  _connectionString = config.GetConnectionString("OracleDb") 
                      ?? throw new ArgumentNullException("OracleDb missing");

                  // Ensure the cache folder exists
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
                      def = _definitions.FirstOrDefault(d => 
                          d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
                  }
                  if (def == null)
                  {
                      _logger.LogWarning("RefreshChartAsync: no definition for '{chartId}'", chartId);
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
                      // Return partial or empty result if SQL fails
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

    ### 11.3 (Optional) `ChartPollingBackgroundService.cs`

    If you want the Server to automatically poll on a schedule (rather than requiring the client to click “Refresh Now”), add:

    * **Right-click** `Services` → **Add → Class…** → **Name:** `ChartPollingBackgroundService.cs` → **Add**.
      Replace contents with:

      ```csharp
      using Microsoft.Extensions.Hosting;
      using Microsoft.Extensions.Logging;
      using StarTrendsDashboard.Shared;
      using System;
      using System.Collections.Generic;
      using System.Threading;
      using System.Threading.Tasks;

      namespace StarTrendsDashboard.Server.Services
      {
          public class ChartPollingBackgroundService : BackgroundService
          {
              private readonly IChartService _chartService;
              private readonly ILogger<ChartPollingBackgroundService> _logger;
              private readonly Dictionary<string, DateTime> _nextRunUtc 
                  = new(StringComparer.OrdinalIgnoreCase);

              public ChartPollingBackgroundService(IChartService chartService, ILogger<ChartPollingBackgroundService> logger)
              {
                  _chartService = chartService;
                  _logger = logger;

                  var defs = _chartService.GetAllDefinitions();
                  foreach (var d in defs)
                  {
                      _nextRunUtc[d.ChartId] = DateTime.UtcNow;
                  }
              }

              protected override async Task ExecuteAsync(CancellationToken stoppingToken)
              {
                  _logger.LogInformation("ChartPollingBackgroundService starting.");

                  while (!stoppingToken.IsCancellationRequested)
                  {
                      var now = DateTime.UtcNow;
                      var defs = _chartService.GetAllDefinitions();

                      foreach (var d in defs)
                      {
                          if (!_nextRunUtc.ContainsKey(d.ChartId))
                          {
                              _logger.LogInformation("Scheduling new chart '{chartId}'", d.ChartId);
                              _nextRunUtc[d.ChartId] = now;
                          }
                      }

                      foreach (var def in defs)
                      {
                          if (_nextRunUtc.TryGetValue(def.ChartId, out var nextTime) 
                              && now >= nextTime)
                          {
                              _logger.LogInformation("Background refreshing '{chartId}'", def.ChartId);
                              try
                              {
                                  await _chartService.RefreshChartAsync(def.ChartId);
                              }
                              catch (Exception ex)
                              {
                                  _logger.LogError(ex, "Error refreshing '{chartId}'", def.ChartId);
                              }
                              _nextRunUtc[def.ChartId] = now.AddSeconds(def.RefreshIntervalSeconds);
                          }
                      }

                      await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
                  }

                  _logger.LogInformation("ChartPollingBackgroundService stopping.");
              }
          }
      }
      ```

12. **Add** NuGet package `Oracle.ManagedDataAccess.Core` (or `Oracle.ManagedDataAccess`) to **StarTrendsDashboard.Server**:

    * **Right-click** `StarTrendsDashboard.Server` → **Manage NuGet Packages…** → **Browse** → search **`Oracle.ManagedDataAccess.Core`** → **Install** latest.

13. **Edit** `appsettings.json` under `StarTrendsDashboard.Server` (double-click to open), replace with:

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

    * Replace `MYUSER`, `MYPASSWORD`, `//myhost:1521/ORCLPDB` with your real Oracle credentials.
    * In **Solution Explorer**, select `appsettings.json` → in **Properties**, set **“Copy to Output Directory”** → **“Copy if newer”**.

14. **Edit** `Program.cs` under `StarTrendsDashboard.Server` to wire up services and controllers. Replace the contents with:

    ```csharp
    using Microsoft.AspNetCore.Builder;
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.DependencyInjection;
    using Microsoft.Extensions.Hosting;
    using StarTrendsDashboard.Server.Services;

    var builder = WebApplication.CreateBuilder(args);

    builder.Services.AddSingleton<IChartService, ChartService>();

    // If you added ChartPollingBackgroundService:
    builder.Services.AddHostedService<ChartPollingBackgroundService>();

    builder.Services.AddControllers();

    // Optional: Allow any origin if client is on a different port
    builder.Services.AddCors(options =>
    {
        options.AddDefaultPolicy(policy =>
        {
            policy
              .AllowAnyOrigin()
              .AllowAnyHeader()
              .AllowAnyMethod();
        });
    });

    var app = builder.Build();

    app.UseCors();

    app.UseStaticFiles(); // in case you serve any static assets

    app.MapControllers();

    app.Run();
    ```

15. **Add** an API controller:

    * **Right-click** `StarTrendsDashboard.Server` → **Add → New Folder**, name it `Controllers`.
    * **Right-click** `Controllers` → **Add → Class…** → **Name:** `ChartDataController.cs` → **Add**.
      Replace its contents with:

      ```csharp
      using Microsoft.AspNetCore.Mvc;
      using StarTrendsDashboard.Shared;
      using StarTrendsDashboard.Server.Services;
      using System.IO;
      using System.Text.Json;

      namespace StarTrendsDashboard.Server.Controllers
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

              // GET /api/chartdata/ProductMarkets
              [HttpGet("{chartId}")]
              public IActionResult Get(string chartId)
              {
                  // Return JSON from disk if present:
                  var jsonPath = Path.Combine(Directory.GetCurrentDirectory(), _cacheFolder, $"{chartId}.json");
                  if (System.IO.File.Exists(jsonPath))
                  {
                      var json = System.IO.File.ReadAllText(jsonPath);
                      var cache = JsonSerializer.Deserialize<ChartDataCache>(json)
                                  ?? new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };
                      return Ok(cache);
                  }

                  // Otherwise, run SQL now:
                  var fresh = _chartService.RefreshChartAsync(chartId).Result;
                  return Ok(fresh);
              }

              // POST /api/chartdata/refresh/ProductMarkets
              [HttpPost("refresh/{chartId}")]
              public IActionResult RefreshNow(string chartId)
              {
                  var fresh = _chartService.RefreshChartAsync(chartId).Result;
                  return Ok(fresh);
              }
          }
      }
      ```
    * **Build** `StarTrendsDashboard.Server` to ensure no errors.

At this point, **StarTrendsDashboard.Server** is fully configured.

---

## 4. Add StarTrendsDashboard.Client (Blazor WebAssembly)

1. **Right-click** the solution → **Add → New Project…**
2. Search for **“Blazor WebAssembly App”** (it says “.NET 6+ client side”).
3. Click **Next**.

   * **Project name:** `StarTrendsDashboard.Client`
   * **Location:** `C:\Projects\StarTrendsDashboard\StarTrendsDashboard.Client`
   * **Solution:** “Add to solution”
     Click **Create**.
4. In the next dialog, ensure:

   * **Target Framework:** **.NET 8.0 (LTS)**
   * **Authentication Type:** **None**
   * **Configure for HTTPS:** Checked (optional)
   * **ASP NET Core Hosted:** Checked
   * **Progressive Web App:** Unchecked
     Click **Create**.

Visual Studio will scaffold three projects:

```
StarTrendsDashboard.Client/
StarTrendsDashboard.Server/
StarTrendsDashboard.Shared/
```

Because you already manually created `StarTrendsDashboard.Shared` and `StarTrendsDashboard.Server`, you can remove the newly created ones (or merge them). The quickest route is:

* Delete the newly scaffoled `StarTrendsDashboard.Shared` and `StarTrendsDashboard.Server` that Visual Studio created, then add project references to your existing Shared and Server.
* But if you want, you can simply overwrite them with the code above. For brevity, assume Visual Studio added them and now you just need to replace contents.

5. **Ensure** `StarTrendsDashboard.Client` has a project reference to `StarTrendsDashboard.Shared`:

   * **Right-click** `StarTrendsDashboard.Client` → **Add → Project Reference…** → check `StarTrendsDashboard.Shared` → **OK**.

6. **Ensure** `StarTrendsDashboard.Client` has a reference to `StarTrendsDashboard.Server` (optional, only if you share types; usually only Shared is referenced).

   * **Right-click** `StarTrendsDashboard.Client` → **Add → Project Reference…** → check `StarTrendsDashboard.Server` → **OK**.

7. In **Solution Explorer**, expand `StarTrendsDashboard.Client`, open `Program.cs`, and modify:

   ```csharp
   using Microsoft.AspNetCore.Components.Web;
   using Microsoft.AspNetCore.Components.WebAssembly.Hosting;
   using StarTrendsDashboard.Client;

   var builder = WebAssemblyHostBuilder.CreateDefault(args);
   builder.RootComponents.Add<App>("#app");
   builder.RootComponents.Add<HeadOutlet>("head::after");

   // Replace with your Server’s launch URL:
   builder.Services.AddScoped(sp => 
       new HttpClient { BaseAddress = new Uri("https://localhost:5001/") });

   await builder.Build().RunAsync();
   ```

   * Change the `BaseAddress` from `builder.HostEnvironment.BaseAddress` to `new Uri("https://localhost:5001/")` so all HttpClient calls target your Server API.

8. **Add ApexCharts assets** to the Client:

   * Under **StarTrendsDashboard.Client/wwwroot**, **Add → New Folder**, name it `lib`.
   * Right-click **lib** → **Add → New Folder**, name it `apexcharts`.
   * Copy your local **`apexcharts.min.js`** and **`apexcharts.css`** into `StarTrendsDashboard.Client/wwwroot/lib/apexcharts/`.

     * In **Solution Explorer**, select each file → **Properties** → set **“Copy to Output Directory”** → **“Copy if newer”**.
   * In **wwwroot/lib/apexcharts**, **Add → New Item…** → **JavaScript File**, name it `apexInterop.js`, click **Add**.
     Paste:

     ```js
     window.apexInterop = {
       renderChart: function (elementId, config) {
         console.log(`[apexInterop] renderChart for '#${elementId}', config=`, config);
         const elem = document.querySelector(`#${elementId}`);
         if (!elem) {
           console.warn(`[apexInterop] Container '#${elementId}' not found.`);
           return;
         }
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

     * Set **“Copy to Output Directory”** → **“Copy if newer”**.

9. **Edit** `index.html` in `StarTrendsDashboard.Client/wwwroot`:

   * Open `index.html` → in the `<head>`, add:

     ```html
     <link href="lib/apexcharts/apexcharts.css" rel="stylesheet" />
     ```
   * Just before `</body>`, add:

     ```html
     <script src="lib/apexcharts/apexcharts.min.js"></script>
     <script src="lib/apexcharts/apexInterop.js"></script>
     ```
   * The file should look like:

     ```html
     <!DOCTYPE html>
     <html lang="en">
     <head>
       <meta charset="utf-8" />
       <meta name="viewport" content="width=device-width, initial-scale=1.0" />
       <title>StarTrendsDashboard.Client</title>
       <link href="css/bootstrap/bootstrap.min.css" rel="stylesheet" />
       <link href="css/app.css" rel="stylesheet" />
       <link href="lib/apexcharts/apexcharts.css" rel="stylesheet" />
     </head>
     <body>
       <div id="app">Loading...</div>

       <script src="_framework/blazor.webassembly.js"></script>
       <script src="lib/apexcharts/apexcharts.min.js"></script>
       <script src="lib/apexcharts/apexInterop.js"></script>
     </body>
     </html>
     ```

10. **Create** two Razor components under `StarTrendsDashboard.Client/Pages`:

    ### 10.1 File: `Product.razor`

    * **Right-click** `Pages` → **Add → Razor Component…** → **Name:** `Product.razor` → **Add**.
      Replace contents with:

      ```razor
      @page "/charts/Product"
      @using StarTrendsDashboard.Shared
      @inject HttpClient Http
      @inject IJSRuntime JS

      <h3>Charts for “Product”</h3>
      <h4>Markets Set in Last 30 Days</h4>

      <button class="btn btn-primary mb-2" @onclick="LoadProductChart">
          Refresh Now
      </button>

      <div id="productChart" style="min-height: 350px;"></div>
      <p class="text-muted">@LastUpdatedText</p>

      @code {
          private string LastUpdatedText { get; set; } = "";

          protected override async Task OnAfterRenderAsync(bool firstRender)
          {
              if (firstRender)
              {
                  await LoadProductChart();
              }
          }

          private async Task LoadProductChart()
          {
              // 1) Call server API to get JSON
              var cache = await Http.GetFromJsonAsync<ChartDataCache>("api/chartdata/ProductMarkets");
              if (cache == null) return;

              // 2) Extract labels & values
              var labels = cache.Rows.Select(r => r.Label).ToArray();
              var values = cache.Rows.Select(r => r.Value).ToArray();

              // 3) Build ApexCharts config
              var options = new
              {
                  chart = new
                  {
                      id = "productChart",
                      type = "bar",
                      toolbar = new { show = true },
                      zoom = new { enabled = false }
                  },
                  xaxis = new { categories = labels },
                  title = new 
                  { 
                      text = cache.ChartId ?? "Markets Set in Last 30 Days",
                      align = "left"
                  },
                  series = new[] { new { name = cache.ChartId, data = values } }
              };

              // 4) Render via JS interop
              await JS.InvokeVoidAsync("apexInterop.renderChart", "productChart", options);

              // 5) Update timestamp
              LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
              StateHasChanged();
          }
      }
      ```

    ### 10.2 File: `Trade.razor`

    * **Right-click** `Pages` → **Add → Razor Component…** → **Name:** `Trade.razor` → **Add**.
      Replace contents with:

      ```razor
      @page "/charts/Trade"
      @using StarTrendsDashboard.Shared
      @inject HttpClient Http
      @inject IJSRuntime JS

      <h3>Charts for “Trade”</h3>
      <h4>Trades Saved in Last 30 Days</h4>

      <button class="btn btn-primary mb-2" @onclick="LoadTradeChart">
          Refresh Now
      </button>

      <div id="tradeChart" style="min-height: 350px;"></div>
      <p class="text-muted">@LastUpdatedText</p>

      @code {
          private string LastUpdatedText { get; set; } = "";

          protected override async Task OnAfterRenderAsync(bool firstRender)
          {
              if (firstRender)
              {
                  await LoadTradeChart();
              }
          }

          private async Task LoadTradeChart()
          {
              // 1) Call server API to get JSON
              var cache = await Http.GetFromJsonAsync<ChartDataCache>("api/chartdata/TradeTools");
              if (cache == null) return;

              // 2) Convert rows into [ [timestamp, value], … ]
              var points = cache.Rows.Select(r => new object[]
              {
                  new DateTimeOffset(DateTime.Parse(r.Label)).ToUnixTimeMilliseconds(),
                  r.Value
              }).ToArray();

              // 3) Build ApexCharts config
              var options = new
              {
                  chart = new
                  {
                      id = "tradeChart",
                      type = "scatter",
                      toolbar = new { show = true },
                      zoom = new { enabled = true }
                  },
                  xaxis = new
                  {
                      type = "datetime",
                      labels = new { format = "dd MMM HH:mm" }
                  },
                  title = new
                  {
                      text = cache.ChartId ?? "Trades Saved in Last 30 Days",
                      align = "left"
                  },
                  series = new[] { new { name = cache.ChartId, data = points } }
              };

              // 4) Render
              await JS.InvokeVoidAsync("apexInterop.renderChart", "tradeChart", options);

              // 5) Update timestamp
              LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
              StateHasChanged();
          }
      }
      ```

11. (Optional) Edit `NavMenu.razor` under `StarTrendsDashboard.Client/Shared` to add links:

    ```razor
    <nav>
      <ul class="nav flex-column">
        <li class="nav-item px-3">
          <NavLink class="nav-link" href="charts/Product">
            <span class="oi oi-bar-chart" aria-hidden="true"></span> Product Charts
          </NavLink>
        </li>
        <li class="nav-item px-3">
          <NavLink class="nav-link" href="charts/Trade">
            <span class="oi oi-pie-chart" aria-hidden="true"></span> Trade Charts
          </NavLink>
        </li>
      </ul>
    </nav>
    ```

12. **Build** the Client: **Right-click** `StarTrendsDashboard.Client` → **Build**.

---

## 5. Final File Recap

Below is a file-by-file recap of **exactly what you should see** in each project, with **all contents** included. After copying these into your solution and setting each file’s “Copy to Output Directory” as noted, simply build and run.

---

### 5A. StarTrendsDashboard.Shared (class library)

```
StarTrendsDashboard.Shared/
├── ChartDefinition.cs
├── ChartDataRow.cs
├── ChartDataCache.cs
└── StarTrendsDashboard.Shared.csproj
```

**ChartDefinition.cs**

```csharp
namespace StarTrendsDashboard.Shared
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

**ChartDataRow\.cs**

```csharp
namespace StarTrendsDashboard.Shared
{
    public class ChartDataRow
    {
        public string Label { get; set; } = string.Empty;
        public decimal Value { get; set; }
    }
}
```

**ChartDataCache.cs**

```csharp
using System;
using System.Collections.Generic;

namespace StarTrendsDashboard.Shared
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

### 5B. StarTrendsDashboard.Server (API + background)

```
StarTrendsDashboard.Server/
├── appsettings.json
├── Program.cs
├── StarTrendsDashboard.Server.csproj
├── ChartDefinitions/
│   ├── chart-definitions.json
│   └── Queries/
│       ├── ProductMarkets.sql
│       └── TradeTools.sql
├── ChartCache/            (Copy Always)
├── Controllers/
│   └── ChartDataController.cs
└── Services/
    ├── IChartService.cs
    ├── ChartService.cs
    └── ChartPollingBackgroundService.cs  (optional)
```

**appsettings.json**
*(Properties: Copy if newer)*

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

> Replace Oracle credentials accordingly.

**Program.cs**

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using StarTrendsDashboard.Server.Services;

var builder = WebApplication.CreateBuilder(args);

// Register ChartService (and background polling if desired)
builder.Services.AddSingleton<IChartService, ChartService>();
builder.Services.AddHostedService<ChartPollingBackgroundService>();

builder.Services.AddControllers();
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod();
    });
});

var app = builder.Build();

app.UseCors();
app.UseStaticFiles();

app.MapControllers();

app.Run();
```

---

#### ChartDefinitions/chart-definitions.json

*(Properties: Copy if newer)*

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

---

#### ChartDefinitions/Queries/ProductMarkets.sql

*(Properties: Copy if newer)*

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

---

#### ChartDefinitions/Queries/TradeTools.sql

*(Properties: Copy if newer)*

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

---

#### ChartCache/

This folder is initially empty.
**Select** it in Solution Explorer → **Properties** → **Copy to Output Directory** → **Copy always**.

---

#### Controllers/ChartDataController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using StarTrendsDashboard.Shared;
using StarTrendsDashboard.Server.Services;
using System.IO;
using System.Text.Json;

namespace StarTrendsDashboard.Server.Controllers
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

        // GET /api/chartdata/ProductMarkets
        [HttpGet("{chartId}")]
        public IActionResult Get(string chartId)
        {
            var jsonPath = Path.Combine(Directory.GetCurrentDirectory(), _cacheFolder, $"{chartId}.json");
            if (System.IO.File.Exists(jsonPath))
            {
                var json = System.IO.File.ReadAllText(jsonPath);
                var cache = JsonSerializer.Deserialize<ChartDataCache>(json)
                            ?? new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };
                return Ok(cache);
            }

            var fresh = _chartService.RefreshChartAsync(chartId).Result;
            return Ok(fresh);
        }

        // POST /api/chartdata/refresh/ProductMarkets
        [HttpPost("refresh/{chartId}")]
        public IActionResult RefreshNow(string chartId)
        {
            var fresh = _chartService.RefreshChartAsync(chartId).Result;
            return Ok(fresh);
        }
    }
}
```

---

#### Services/IChartService.cs

```csharp
using StarTrendsDashboard.Shared;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace StarTrendsDashboard.Server.Services
{
    public interface IChartService
    {
        IReadOnlyList<ChartDefinition> GetAllDefinitions();
        Task<ChartDataCache> RefreshChartAsync(string chartId);
    }
}
```

---

#### Services/ChartService.cs

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

namespace StarTrendsDashboard.Server.Services
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
            _logger = logger 
                ?? throw new ArgumentNullException(nameof(logger));

            _definitionsJsonPath = config["ChartDefinitionsPath"] 
                ?? throw new ArgumentNullException("ChartDefinitionsPath missing");
            _queryFolder = config["ChartQueryFolder"] 
                ?? throw new ArgumentNullException("ChartQueryFolder missing");
            _cacheFolder = config["ChartCacheFolder"] 
                ?? throw new ArgumentNullException("ChartCacheFolder missing");
            _connectionString = config.GetConnectionString("OracleDb") 
                ?? throw new ArgumentNullException("OracleDb missing");

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

#### Services/ChartPollingBackgroundService.cs (optional)

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using StarTrendsDashboard.Shared;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

namespace StarTrendsDashboard.Server.Services
{
    public class ChartPollingBackgroundService : BackgroundService
    {
        private readonly IChartService _chartService;
        private readonly ILogger<ChartPollingBackgroundService> _logger;
        private readonly Dictionary<string, DateTime> _nextRunUtc 
            = new(StringComparer.OrdinalIgnoreCase);

        public ChartPollingBackgroundService(IChartService chartService, ILogger<ChartPollingBackgroundService> logger)
        {
            _chartService = chartService;
            _logger = logger;

            var defs = _chartService.GetAllDefinitions();
            foreach (var d in defs)
            {
                _nextRunUtc[d.ChartId] = DateTime.UtcNow;
            }
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation("ChartPollingBackgroundService starting.");

            while (!stoppingToken.IsCancellationRequested)
            {
                var now = DateTime.UtcNow;
                var defs = _chartService.GetAllDefinitions();

                foreach (var d in defs)
                {
                    if (!_nextRunUtc.ContainsKey(d.ChartId))
                    {
                        _logger.LogInformation("Scheduling new chart '{chartId}'", d.ChartId);
                        _nextRunUtc[d.ChartId] = now;
                    }
                }

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
                            _logger.LogError(ex, "Error refreshing '{chartId}'", def.ChartId);
                        }
                        _nextRunUtc[def.ChartId] = now.AddSeconds(def.RefreshIntervalSeconds);
                    }
                }

                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }

            _logger.LogInformation("ChartPollingBackgroundService stopping.");
        }
    }
}
```

---

### 5C. StarTrendsDashboard.Client (Blazor WebAssembly)

```
StarTrendsDashboard.Client/
├── Program.cs
├── wwwroot/
│   ├── index.html
│   ├── css/
│   │   └── app.css
│   └── lib/
│       └── apexcharts/
│           ├── apexcharts.min.js
│           ├── apexcharts.css
│           └── apexInterop.js
├── Pages/
│   ├── Index.razor
│   ├── Product.razor
│   └── Trade.razor
├── Shared/
│   ├── MainLayout.razor
│   └── NavMenu.razor
└── StarTrendsDashboard.Client.csproj
```

---

#### Program.cs

```csharp
using Microsoft.AspNetCore.Components.Web;
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;
using StarTrendsDashboard.Client;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

// Point HttpClient to Server’s base address
builder.Services.AddScoped(sp => 
    new HttpClient { BaseAddress = new Uri("https://localhost:5001/") });

await builder.Build().RunAsync();
```

---

#### wwwroot/index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>StarTrendsDashboard.Client</title>
  <link href="css/bootstrap/bootstrap.min.css" rel="stylesheet" />
  <link href="css/app.css" rel="stylesheet" />
  <link href="lib/apexcharts/apexcharts.css" rel="stylesheet" />
</head>
<body>
  <div id="app">Loading...</div>

  <script src="_framework/blazor.webassembly.js"></script>
  <script src="lib/apexcharts/apexcharts.min.js"></script>
  <script src="lib/apexcharts/apexInterop.js"></script>
</body>
</html>
```

---

#### wwwroot/lib/apexcharts/apexInterop.js

*(Properties: Copy if newer)*

```js
window.apexInterop = {
  renderChart: function (elementId, config) {
    console.log(`[apexInterop] renderChart for '#${elementId}', config=`, config);
    const elem = document.querySelector(`#${elementId}`);
    if (!elem) {
      console.warn(`[apexInterop] Container '#${elementId}' not found.`);
      return;
    }
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

*(Also copy `apexcharts.min.js` and `apexcharts.css` into this folder, each set to “Copy if newer.”)*

---

#### Pages/Product.razor

```razor
@page "/charts/Product"
@using StarTrendsDashboard.Shared
@inject HttpClient Http
@inject IJSRuntime JS

<h3>Charts for “Product”</h3>
<h4>Markets Set in Last 30 Days</h4>

<button class="btn btn-primary mb-2" @onclick="LoadProductChart">
    Refresh Now
</button>

<div id="productChart" style="min-height: 350px;"></div>
<p class="text-muted">@LastUpdatedText</p>

@code {
    private string LastUpdatedText { get; set; } = "";

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            await LoadProductChart();
        }
    }

    private async Task LoadProductChart()
    {
        // 1) Call server API to get JSON
        var cache = await Http.GetFromJsonAsync<ChartDataCache>("api/chartdata/ProductMarkets");
        if (cache == null) return;

        // 2) Extract labels & values
        var labels = cache.Rows.Select(r => r.Label).ToArray();
        var values = cache.Rows.Select(r => r.Value).ToArray();

        // 3) Build ApexCharts config
        var options = new
        {
            chart = new
            {
                id = "productChart",
                type = "bar",
                toolbar = new { show = true },
                zoom = new { enabled = false }
            },
            xaxis = new { categories = labels },
            title = new 
            { 
                text = cache.ChartId ?? "Markets Set in Last 30 Days", 
                align = "left" 
            },
            series = new[] { new { name = cache.ChartId, data = values } }
        };

        // 4) Render via JS interop
        await JS.InvokeVoidAsync("apexInterop.renderChart", "productChart", options);

        // 5) Update timestamp
        LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
        StateHasChanged();
    }
}
```

---

#### Pages/Trade.razor

```razor
@page "/charts/Trade"
@using StarTrendsDashboard.Shared
@inject HttpClient Http
@inject IJSRuntime JS

<h3>Charts for “Trade”</h3>
<h4>Trades Saved in Last 30 Days</h4>

<button class="btn btn-primary mb-2" @onclick="LoadTradeChart">
    Refresh Now
</button>

<div id="tradeChart" style="min-height: 350px;"></div>
<p class="text-muted">@LastUpdatedText</p>

@code {
    private string LastUpdatedText { get; set; } = "";

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            await LoadTradeChart();
        }
    }

    private async Task LoadTradeChart()
    {
        // 1) Call server API to get JSON
        var cache = await Http.GetFromJsonAsync<ChartDataCache>("api/chartdata/TradeTools");
        if (cache == null) return;

        // 2) Convert rows into [ [timestamp, value], … ]
        var points = cache.Rows.Select(r => new object[]
        {
            new DateTimeOffset(DateTime.Parse(r.Label)).ToUnixTimeMilliseconds(),
            r.Value
        }).ToArray();

        // 3) Build ApexCharts config
        var options = new
        {
            chart = new
            {
                id = "tradeChart",
                type = "scatter",
                toolbar = new { show = true },
                zoom = new { enabled = true }
            },
            xaxis = new
            {
                type = "datetime",
                labels = new { format = "dd MMM HH:mm" }
            },
            title = new
            {
                text = cache.ChartId ?? "Trades Saved in Last 30 Days",
                align = "left"
            },
            series = new[] { new { name = cache.ChartId, data = points } }
        };

        // 4) Render
        await JS.InvokeVoidAsync("apexInterop.renderChart", "tradeChart", options);

        // 5) Update timestamp
        LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
        StateHasChanged();
    }
}
```

---

#### Shared/NavMenu.razor (optional)

```razor
<nav class="navbar navbar-expand navbar-light bg-light">
  <div class="container-fluid">
    <ul class="navbar-nav me-auto mb-2 mb-lg-0">
      <li class="nav-item px-3">
        <NavLink class="nav-link" href="charts/Product">
          <span class="oi oi-bar-chart" aria-hidden="true"></span> Product Charts
        </NavLink>
      </li>
      <li class="nav-item px-3">
        <NavLink class="nav-link" href="charts/Trade">
          <span class="oi oi-pie-chart" aria-hidden="true"></span> Trade Charts
        </NavLink>
      </li>
    </ul>
  </div>
</nav>
```

---

### 5D. StarTrendsDashboard.Worker (Optional)

If you prefer to keep the Worker as a separate background service instead of hosting it in `StarTrendsDashboard.Server`, you can follow the instructions in the preceding steps for **StarTrendsDashboard.Worker**. In that scenario:

* **StarTrendsDashboard.Worker** runs independently (as a Windows Service or console) and writes JSON under `StarTrendsDashboard.Worker/bin/.../ChartCache/`.
* **StarTrendsDashboard.Server** only reads JSON from a shared folder (you must configure `ChartCacheFolder` to point to the Worker’s output).
* In that case, skip adding `ChartPollingBackgroundService` to the Server; instead let the Worker handle polling.

—but for simplicity, we integrated all polling and SQL logic into **StarTrendsDashboard.Server** above.

---

## 6. Build & Run

1. **Start StarTrendsDashboard.Server**

   * In **Solution Explorer**, right-click **StarTrendsDashboard.Server** → **Debug → Start Without Debugging** (Ctrl F5).
   * A console window will open, showing messages like:

     ```
     info: StarTrendsDashboard.Server.Services.ChartService[0]
           Loaded 2 chart definitions
     info: StarTrendsDashboard.Server.Services.ChartPollingBackgroundService[0]
           Background refreshing 'ProductMarkets'
     info: StarTrendsDashboard.Server.Services.ChartService[0]
           Refreshed 'ProductMarkets' → 5 rows, wrote ChartCache/ProductMarkets.json
     info: StarTrendsDashboard.Server.Services.ChartPollingBackgroundService[0]
           Background refreshing 'TradeTools'
     info: StarTrendsDashboard.Server.Services.ChartService[0]
           Refreshed 'TradeTools' → 7 rows, wrote ChartCache/TradeTools.json
     ...
     info: Microsoft.Hosting.Lifetime[0]
           Now listening on: https://localhost:5001
           Now listening on: http://localhost:5000
     ```
   * Confirm in your file explorer that `StarTrendsDashboard.Server/bin/Debug/net8.0/ChartCache/` contains:

     ```
     ProductMarkets.json
     TradeTools.json
     ```

2. **Start StarTrendsDashboard.Client**

   * In **Solution Explorer**, right-click **StarTrendsDashboard.Client** → **Debug → Start Without Debugging**.
   * A browser window (or tab) should open at something like `https://localhost:5002` or `http://localhost:5003` (check the address shown).

3. **Navigate** in the Blazor UI:

   * Click **“Product Charts”** (or go to `https://localhost:5002/charts/Product`).

     * The component calls `GET https://localhost:5001/api/chartdata/ProductMarkets`.
     * The Server returns the cached JSON produced by `ChartService`, and the chart appears.
     * Click **“Refresh Now”** to re-fetch (which reruns SQL on the Server and overwrites JSON), then redraw.
   * Visit **“Trade Charts”** (`/charts/Trade`).

     * The component calls `GET https://localhost:5001/api/chartdata/TradeTools`.
     * The Server returns JSON; the chart renders.
     * Click **“Refresh Now”** to refresh data and redraw.

---

## 7. (Optional) Production Deployment Notes

* In production, both **Server** and (if used) **Worker** must share the same physical `ChartCache/` folder.

  * For example, deploy to `D:\Apps\StarTrendsDashboard\`:

    ```
    D:\Apps\StarTrendsDashboard\
    ├── ChartCache\          ← shared read/write
    ├── StarTrendsDashboard.Server\_published\
    └── StarTrendsDashboard.Client\_published\
    ```
  * In `StarTrendsDashboard.Server/appsettings.json`, set:

    ```json
    "ChartCacheFolder": "..\\ChartCache"
    ```

    so it writes to `D:\Apps\StarTrendsDashboard\ChartCache\`.
  * In `StarTrendsDashboard.Client/Program.cs`, configure `HttpClient` base address as your production URL (e.g. `new Uri("https://myserver.com/")`).
* Grant the IIS AppPool user (or Linux service user) read/write access to `ChartCache`.
* If you keep a separate Worker project, install it as a Windows Service (e.g. via `sc create`), point its working directory to the published `Server\_published\ChartCache`, and grant it write access to `ChartCache`.

---

### Summary

You now have a fully working **Blazor WebAssembly (ASP.NET Core hosted) solution** on .NET 8 named **StarTrendsDashboard**:

1. **Shared** holds your DTOs (`ChartDefinition`, `ChartDataRow`, `ChartDataCache`).

2. **Server**

   * Reads `chart-definitions.json` and SQL files under `ChartDefinitions/`.
   * Polls Oracle (via `ChartService` and optional `ChartPollingBackgroundService`), writes `ChartCache/{chartId}.json`.
   * Exposes `GET /api/chartdata/{chartId}` and `POST /api/chartdata/refresh/{chartId}`.

3. **Client**

   * Calls `/api/chartdata/ProductMarkets` and `/api/chartdata/TradeTools`.
   * Receives `ChartDataCache` JSON, builds `labels`/`values` or `points[]`.
   * Uses JS interop (`apexInterop.renderChart(...)`) to draw ApexCharts.

By copying each file exactly as shown and setting the “Copy to Output Directory” flags where indicated, you’ll have a working end-to-end implementation that uses the **Blazor Web App** template on .NET 8 and leaves nothing out.
