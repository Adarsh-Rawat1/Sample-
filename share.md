The red squiggle is simply telling you **“the ApexCharts.Blazor wrapper isn’t referenced.”**
Right now `@using ApexCharts` and the `<ApexChart …>` tag helper belong to that wrapper NuGet, so the compiler can’t find them.

You have **two easy fixes — pick one**:

| Option                                                                                         | What you do                                                                                                                                                             | When to choose                                   |
| ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| **① Add the free NuGet wrapper** (**fastest**)                                                 | \`\`\`bash                                                                                                                                                              |                                                  |
| dotnet add package ApexCharts.Blazor                                                           |                                                                                                                                                                         |                                                  |
| \`\`\`<br>Re-build → the error disappears; keep your existing `ChartHost.razor` exactly as-is. | You’re fine adding one small open-source package (no cost).                                                                                                             |                                                  |
| **② Drop the wrapper & call JS directly**                                                      | Remove `@using ApexCharts` and the `<ApexChart>` tag; instead render a `<div id="chart-@Meta.Id">` and invoke ApexCharts via **IJSRuntime**. (Sample code shown below.) | If you absolutely cannot add any NuGet packages. |

---

### Quick path ① – install the wrapper (recommended)

```bash
# in project folder
dotnet add package ApexCharts.Blazor
dotnet build
```

Nothing else changes. The wrapper just calls the global `apexcharts.min.js` you already placed in *wwwroot*.

---

### Path ② – pure JS-Interop (only if you really need to)

1. **Create** `wwwroot/js/chartInterop.js`

   ```js
   export function render(meta, payload) {
     const opts = {
       chart: { type: meta.graph.type, height: 400 },
       series: meta.graph.y.map(y => ({
         name: y,
         data: payload.rows.map(r => r[y])
       })),
       xaxis: { categories: payload.rows.map(r => r[meta.graph.x]) }
     };
     const el = document.getElementById(`chart-${meta.id}`);
     if (el.__chart) { el.__chart.updateOptions(opts); return; }
     el.__chart = new ApexCharts(el, opts);
     el.__chart.render();
   }
   ```

2. **Reference** it after `apexcharts.min.js` in *App.razor*

   ```html
   <script src="apexcharts/apexcharts.min.js"></script>
   <script src="js/chartInterop.js"></script>
   ```

3. **Rewrite** `ChartHost.razor`

   ```razor
   @inject IJSRuntime JS
   @inject HttpClient Http
   <div class="card mb-4 p-3">
     <h5>@Meta.Title <button class="btn btn-sm" @onclick="Refresh">🔄</button></h5>
     <div id="chart-@Meta.Id"></div>
   </div>

   @code {
       [Parameter] public ChartMeta Meta { get; set; } = default!;
       protected override async Task OnAfterRenderAsync(bool first)
       {
           if (first) await LoadAndRender();
       }
       private async Task Refresh()
       {
           await Http.PostAsync($"/api/chartdata/{Meta.Id}/refresh", null);
           await LoadAndRender();
       }
       private async Task LoadAndRender()
       {
           var payload = await Http.GetFromJsonAsync<JsonElement>(
               $"/chartcache/{Meta.Id}.json?nocache={DateTime.UtcNow.Ticks}");
           await JS.InvokeVoidAsync("render", Meta, payload);
       }
   }
   ```

The component now renders through your own JS file, no wrapper needed.

---

### TL;DR

* **Simplest**: add the NuGet → error gone, keep code.
* **If packages are forbidden**: replace the wrapper with the JS-interop snippet above.












@using StarTrends.Shared
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <base href="/" />

    <!-- existing styles -->
    <link rel="stylesheet" href="bootstrap/bootstrap.min.css" />
    <link rel="stylesheet" href="app.css" />

    <!-- ① ApexCharts CSS  -->
    <link rel="stylesheet" href="apexcharts/apexcharts.css" />

    <HeadOutlet />          <!-- leave this where it is -->
</head>

<body>
    <Routes />

    <!-- ② ApexCharts JS – before the Blazor framework script -->
    <script src="apexcharts/apexcharts.min.js"></script>

    <!-- if you made your own helper, add it just after -->
    <!-- <script src="js/chartInterop.js"></script> -->

    <script src="_framework/blazor.web.js"></script>
</body>
</html>



Below is a **clean “green-field” recipe** that will spin up a Blazor Server (.NET 8) dashboard with exactly **two pages** and **one chart each**, wired to Oracle through a connection string in *appsettings.json*.
Everything else (more pages / charts / refresh rules) can be added later by dropping files.

---

## 1  Create the solution & install packages

```bash
dotnet new blazorserver -n StarTrends
cd StarTrends

