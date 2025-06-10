```
@using StarTrendsDashboard.Shared
@rendermode InteractiveServer
@using StarTrendsDashboard.Services
@inject IChartService ChartService
@inject IJSRuntime JS

<div class="mb-5">
  <div class="d-flex justify-content-between align-items-center mb-2">
    <h4 class="m-0">@Definition.Title</h4>
    <div class="d-flex align-items-center">
      <button class="btn btn-outline-primary me-3"
              @onclick="() => RefreshAsync(force: true)"
              disabled="@isLoading">
        @(isLoading ? "Loading…" : "Refresh")
      </button>
      @if (!string.IsNullOrEmpty(LastUpdatedText))
      {
        <small class="text-muted">@LastUpdatedText</small>
      }
    </div>
  </div>

  @if (isLoading)
  {
    <div class="spinner-border text-primary" role="status">
      <span class="visually-hidden">Loading…</span>
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
    <div id="@ElementId" style="height:350px; width:100%;"></div>
  }
</div>

@code {
  [Parameter] public ChartDefinition Definition { get; set; } = default!;

  private bool isLoading, hasNoData, loadError, needsRender;
  private object? chartOptions;
  private string LastUpdatedText { get; set; } = string.Empty;
  private string ElementId => $"chart_{Definition.ChartId}";

  protected override async Task OnInitializedAsync()
    => await RefreshAsync(force: false);

  private async Task RefreshAsync(bool force)
  {
    isLoading = true; loadError = false; hasNoData = false; needsRender = false;
    StateHasChanged();

    try
    {
      var cache = force
        ? await ChartService.RefreshChartAsync(Definition.ChartId)
        : await ChartService.RefreshChartIfNeededAsync(Definition.ChartId);

      if (cache.Rows?.Count > 0)
      {
        chartOptions = Definition.ChartType.ToLower() switch
        {
          "bar"     => BuildBarOptions(cache),
          "line"    => BuildLineOptions(cache),
          "scatter" => BuildScatterOptions(cache),
          _         => BuildBarOptions(cache)
        };

        LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
        needsRender = true;
      }
      else hasNoData = true;
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
    if (needsRender && chartOptions is not null)
    {
      await JS.InvokeVoidAsync("apexInterop.renderChart", ElementId, chartOptions);
      needsRender = false;
    }
  }

  private object BuildBarOptions(ChartDataCache c) => new
  {
    chart = new
    {
      id = ElementId,
      type = "bar",
      height = 350,
      toolbar = new { show = true },
      zoom    = new { enabled = false }
    },
    plotOptions = new
    {
      bar = new { borderRadius = 4 }
    },
    dataLabels = new { enabled = true },
    title    = new { text = Definition.Title, align = "left", style = new { fontSize = "16px" } },
    subtitle = new { text = c.Summary ?? "", align = "left", style = new { fontSize = "14px" } },

    xaxis = new
    {
      categories = c.Rows.Select(r => r.Label).ToArray(),
      title      = new { text = c.XLabel, style = new { fontSize = "14px" } },
      labels     = new { style = new { fontSize = "12px" } }
    },
    yaxis = new
    {
      title  = new { text = c.YLabel, style = new { fontSize = "14px" } },
      labels = new { style = new { fontSize = "12px" } }
    },
    series = new[]
    {
      new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() }
    }
  };

  private object BuildLineOptions(ChartDataCache c) => new
  {
    chart = new
    {
      id = ElementId,
      type = "line",
      height = 350,
      toolbar = new { show = true },
      zoom    = new { enabled = true }
    },
    dataLabels = new { enabled = true },
    title    = new { text = Definition.Title, align = "left", style = new { fontSize = "16px" } },
    subtitle = new { text = c.Summary ?? "", align = "left", style = new { fontSize = "14px" } },

    xaxis = new
    {
      categories = c.Rows.Select(r => r.Label).ToArray(),
      title      = new { text = c.XLabel, style = new { fontSize = "14px" } },
      labels     = new { style = new { fontSize = "12px" } }
    },
    yaxis = new
    {
      title  = new { text = c.YLabel, style = new { fontSize = "14px" } },
      labels = new { style = new { fontSize = "12px" } }
    },
    series = new[]
    {
      new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() }
    }
  };

  private object BuildScatterOptions(ChartDataCache c) => new
  {
    chart = new
    {
      id = ElementId,
      type = "scatter",
      height = 350,
      toolbar = new { show = true },
      zoom    = new { enabled = true }
    },
    dataLabels = new { enabled = true },
    title    = new { text = Definition.Title, align = "left", style = new { fontSize = "16px" } },
    subtitle = new { text = c.Summary ?? "", align = "left", style = new { fontSize = "14px" } },

    xaxis = new
    {
      type    = "datetime",
      title   = new { text = c.XLabel, style = new { fontSize = "14px" } },
      labels  = new { format = "dd MMM HH:mm", style = new { fontSize = "12px" } }
    },
    yaxis = new
    {
      title  = new { text = c.YLabel, style = new { fontSize = "14px" } },
      labels = new { style = new { fontSize = "12px" } }
    },
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
