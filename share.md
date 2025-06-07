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

  private bool isLoading, hasNoData, loadError;
  private bool _needsRender;
  private object? _pendingOptions;
  private string LastUpdatedText = "";
  private string elementId => $"chart_{Definition.ChartId}";

  protected override async Task OnInitializedAsync()
  {
    await LoadData();
  }

  private async Task Refresh()
  {
    await LoadData();
  }

  private async Task LoadData()
  {
    isLoading = true;
    loadError = false;
    hasNoData = false;
    _needsRender = false;       // reset until we have new data
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
        // Build the right ApexCharts options based on ChartType 
        _pendingOptions = Definition.ChartType.ToLower() switch
        {
          "bar"     => BuildBarOptions(cache),
          "line"    => BuildLineOptions(cache),
          "scatter" => BuildScatterOptions(cache),
          _         => BuildBarOptions(cache)
        };

        LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
        _needsRender = true;    // signal OnAfterRenderAsync to draw the chart
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

  // This runs after each render. On the first render after data arrives, we invoke JS.
  protected override async Task OnAfterRenderAsync(bool firstRender)
  {
    if (_needsRender && _pendingOptions is not null)
    {
      await JS.InvokeVoidAsync("apexInterop.renderChart", elementId, _pendingOptions);
      _needsRender = false;  // ensure we only render once per data load
    }
  }

  private object BuildBarOptions(ChartDataCache c) => new
  {
    chart  = new { id = elementId, type = "bar", toolbar = new { show = true }, zoom = new { enabled = false } },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    title  = new { text = Definition.Title, align = "left" },
    series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  private object BuildLineOptions(ChartDataCache c) => new
  {
    chart  = new { id = elementId, type = "line", toolbar = new { show = true }, zoom = new { enabled = true } },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    title  = new { text = Definition.Title, align = "left" },
    series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  private object BuildScatterOptions(ChartDataCache c) => new
  {
    chart  = new { id = elementId, type = "scatter", toolbar = new { show = true }, zoom = new { enabled = true } },
    xaxis  = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
    title  = new { text = Definition.Title, align = "left" },
    series = new[] {
      new {
        name = Definition.ChartId,
        data = c.Rows
                .Select(r => new object[] {
                   DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(),
                   r.Value
                })
                .ToArray()
      }
    }
  };
}
