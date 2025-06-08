Below is a **minimal** drop-in that will at least show **whatever JSON is already in `ChartCache/`** as an ApexCharts graph—no SQL, no refresh button—so you can verify the plumbing is correct. It uses your Blazor Server service to **serve** that folder as static files, then fetches the JSON via `HttpClient`, and finally renders it once.

---

## 1) Serve `ChartCache/` as static files

In **Program.cs**, add a second `UseStaticFiles(...)` **after** the default one:

```csharp
using Microsoft.Extensions.FileProviders;
using StarTrendsDashboard.Services;

var builder = WebApplication.CreateBuilder(args);

// Blazor services, ChartService registrations, etc.
builder.Services
    .AddRazorComponents()
    .AddInteractiveServerComponents()
    .AddHeadOutlet();
builder.Services.AddSingleton<IChartService, ChartService>();
builder.Services.AddHostedService<ChartPollingBackgroundService>();

var app = builder.Build();

// 1) Default wwwroot
app.UseStaticFiles();

// 2) Serve ChartCache folder at URL path "/ChartCache"
app.UseStaticFiles(new StaticFileOptions {
    FileProvider = new PhysicalFileProvider(
        Path.Combine(app.Environment.ContentRootPath, "ChartCache")
    ),
    RequestPath = "/ChartCache"
});

app.UseRouting();

app.MapRazorComponents<App>()
   .AddInteractiveServerRenderMode();

app.Run();
```

This makes any file placed in `ChartCache/` available at `https://<your-host>/ChartCache/{chartId}.json`.

---

## 2) Minimal `ChartBlock.razor`

Create or overwrite **`Components/Pages/ChartBlock.razor`** with exactly this:

```razor
@using System.Linq
@using System.Net.Http.Json
@inject HttpClient Http
@inject IJSRuntime JS

<div id="@ElementId" style="min-height:350px"></div>

@code {
  [Parameter] public ChartDefinition Definition { get; set; } = default!;

  bool _hasRendered;
  ChartDataCache? _cache;

  string ElementId => $"chart_{Definition.ChartId}";

  protected override async Task OnAfterRenderAsync(bool firstRender)
  {
    if (firstRender && !_hasRendered)
    {
      _hasRendered = true;

      // 1) Load existing JSON from the static /ChartCache endpoint
      _cache = await Http.GetFromJsonAsync<ChartDataCache>(
        $"/ChartCache/{Definition.ChartId}.json"
      );

      if (_cache?.Rows?.Any() ?? false)
      {
        // 2) Build the ApexCharts options
        var opts = Definition.ChartType.ToLower() switch
        {
          "line"    => BuildLine(_cache),
          "scatter" => BuildScatter(_cache),
          _         => BuildBar(_cache)
        };

        // 3) Render the chart
        await JS.InvokeVoidAsync("apexInterop.renderChart", ElementId, opts);
      }
    }
  }

  object BuildBar(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "bar" },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    series = new[] { new { name = Definition.Title, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  object BuildLine(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "line" },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    series = new[] { new { name = Definition.Title, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  object BuildScatter(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "scatter" },
    xaxis  = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
    series = new[]
    {
      new {
        name = Definition.Title,
        data = c.Rows.Select(r =>
          new object[] {
            DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(),
            r.Value
          }).ToArray()
      }
    }
  };
}
```

### 3) Usage in your category page

Make sure your `CategoryViewer.razor` loops:

```razor
@page "/charts/{category}"
@inject IChartService ChartService

@foreach (var def in ChartService.GetAllDefinitions().Where(d => d.Page == category))
{
  <ChartBlock Definition="def" />
}
```

---

### 4) Verify

1. Build & run.
2. Place a valid `{chartId}.json` in `ChartCache/` (the same JSON your service writes).
3. Browse to `/charts/Product` or `/charts/Trade`.
4. You should immediately see a bar/line/scatter chart rendered from that JSON.

Once this minimal path is proven, you can layer back your refresh button, polling, and error handling on top of it.
