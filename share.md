Below are detailed **Visual Studio (UI) steps**—no CLI—showing exactly how to build a three-project solution named **StarTrendsDashboard** that uses:

* **StarTrendsDashboard.Shared** (a class library for shared models)
* **StarTrendsDashboard.Worker** (a .NET Worker Service that polls Oracle and writes JSON)
* **StarTrendsDashboard.Web** (a Blazor Server App that injects the same service and renders ApexCharts)

By the end, you’ll have:

1. A **Shared** project containing your C# DTOs.
2. A **Worker** project that reads SQL files and writes `ChartCache/*.json` on a schedule.
3. A **Web** (Blazor Server) project that, on each page load or “Refresh Now” click, calls the same service to run SQL and draw a chart in the browser.

All code‐behind, folder names, file contents, and properties are specified below exactly. Simply follow each step in Visual Studio to create, configure, and wire up the solution.

---

## Prerequisites

* Visual Studio 2022 (or 2023) installed with:

  * **“.NET Desktop Development”** workload (for class libraries and Worker projects)
  * **“ASP.NET and web development”** (for Blazor)
  * **Oracle.ManagedDataAccess.Client** NuGet package available (we’ll install this in the Worker project).

* You have a valid Oracle connection string (for example, `User Id=MYUSER;Password=MYPASSWORD;Data Source=//myhost:1521/ORCLPDB`).

* You have downloaded the **ApexCharts** files (`apexcharts.min.js` and `apexcharts.css`) and know where to paste them.

---

## Step 1: Create a new Solution

1. Open **Visual Studio**.
2. Select **File → New → Project…**
3. In the “Create a new project” dialog, type **“Blank Solution”** into the search box.

   * Select **“Blank Solution”** (under “Visual Studio solutions”) and click **Next**.
4. **Solution name:** `StarTrendsDashboard`

   * **Location:** choose your desired folder (e.g. `C:\Projects\StarTrendsDashboard`)
   * **Solution name:** `StarTrendsDashboard`
   * Leave “Place solution and project in the same directory” **unchecked** (so you get a folder structure).
   * Click **Create**.

You now have a blank solution called `StarTrendsDashboard.sln`.

---

## Step 2: Add the Shared Class Library (StarTrendsDashboard.Shared)

1. In **Solution Explorer**, right-click on the solution node (`StarTrendsDashboard`) and choose **Add → New Project…**

