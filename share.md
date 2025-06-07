To skip any HTTP round‐trip and just render charts directly on the server, have your `<ChartBlock>` call your `IChartService` instead of `HttpClient`. This way you’ll see the graphs (or “No data”) as soon as the service returns, with no network fetch at all.

---

## 1. Update `ChartBlock.razor`

Replace your existing file with the version below:

```razor
@using StarTrendsDashboard.Shared
@using Microsoft.JSInterop
@inject IChartService ChartService
@inject IJSRuntime JS

<div class="mb-5">
  <h4>@Definition.Title</h4>

  <button class="btn btn-outline-primary mb-2"
          @onclick="Load"
          disabled="@isLoading">
    @if (isLoading)
    {
      <span>Loading…</span>
    }
    else
    {
      <span>Refresh</span>
    }
  </button>

  @if (isLoading)
  {
    <div class="spinner-border text-primary" role="status">
      <span class="visually-hidden">Loading...</span>
    </div>
  }
  else if (loadError)
  {
    <div class="alert alert-danger">Error loading data. Try again.</div>
  }
  else if (hasNoData)
  {
    <div class="alert alert-warning">No data available.</div>
  }
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
  string elementId   => $"chart_{Definition.ChartId}";

  protected override async Task OnInitializedAsync()
    => await Load();

  private async Task Load()
  {
    isLoading = true;
    hasNoData  = false;
    loadError  = false;
    StateHasChanged();

    try
    {
      // Directly invoke your ChartService (no HTTP)
      var cache = await ChartService.RefreshChartAsync(Definition.ChartId);

      if (cache.Rows == null || cache.Rows.Count == 0)
      {
        hasNoData = true;
      }
      else
      {
        object options = Definition.ChartType.ToLower() switch
        {
          "bar"     => BuildBarOptions(cache),
          "line"    => BuildLineOptions(cache),
          "scatter" => BuildScatterOptions(cache),
          _         => BuildBarOptions(cache)
        };

        await JS.InvokeVoidAsync("apexInterop.renderChart", elementId, options);
        LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
      }
    }
    catch
    {
      // You can also log the exception here to diagnose SQL errors
      loadError = true;
    }
    finally
    {
      isLoading = false;
      StateHasChanged();
    }
  }

  private object BuildBarOptions(ChartDataCache c)
    => new {
        chart  = new { id = elementId, type = "bar", toolbar = new { show = true }, zoom = new { enabled = false } },
        xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
        title  = new { text = Definition.Title, align = "left" },
        series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
      };

  private object BuildLineOptions(ChartDataCache c)
    => new {
        chart  = new { id = elementId, type = "line", toolbar = new { show = true }, zoom = new { enabled = true } },
        xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
        title  = new { text = Definition.Title, align = "left" },
        series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
      };

  private object BuildScatterOptions(ChartDataCache c)
    => new {
        chart  = new { id = elementId, type = "scatter", toolbar = new { show = true }, zoom = new { enabled = true } },
        xaxis  = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
        title  = new { text = Definition.Title, align = "left" },
        series = new[] {
          new {
            name = Definition.ChartId,
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

### What changed

* **Removed** `@inject HttpClient Http`.
* **Added** `@inject IChartService ChartService`.
* **Load()** now calls `ChartService.RefreshChartAsync(...)` directly.

---

## 2. Ensure `ChartService` is wired

In **Program.cs**, make sure you have:

```csharp
builder.Services.AddSingleton<IChartService, ChartService>();
// optional background poller
builder.Services.AddHostedService<ChartPollingBackgroundService>();
```

so that Blazor can inject it into your component.

---

## 3. Verify your SQL & JSON

* **chart-definitions.json** must list each chart (with correct `SqlFile` name and `ChartType`).
* **ChartDefinitions/Queries** must contain those `.sql` files, and they should return at least one row.
* If your SQL takes a while, the spinner will show until it finishes.

You can also temporarily log inside the `catch`:

```csharp
catch (Exception ex)
{
    Console.WriteLine($"Error loading {Definition.ChartId}: {ex}");
    loadError = true;
}
```

so you can see any Oracle/SQL exceptions on your server console.

---

### Result

With this change, no HTTP calls are made, so you’ll never need to watch the Network tab. As soon as each chart’s SQL returns data, the graph will render in the page. If there truly is no data, you’ll see the “No data available” alert; if there’s a real error (bad SQL, missing file, Oracle down), you’ll see the red “Error loading data” box.