# Oracle client + ApexCharts Blazor wrapper + (optional) Markdown
dotnet add package Oracle.ManagedDataAccess.Core
dotnet add package ApexCharts.Blazor
dotnet add package Markdig          # only if you’ll use _intro.md banners
```

---

## 2  Configure the DB connection

```jsonc
// appsettings.json
{
  "ConnectionStrings": {
    "StarOracle": "User Id=STAR_RO;Password=SECRET;Data Source=HOST/Service"
  },
  "Logging": { /* default stuff */ }
}
```

---

## 3  Create the query folders & files

```
Queries/
├─ Product/
│  ├─ 01_source_systems.sql
└─ Trade/
   ├─ 02_trades_per_minute.sql
charts.catalog.json
```

`Queries/Product/01_source_systems.sql`

```sql
/* Source System,Contracts booked,Trade */
SELECT COALESCE(s.dsc,'Star') AS "Source system",
       COUNT(1)               AS "Deals Booked"
  FROM (SELECT c.mrr_typ_cod, c.inp_dt, c.src_sys_cod
          FROM star_contract PARTITION(product_oth) c) t
  LEFT JOIN star_src_sys s ON t.src_sys_cod = s.src_sys_cod
 WHERE t.inp_dt > TRUNC(SYSDATE) - 7
   AND t.mrr_typ_cod IN (0, 1, 6)
 GROUP BY s.dsc
 ORDER BY "Deals Booked" DESC;
```

`Queries/Trade/02_trades_per_minute.sql`

```sql
/* Minute,Trades saved,Trade */
SELECT TO_CHAR(l.upd_timestamp,'dd-Mon-yyyy HH24:MI') AS "Minute",
       COUNT(1)                                       AS "Trades saved"
  FROM star_link_timestamp l
  JOIN star_contract PARTITION(product_oth) c
    ON l.chd_no = c.con_no
 WHERE l.upd_timestamp >= TRUNC(SYSDATE) - 5
   AND l.upd_timestamp <  TRUNC(SYSDATE)
 GROUP BY TO_CHAR(l.upd_timestamp,'dd-Mon-yyyy HH24:MI')
 ORDER BY 1;
```

`charts.catalog.json`

```jsonc
[
  {
    "id": 1,
    "sqlPath": "Product/01_source_systems.sql",
    "title": "OTC contracts booked by source system (last 7 days)",
    "page": "Product",
    "order": 1,
    "graph": { "type": "bar", "x": "Source system", "y": ["Deals Booked"] },
    "pollSeconds": 21600,
    "isEnabled": true
  },
  {
    "id": 2,
    "sqlPath": "Trade/02_trades_per_minute.sql",
    "title": "OTC trades saved per minute (last 5 days)",
    "page": "Trade",
    "order": 1,
    "graph": { "type": "scatter", "x": "Minute", "y": ["Trades saved"] },
    "pollSeconds": 21600,
    "isEnabled": true
  }
]
```

---

## 4  Core C# building blocks

### 4.1 `ChartMeta.cs`

```csharp
public sealed class ChartMeta
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public string Page { get; set; } = "";
    public int Order { get; set; }
    public string SqlPath { get; set; } = "";
    public string SqlText { get; set; } = "";
    public GraphInfo Graph { get; set; } = new();
    public int PollSeconds { get; set; } = 21600;
    public bool IsEnabled { get; set; } = true;
    public DateTime LastRunUtc { get; set; } = DateTime.MinValue;
    public sealed class GraphInfo
    {
        public string Type { get; set; } = "bar";
        public string X { get; set; } = "";
        public string[] Y { get; set; } = Array.Empty<string>();
    }
}
```

### 4.2 `ChartCatalog` (singleton)

```csharp
public interface IChartCatalog
{
    IReadOnlyList<string> Pages { get; }
    IReadOnlyList<ChartMeta> GetCharts(string page);
    ChartMeta? Get(int id);
}