2. In “Create a new project,” search for **“Class Library”**, then choose **Class Library (.NET)** (it should say “C#, .NET”).

   * Click **Next**.
   * **Project name:** `StarTrendsDashboard.Shared`
   * **Location:** should already be the solution folder (e.g. `C:\Projects\StarTrendsDashboard\StarTrendsDashboard.Shared`)
   * **Solution:** make sure it says **“Add to solution”** (instead of new solution).
   * Click **Create**.

3. Visual Studio will generate:

   ```
   StarTrendsDashboard.Shared/
     └── StarTrendsDashboard.Shared.csproj
     └── Class1.cs
   ```

4. In **Solution Explorer**, expand `StarTrendsDashboard.Shared`, right-click `Class1.cs` and select **Delete**. Confirm deletion.

5. Right-click **StarTrendsDashboard.Shared** → **Add → Class…**. Name it `ChartDefinition.cs` and click **Add**.

   * Replace its entire contents with:

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

6. Right-click **StarTrendsDashboard.Shared** → **Add → Class…**. Name it `ChartDataRow.cs` and click **Add**.

   * Replace its entire contents with:

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

7. Right-click **StarTrendsDashboard.Shared** → **Add → Class…**. Name it `ChartDataCache.cs` and click **Add**.

   * Replace its entire contents with:

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

8. **Build** the Shared project to confirm there are no errors:

   * Right-click **StarTrendsDashboard.Shared** → **Build**.

That completes the **Shared** project.

---

## Step 3: Add the Worker Service (StarTrendsDashboard.Worker)

1. In **Solution Explorer**, right-click the solution (`StarTrendsDashboard`) and choose **Add → New Project…**

2. Search for **“Worker Service”** and select **Worker Service (.NET)**. Click **Next**.

   * **Project name:** `StarTrendsDashboard.Worker`
   * **Location:** should be `C:\Projects\StarTrendsDashboard\StarTrendsDashboard.Worker` (same parent folder).
   * **Solution:** make sure **“Add to solution”** is selected.
   * Click **Create**.

3. Visual Studio will generate:

   ```
   StarTrendsDashboard.Worker/
     ├── Program.cs
     ├── Worker.cs      (auto-generated “Worker” class)
     └── StarTrendsDashboard.Worker.csproj
   ```

4. **Delete** the auto-generated `Worker.cs`:

   * In **Solution Explorer**, right-click `Worker.cs` under `StarTrendsDashboard.Worker` → **Delete**.

5. Add a reference to the Shared project:

   * Right-click **StarTrendsDashboard.Worker** → **Add → Project Reference…**
   * In the dialog, check **StarTrendsDashboard.Shared** and click **OK**.

6. In **Solution Explorer**, right-click **StarTrendsDashboard.Worker** → **Add → New Folder**, name it `ChartDefinitions`.

   * Right-click **ChartDefinitions** → **Add → New Item…**

     * Choose **JSON File**, name it `chart-definitions.json`, click **Add**.
     * Replace its contents with:

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

   * Right-click **ChartDefinitions** → **Add → New Folder**, name it `Queries`.

     * Right-click **Queries** → **Add → New Item…**

       * Choose **Text File**, name it `ProductMarkets.sql`, click **Add**.
       * Replace the new `ProductMarkets.sql` contents with:

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
     * Right-click **Queries** → **Add → New Item…**

       * Choose **Text File**, name it `TradeTools.sql`, click **Add**.
       * Replace `TradeTools.sql` contents with:

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

7. Still in **StarTrendsDashboard.Worker**, right-click the project → **Add → New Folder**, name it `ChartCache`.

   * This folder will hold the runtime JSON files.
   * Right-click **ChartCache** → **Properties** → in the Properties window set **“Copy to Output Directory”** to **“Copy always”**.

8. Right-click **StarTrendsDashboard.Worker** → **Add → New Folder**, name it `Services`. Under `Services`, add three new classes:

   ### 8.1 File: `IChartService.cs`

   * Right-click **Services** → **Add → Class…**, name it `IChartService.cs`.
   * Replace its contents with:

     ```csharp
     using StarTrendsDashboard.Shared;
     using System.Collections.Generic;
     using System.Threading.Tasks;

     namespace StarTrendsDashboard.Worker.Services
     {
         public interface IChartService
         {
             IReadOnlyList<ChartDefinition> GetAllDefinitions();
             Task<ChartDataCache> RefreshChartAsync(string chartId);
         }
     }
     ```

   ### 8.2 File: `ChartService.cs`

   * Right-click **Services** → **Add → Class…**, name it `ChartService.cs`.
   * Replace its contents with exactly:

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

     namespace StarTrendsDashboard.Worker.Services
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
                     // Return empty or partial results if SQL fails
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

   ### 8.3 File: `ChartPollingBackgroundService.cs`

   * Right-click **Services** → **Add → Class…**, name it `ChartPollingBackgroundService.cs`.
   * Replace its contents with:

     ```csharp
     using Microsoft.Extensions.Hosting;
     using Microsoft.Extensions.Logging;
     using StarTrendsDashboard.Shared;
     using System;
     using System.Collections.Generic;
     using System.Threading;
     using System.Threading.Tasks;

     namespace StarTrendsDashboard.Worker.Services
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

                 // At startup, schedule all known definitions for immediate run
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

                     // If a new definition appears, schedule it immediately
                     foreach (var d in defs)
                     {
                         if (!_nextRunUtc.ContainsKey(d.ChartId))
                         {
                             _logger.LogInformation("Scheduling new chart '{chartId}'", d.ChartId);
                             _nextRunUtc[d.ChartId] = now;
                         }
                     }

                     // Refresh any chart whose next‐run time has arrived
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
                             // Schedule next run
                             _nextRunUtc[def.ChartId] = now.AddSeconds(def.RefreshIntervalSeconds);
                         }
                     }

                     // Wait 30 seconds before checking again
                     await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
                 }

                 _logger.LogInformation("ChartPollingBackgroundService stopping.");
             }
         }
     }
     ```

9. Finally, **edit** `Program.cs` to wire up the service and DI:

   * In **Solution Explorer**, open `Program.cs` under `StarTrendsDashboard.Worker`. Replace its contents with:

     ```csharp
     using Microsoft.Extensions.DependencyInjection;
     using Microsoft.Extensions.Hosting;
     using StarTrendsDashboard.Worker.Services;

     IHost host = Host.CreateDefaultBuilder(args)
         .ConfigureServices((context, services) =>
         {
             // Register ChartService under IChartService
             services.AddSingleton<IChartService, ChartService>();

             // Register our background polling service
             services.AddHostedService<ChartPollingBackgroundService>();
         })
         .Build();

     await host.RunAsync();
     ```

10. **Install NuGet Package** for Oracle Managed Data Access:

    * Right-click **StarTrendsDashboard.Worker** → **Manage NuGet Packages…**
    * In the “Browse” tab, search for **Oracle.ManagedDataAccess.Core** (or **Oracle.ManagedDataAccess**) and install the latest stable version.

      * This ensures you can use `OracleConnection`, `OracleCommand`, etc.

11. **Set “Copy to Output Directory” for chart definitions and SQL**:

    * In **Solution Explorer**, expand **ChartDefinitions**, right-click `chart-definitions.json` → **Properties** → set **“Copy to Output Directory”** to **“Copy if newer”**.
    * Expand **ChartDefinitions/Queries**, right-click `ProductMarkets.sql` → **Properties** → **“Copy if newer”**.
    * Right-click `TradeTools.sql` → **Properties** → **“Copy if newer”**.
    * The folder **ChartCache** already has **“Copy always”** set from step 7.

12. **Build** the Worker project:

    * Right-click **StarTrendsDashboard.Worker** → **Build**.
    * Fix any errors (if any).

13. **Run** the Worker (in a separate Visual Studio instance or by clicking **▶ Debug → Start Without Debugging**):

    * You should see console output (in the “Worker Service console”) indicating:

      ```
      info: StarTrendsDashboard.Worker.Services.ChartService[0]
            Loaded 2 chart definitions
      info: StarTrendsDashboard.Worker.Services.ChartPollingBackgroundService[0]
            Background refreshing 'ProductMarkets'
      info: StarTrendsDashboard.Worker.Services.ChartService[0]
            Refreshed 'ProductMarkets' → X rows, wrote ChartCache/ProductMarkets.json
      info: StarTrendsDashboard.Worker.Services.ChartPollingBackgroundService[0]
            Background refreshing 'TradeTools'
      info: StarTrendsDashboard.Worker.Services.ChartService[0]
            Refreshed 'TradeTools' → Y rows, wrote ChartCache/TradeTools.json
      …
      ```
    * Under the folder `StarTrendsDashboard.Worker/bin/Debug/net8.0/ChartCache/` you should now see:

      ```
      ProductMarkets.json
      TradeTools.json
      ```
    * If you stop and restart the Worker, it will continually refresh on a 30-second loop (or whatever you configured).

---

## Step 4: Add the Blazor Server App (StarTrendsDashboard.Web)

1. In **Solution Explorer**, right-click the solution (`StarTrendsDashboard`) → **Add → New Project…**

2. Search for **“Blazor Server App”** and select **“Blazor Server App”** (it might say “Blazor Web App” with “Server” under “Interactive render mode”). Click **Next**.

   * **Project name:** `StarTrendsDashboard.Web`
   * **Location:** `C:\Projects\StarTrendsDashboard\StarTrendsDashboard.Web`
   * **Solution:** ensure it says **“Add to solution”**.
   * **Framework:** `.NET 8.0 (LTS)` (or whichever .NET version you used above)
   * **Authentication type:** **None**
   * **Configure for HTTPS:** checked (optional, recommended)
   * **Interactive render mode:** **Server**
   * **Interactivity location:** **Per page/component** (default)
   * Click **Create**.

3. Visual Studio will generate a standard Blazor Server folder structure:

   ```
   StarTrendsDashboard.Web/
   ├── _Imports.razor
   ├── App.razor
   ├── Program.cs
   ├── wwwroot/
   │   ├── css/
   │   │   ├── app.css
   │   │   └── bootstrap.min.css
   │   ├── favicon.ico
   │   ├── index.html
   │   └── js/
   └── Pages/
       ├── _Host.cshtml
       ├── Index.razor
       ├── Error.razor
       └── Shared/
   ```

4. **Add references** to Shared and Worker:

   * Right-click **StarTrendsDashboard.Web** → **Add → Project Reference…**
   * Check **StarTrendsDashboard.Shared** and **StarTrendsDashboard.Worker**, click **OK**.

5. Under **StarTrendsDashboard.Web** create these new folders (right-click project → **Add → New Folder**):

   ```
   StarTrendsDashboard.Web/
   ├── appsettings.json
   ├── Program.cs
   ├── Pages/
   │   ├── _Imports.razor
   │   ├── _Host.cshtml
   │   ├── Index.razor
   │   └── <we will add> Product.razor
   │   └── <we will add> Trade.razor
   └── wwwroot/
       ├── css/
       │   └── site.css
       └── lib/
           └── apexcharts/
               ├── apexcharts.min.js
               ├── apexcharts.css
               └── apexInterop.js
   ```

6. **Add** `appsettings.json` to the Web project:

   * Right-click **StarTrendsDashboard.Web** → **Add → New Item…** → choose **JSON File**, name it `appsettings.json`, click **Add**.
   * Replace its contents with:

     ```jsonc
     {
       "CacheFolder": "ChartCache"
     }
     ```
   * In **Solution Explorer**, right-click `appsettings.json` → **Properties** → set **“Copy to Output Directory”** to **“Copy if newer”**.

7. **Edit** `Program.cs` in the Web project:

   * Double-click `Program.cs` under `StarTrendsDashboard.Web`. Replace its contents with:

     ```csharp
     using StarTrendsDashboard.Worker.Services; // ChartService, IChartService

     var builder = WebApplication.CreateBuilder(args);

     // 1) Register ChartService so Blazor pages can call RefreshChartAsync()
     builder.Services.AddSingleton<IChartService, ChartService>();

     // (Optional) If you want an API controller later (e.g. /api/refresh), do:
     // builder.Services.AddControllers();

     builder.Services.AddRazorPages();
     builder.Services.AddServerSideBlazor();

     var app = builder.Build();

     app.UseStaticFiles(); 

     // If you add API controllers, uncomment:
     // app.MapControllers();

     app.MapBlazorHub();
     app.MapFallbackToPage("/_Host");

     app.Run();
     ```

8. **Add** ApexCharts assets:

   * Under **StarTrendsDashboard.Web/wwwroot**, right-click → **Add → New Folder**, name it `lib`.
   * Right-click **lib** → **Add → New Folder**, name it `apexcharts`.
   * Copy your local `apexcharts.min.js` into `StarTrendsDashboard.Web/wwwroot/lib/apexcharts/`.

     * In **Solution Explorer**, find `apexcharts.min.js`, right-click → **Properties** → set **“Copy to Output Directory”** → **“Copy if newer”**.
   * Copy `apexcharts.css` into the same folder (`lib/apexcharts/`), set **“Copy if newer”**.
   * In that same folder, right-click **apexcharts** subfolder → **Add → New Item…** → choose **JavaScript File**, name it `apexInterop.js`, click **Add**.

     * Replace its contents with:

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
     * In **Solution Explorer**, right-click `apexInterop.js` → **Properties** → set **“Copy to Output Directory”** → **“Copy if newer”**.

9. **Edit** `_Host.cshtml` to include the ApexCharts CSS/JS:

   * In **Solution Explorer**, under `Pages`, double-click `_Host.cshtml`. Modify it so that the `<head>` and `<body>` include exactly:

     ```html
     @page "/"
     @namespace StarTrendsDashboard.Web.Pages
     @addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
     @{
         Layout = null;
     }

     <!DOCTYPE html>
     <html lang="en">
     <head>
         <meta charset="utf-8" />
         <meta name="viewport" content="width=device-width, initial-scale=1.0" />
         <title>StarTrendsDashboard</title>
         <base href="~/" />

         <link href="css/site.css" rel="stylesheet" />
         <link href="lib/apexcharts/apexcharts.css" rel="stylesheet" />
     </head>
     <body>
         <app>
             <component type="typeof(App)" render-mode="ServerPrerendered" />
         </app>

         <script src="_framework/blazor.server.js"></script>
         <script src="lib/apexcharts/apexcharts.min.js"></script>
         <script src="lib/apexcharts/apexInterop.js"></script>
     </body>
     </html>
     ```

   * **Ensure**:

     * There is a `<link href="lib/apexcharts/apexcharts.css" …>`
     * There are `<script>` tags for `lib/apexcharts/apexcharts.min.js` and `lib/apexcharts/apexInterop.js` just before `</body>`.

10. **Add** the `Product.razor` and `Trade.razor` components:

    * Under **Pages**, right-click → **Add → Razor Component…**, name it `Product.razor`, click **Add**.

      * Replace its contents with:

        ```razor
        @page "/charts/Product"
        @using StarTrendsDashboard.Shared
        @inject IChartService ChartService
        @inject IJSRuntime JS

        <h3>Charts for “Product”</h3>
        <h4>Markets Set in Last 30 Days</h4>

        <button class="btn btn-primary mb-2" @onclick="RefreshProductChart">
            Refresh Now
        </button>

        <div id="productChart" style="min-height: 350px;"></div>
        <p class="text-muted">Last updated: @LastUpdatedText</p>

        @code {
            private string LastUpdatedText { get; set; } = "";

            protected override async Task OnAfterRenderAsync(bool firstRender)
            {
                if (firstRender)
                {
                    await RefreshProductChart();
                }
            }

            private async Task RefreshProductChart()
            {
                // 1) Run SQL and get cache
                ChartDataCache cache = await ChartService.RefreshChartAsync("ProductMarkets");

                // 2) Extract labels & values
                string[] labels = cache.Rows.Select(r => r.Label).ToArray();
                decimal[] values = cache.Rows.Select(r => r.Value).ToArray();

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
                LastUpdatedText = cache.LastUpdatedUtc.ToLocalTime().ToString("g");
                StateHasChanged();
            }
        }
        ```

    * Under **Pages**, right-click → **Add → Razor Component…**, name it `Trade.razor`, click **Add**.

      * Replace its contents with:

        ```razor
        @page "/charts/Trade"
        @using StarTrendsDashboard.Shared
        @inject IChartService ChartService
        @inject IJSRuntime JS

        <h3>Charts for “Trade”</h3>
        <h4>Trades Saved in Last 30 Days</h4>

        <button class="btn btn-primary mb-2" @onclick="RefreshTradeChart">
            Refresh Now
        </button>

        <div id="tradeChart" style="min-height: 350px;"></div>
        <p class="text-muted">Last updated: @LastUpdatedText</p>

        @code {
            private string LastUpdatedText { get; set; } = "";

            protected override async Task OnAfterRenderAsync(bool firstRender)
            {
                if (firstRender)
                {
                    await RefreshTradeChart();
                }
            }

            private async Task RefreshTradeChart()
            {
                // 1) Run SQL and get cache
                ChartDataCache cache = await ChartService.RefreshChartAsync("TradeTools");

                // 2) Convert rows: [ [timestamp, value], … ]
                var points = cache.Rows.Select(r =>
                    new object[]
                    {
                        new DateTimeOffset(DateTime.Parse(r.Label)).ToUnixTimeMilliseconds(),
                        r.Value
                    }
                ).ToArray();

                // 3) Build config for scatter/datetime
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
                    series = new[]
                    {
                        new { name = cache.ChartId, data = points }
                    }
                };

                // 4) Render
                await JS.InvokeVoidAsync("apexInterop.renderChart", "tradeChart", options);

                // 5) Update timestamp
                LastUpdatedText = cache.LastUpdatedUtc.ToLocalTime().ToString("g");
                StateHasChanged();
            }
        }
        ```

11. **Ensure** that your default routing includes links to these pages (optional). For example, open `Pages/Index.razor` and replace its contents with:

    ```razor
    @page "/"

    <h2>StarTrendsDashboard</h2>
    <ul>
      <li><a href="/charts/Product">Product Charts</a></li>
      <li><a href="/charts/Trade">Trade Charts</a></li>
    </ul>
    ```

    This simply provides a home page linking to `/charts/Product` and `/charts/Trade`.

12. **Build** the Web project:

    * Right-click **StarTrendsDashboard.Web** → **Build**.
    * Resolve any reference errors if they appear.

13. **Run** the Web project (in a separate Visual Studio instance or after starting the Worker):

    * Right-click **StarTrendsDashboard.Web** → **Debug → Start Without Debugging** (or press `Ctrl+F5`).
    * The console will show something like:

      ```
      Now listening on: https://localhost:5001
      Now listening on: http://localhost:5000
      ```
    * A browser window should open at `https://localhost:5001/`.
    * Click the “Product Charts” link (or manually navigate to `https://localhost:5001/charts/Product`).
      You will see a bar chart.
      Click **“Refresh Now”** to force an immediate SQL run (the Blazor component calls `ChartService.RefreshChartAsync("ProductMarkets")`), rewrite `ChartCache/ProductMarkets.json`, and redraw.
    * Navigate to `https://localhost:5001/charts/Trade` and confirm the scatter chart appears. Click **“Refresh Now”** to rerun that query and redraw.

---

## Step 5: Permissions & Deployment Notes

1. Locally, when you run **StarTrendsDashboard.Worker** from Visual Studio, it runs under your user account and writes to:

   ```
   StarTrendsDashboard.Worker/bin/Debug/net8.0/ChartCache/
   ```

   The Blazor Server app, when you run it, will also create and/or read from:

   ```
   StarTrendsDashboard.Web/bin/Debug/net8.0/ChartCache/
   ```

   because `ChartService` in the Web project uses the same `ChartCacheFolder` (“ChartCache”) setting in `appsettings.json`. For local debugging that is fine.

2. **Production deployment** (IIS or Linux):

   * Create a folder structure such as:

     ```
     D:\Apps\StarTrendsDashboard\
     ├── ChartCache\               ← shared read/write folder
     ├── StarTrendsDashboard.Worker\_published\
     └── StarTrendsDashboard.Web\_published\
     ```
   * In **StarTrendsDashboard.Worker**’s `appsettings.json`, set `"ChartCacheFolder": "..\\ChartCache"` (so it writes to `D:\Apps\StarTrendsDashboard\ChartCache`).
   * In **StarTrendsDashboard.Web**’s `appsettings.json`, set `"CacheFolder": "..\\ChartCache"`.
   * Grant the Windows Service account (under which you install the Worker) **write** access to `ChartCache\`.
   * Grant the IIS AppPool identity (if you host the Web app in IIS) **write** access to `ChartCache\` (or at least **read** if Blazor only needs to read—but since you’re calling `RefreshChartAsync` from Blazor Server, it also needs write).
   * Install the Worker by creating a Windows Service (e.g. via `sc create …`) or use a Linux daemon.
   * Deploy the Blazor Server app as an IIS website or reverse-proxy.
   * Both will share `D:\Apps\StarTrendsDashboard\ChartCache\`, so any JSON written by the Worker (or by the Web app’s “Refresh Now” click) is in one place.

---

## Recap of Project Names & Final Folder Layout

After following all of the above steps, your solution structure (in Visual Studio’s Solution Explorer) should look like this:

```
StarTrendsDashboard (solution)
├── StarTrendsDashboard.Shared (class library)
│   ├── ChartDefinition.cs
│   ├── ChartDataRow.cs
│   ├── ChartDataCache.cs
│   └── StarTrendsDashboard.Shared.csproj
│
├── StarTrendsDashboard.Worker (Worker Service)
│   ├── appsettings.json
│   ├── Program.cs
│   ├── ChartDefinitions/
│   │   ├── chart-definitions.json
│   │   └── Queries/
│   │       ├── ProductMarkets.sql
│   │       └── TradeTools.sql
│   ├── ChartCache/                    (Copy Always)
│   └── Services/
│       ├── IChartService.cs
│       ├── ChartService.cs
│       └── ChartPollingBackgroundService.cs
│   └── StarTrendsDashboard.Worker.csproj
│
└── StarTrendsDashboard.Web (Blazor Server App)
    ├── appsettings.json
    ├── Program.cs
    ├── Pages/
    │   ├── _Imports.razor
    │   ├── _Host.cshtml
    │   ├── Index.razor
    │   ├── Product.razor
    │   └── Trade.razor
    ├── wwwroot/
    │   ├── css/
    │   │   └── site.css
    │   └── lib/
    │       └── apexcharts/
    │           ├── apexcharts.min.js
    │           ├── apexcharts.css
    │           └── apexInterop.js
    └── StarTrendsDashboard.Web.csproj
```

After carefully copying each file’s contents as shown, you can **build** each project individually:

1. Build **Shared** (no errors).
2. Build **Worker** (no errors).
3. Build **Web** (no errors).

Then **run**:

* **Worker** first: it will load definitions and create `ChartCache/ProductMarkets.json` & `ChartCache/TradeTools.json`.
* **Web** second: browse to `/charts/Product` and `/charts/Trade` to see live, refreshable charts.

That completes the **end-to-end, Blazor Server App (UI-only) implementation**, with no CLI commands required other than Visual Studio’s normal “Build” and “Run” operations.
