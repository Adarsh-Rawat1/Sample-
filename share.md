```

@using System.Linq
@using StarTrendsDashboard.Shared
@inject IChartService ChartService
@inject IJSRuntime JS

<div class="mb-5">
  <h4>@Definition.Title</h4>

  <button class="btn btn-outline-primary mb-2"
          @onclick="Refresh"
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
      <span class="visually-hidden">Loading…</span>
    </div>
  }
  else if (loadError)
  {
    <div class="alert alert-danger">Error loading data.</div>
  }
  else if (hasNoData)
  {
    <div class="alert alert-warning">No data available.</div>
  }
  else
  {
    <div id="@ElementId" style="min-height:350px"></div>
    <p class="text-muted">@LastUpdatedText</p>
  }
</div>

@code {
  [Parameter] public ChartDefinition Definition { get; set; } = default!;

  bool isLoading, loadError, hasNoData;
  bool _needsRender;
  object? _pendingOptions;
  string LastUpdatedText = "";
  string ElementId => $"chart_{Definition.ChartId}";

  protected override async Task OnInitializedAsync()
    => await LoadDataAsync();

  private async Task Refresh()
    => await LoadDataAsync();

  private async Task LoadDataAsync()
  {
    isLoading = true;
    loadError = false;
    hasNoData = false;
    _needsRender = false;
    StateHasChanged();

    try
    {
      var cache = await ChartService.RefreshChartAsync(Definition.ChartId);

      if (cache.Rows == null || cache.Rows.Count == 0)
      {
        hasNoData = true;
      }
      else
      {
        // Build chart options based on type
        _pendingOptions = Definition.ChartType.ToLower() switch
        {
          "bar"     => BuildBarOptions(cache),
          "line"    => BuildLineOptions(cache),
          "scatter" => BuildScatterOptions(cache),
          _         => BuildBarOptions(cache)
        };

        LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
        _needsRender = true;
      }
    }
    catch
    {
      loadError = true;
    }
    finally
    {
      isLoading = false;
      StateHasChanged();
    }
  }

  protected override async Task OnAfterRenderAsync(bool firstRender)
  {
    if (_needsRender && _pendingOptions is not null)
    {
      try
      {
        await JS.InvokeVoidAsync("console.log",
          $"[ChartBlock] calling apexInterop.renderChart('{ElementId}')");
        await JS.InvokeVoidAsync("apexInterop.renderChart", ElementId, _pendingOptions);
        _needsRender = false;
      }
      catch
      {
        // prerender or other error: ignore & retry next render
      }
    }
  }

  private object BuildBarOptions(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "bar", toolbar = new { show = true } },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    title  = new { text = Definition.Title, align = "left" },
    series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  private object BuildLineOptions(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "line", toolbar = new { show = true } },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    title  = new { text = Definition.Title, align = "left" },
    series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  private object BuildScatterOptions(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "scatter", toolbar = new { show = true } },
    xaxis  = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
    title  = new { text = Definition.Title, align = "left" },
    series = new[]
    {
      new
      {
        name = Definition.ChartId,
        data = c.Rows
                .Select(r => new object[]
                {
                  DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(),
                  r.Value
                })
                .ToArray()
      }
    }
  };
}

```