public sealed class ChartCatalog : IChartCatalog, IDisposable
{
    private readonly ConcurrentDictionary<int,ChartMeta> _charts = new();
    private readonly FileSystemWatcher _fs;

    public ChartCatalog(IWebHostEnvironment env)
    {
        var root = Path.Combine(env.ContentRootPath, "Queries");
        var catalogPath = Path.Combine(root, "charts.catalog.json");
        LoadCatalog(root, catalogPath);

        _fs = new FileSystemWatcher(root) { IncludeSubdirectories = false };
        _fs.Filter = "charts.catalog.json";
        _fs.Changed += (_,__) => LoadCatalog(root, catalogPath);
        _fs.EnableRaisingEvents = true;
    }

    private void LoadCatalog(string root, string catalogPath)
    {
        var list = JsonSerializer.Deserialize<List<ChartMeta>>
                   (File.ReadAllText(catalogPath))!;
        foreach (var meta in list)
        {
            var sqlFull = Path.Combine(root, meta.SqlPath);
            if (!File.Exists(sqlFull)) continue;
            meta.SqlText = File.ReadAllText(sqlFull);
            _charts[meta.Id] = meta;
        }
    }

    public IReadOnlyList<string> Pages =>
        _charts.Values.Select(c => c.Page).Distinct().OrderBy(p => p).ToList();
    public IReadOnlyList<ChartMeta> GetCharts(string page) =>
        _charts.Values.Where(c => c.Page == page && c.IsEnabled)
                      .OrderBy(c => c.Order).ToList();
    public ChartMeta? Get(int id) => _charts.TryGetValue(id, out var m) ? m : null;
    public void Dispose() => _fs.Dispose();
}
```

### 4.3 `OracleDataService`

```csharp
public interface IOracleDataService
{
    Task<List<Dictionary<string,object>>> QueryAsync(string sql,
                                                     CancellationToken ct=default);
}
public sealed class OracleDataService : IOracleDataService
{
    private readonly string _cs;
    public OracleDataService(IConfiguration cfg)
        => _cs = cfg.GetConnectionString("StarOracle");
    public async Task<List<Dictionary<string,object>>> QueryAsync(string sql,
                                                                  CancellationToken ct=default)
    {
        await using var con = new OracleConnection(_cs);
        await con.OpenAsync(ct);
        await using var cmd = new OracleCommand(sql, con);
        await using var rdr = await cmd.ExecuteReaderAsync(ct);
        var rows = new List<Dictionary<string,object>>();
        while (await rdr.ReadAsync(ct))
        {
            var row = new Dictionary<string,object>(rdr.FieldCount);
            for (var i=0;i<rdr.FieldCount;i++) row[rdr.GetName(i)] = rdr.GetValue(i);
            rows.Add(row);
        }
        return rows;
    }
}
```

### 4.4 `QueryRefreshWorker`

```csharp
public sealed class QueryRefreshWorker : BackgroundService
{
    private readonly IChartCatalog _catalog;
    private readonly IOracleDataService _db;
    private readonly IWebHostEnvironment _env;
    private readonly ILogger<QueryRefreshWorker> _log;
    public QueryRefreshWorker(IChartCatalog c, IOracleDataService db,
                              IWebHostEnvironment env, ILogger<QueryRefreshWorker> log)
        { _catalog=c; _db=db; _env=env; _log=log; }

    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        Directory.CreateDirectory(Path.Combine(_env.WebRootPath,"chartcache"));
        while (!stop.IsCancellationRequested)
        {
            foreach (var meta in _catalog.GetCharts(page:null!)
                                          .SelectMany(_ => _catalog.Pages
                                              .SelectMany(p=>_catalog.GetCharts(p))))
            {
                if (DateTime.UtcNow - meta.LastRunUtc <
                    TimeSpan.FromSeconds(meta.PollSeconds)) continue;

                try
                {
                    var rows = await _db.QueryAsync(meta.SqlText, stop);
                    var payload = JsonSerializer.Serialize(new {
                        generatedUtc = DateTime.UtcNow,
                        rows
                    });
                    var outPath = Path.Combine(_env.WebRootPath,
                                "chartcache", $"{meta.Id}.json");
                    await File.WriteAllTextAsync(outPath, payload, stop);
                    meta.LastRunUtc = DateTime.UtcNow;
                }
                catch(Exception ex) { _log.LogError(ex,"refresh {id}",meta.Id); }
            }
            await Task.Delay(TimeSpan.FromMinutes(1), stop);
        }
    }
}
```

---

## 5  Program.cs registrations

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorComponents().AddInteractiveServerComponents();
builder.Services.AddSingleton<IChartCatalog, ChartCatalog>();
builder.Services.AddSingleton<IOracleDataService, OracleDataService>();
builder.Services.AddHostedService<QueryRefreshWorker>();

var app = builder.Build();
app.MapRazorComponents<App>().AddInteractiveServerRenderMode();
app.Run();
```

---

## 6  UI glue

### 6.1 Sidebar (update `Shared/NavMenu.razor`)

```razor
@inject IChartCatalog Catalog
<ul class="nav flex-column px-2">
  <li><NavLink href="">Home</NavLink></li>
  @foreach (var page in Catalog.Pages)
  {
      <li><NavLink href=$"/dashboard/{page}">@page</NavLink></li>
  }
</ul>
```

### 6.2 Dynamic dashboard page `Pages/Dashboard.razor`

```razor
@page "/dashboard/{Page}"
@inject IChartCatalog Catalog

<h3 class="mb-3">@Page</h3>

@if (Charts.Any())
{
    @foreach (var meta in Charts)
    {
        <ChartHost Meta="meta" />
    }
}
else { <p>No charts defined.</p> }

@code {
    [Parameter] public string Page { get; set; } = "";
    IReadOnlyList<ChartMeta> Charts => Catalog.GetCharts(Page);
}
```

### 6.3 `Components/ChartHost.razor`

```razor
@using ApexCharts
@inject HttpClient Http

<div class="card mb-4 p-3">
  <h5>@Meta.Title
      <button class="btn btn-sm" @onclick="Refresh">🔄</button>
  </h5>
  <ApexChart @ref="_chart" TItem="IDictionary<string,object>"
             Width="100%" Height="400" />
</div>

@code {
    [Parameter] public ChartMeta Meta { get; set; } = default!;
    private ApexChart<IDictionary<string,object>> _chart = default!;
    protected override async Task OnAfterRenderAsync(bool first)
    {
        if (first) await LoadAndRender();
    }
    private async Task Refresh()
    {
        await Http.PostAsync($"/api/chartdata/{Meta.Id}/refresh", null);
        await LoadAndRender();
    }
    private async Task LoadAndRender()
    {
        var data = await Http.GetFromJsonAsync<Payload>(
            $"/chartcache/{Meta.Id}.json?nocache={DateTime.UtcNow.Ticks}");
        var series = Meta.Graph.Y.Select(y => new Series<IDictionary<string,object>>
        {
            Name = y,
            Data = data.Rows.Select(r => r[y])
        });
        var opts = new ApexChartOptions<IDictionary<string,object>>();
        opts.Xaxis = new XAxis<IDictionary<string,object>> { Type = "category",
                   Categories = data.Rows.Select(r => r[Meta.Graph.X]).ToList() };
        await _chart.UpdateSeries(series, true, opts);
    }
    private sealed record Payload(DateTime generatedUtc,
        List<Dictionary<string,object>> Rows);
}
```

*(Add a minimal POST endpoint `/api/chartdata/{id}/refresh` that just sets a `ConcurrentDictionary<int,bool> _pending` flag the worker checks—omitted for brevity.)*

---

## 7  Run

```bash
dotnet run
```

* **Navigate** to `/dashboard/Product` → bar chart.
* **Navigate** to `/dashboard/Trade` → scatter plot.
* **Sidebar** shows “Product” and “Trade” automatically.
* **Refresh 🔄** beside a chart forces an immediate update; otherwise, the worker does it every 6 hours.

That’s the full skeleton—add more `.sql` files and catalog entries, or drop `_intro.md` in any folder, and the dashboard grows without more C#.
