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
    isLoading = true;
    loadError = false;
    hasNoData = false;
    needsRender = false;
    StateHasChanged();

    try
    {
      var cache = force
        ? await ChartService.RefreshChartAsync(Definition.ChartId)
        : await ChartService.RefreshChartIfNeededAsync(Definition.ChartId);

      if (cache.Data?.Count > 0)
      {
        chartOptions = Definition.ChartType.ToLower() switch
        {
          "bar"     => BuildBarOptions(cache),
          "line"    => BuildLineOptions(cache),
          "scatter" => BuildScatterOptions(cache),
          "heatmap" => BuildHeatmapOptions(cache),
          _          => BuildBarOptions(cache)
        };

        LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
        needsRender = true;
      }
      else
      {
        hasNoData = true;
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
    if (needsRender && chartOptions is not null)
    {
      await JS.InvokeVoidAsync(
        Definition.Renderer == "plotly3d" ? "plotlyInterop.render3d" : "apexInterop.renderChart", 
        ElementId, chartOptions);
      needsRender = false;
    }
  }

  private object BuildBarOptions(ChartDataCache c) => new
  {
    chart = new { id = ElementId, type = "bar", height = 450, toolbar = new { show = true }, zoom = new { enabled = false } },
    plotOptions = new { bar = new { borderRadius = 4 } },
    dataLabels = new { enabled = true },
    title = new { text = c.Summary ?? Definition.Title, align = "left", style = new { fontSize = "16px" } },
    xaxis = new { categories = c.Data.Select(d => (string)d[0]).ToArray(), title = new { text = c.XLabel, style = new { fontSize = "14px" } }, labels = new { style = new { fontSize = "12px" } } },
    yaxis = new { title = new { text = c.YLabel, style = new { fontSize = "14px" } }, labels = new { style = new { fontSize = "12px" } } },
    series = new[] { new { name = Definition.ChartId, data = c.Data.Select(d => Convert.ToDecimal(d[1])).ToArray() } }
  };

  private object BuildLineOptions(ChartDataCache c)
  {
    var timestamps = c.Data.Select(d => DateTime.Parse((string)d[0]).ToUniversalTime().ToUnixTimeMilliseconds()).ToArray();
    var values = c.Data.Select(d => Convert.ToDecimal(d[1])).ToArray();
    var seriesList = new List<object>
    {
      new {
        name = Definition.Title,
        type = "line",
        data = timestamps.Zip(values, (x, y) => new object[] { x, y }).ToArray(),
        stroke = new { width = 2, curve = "smooth" },
        markers = new { size = 4, colors = new[] { "#008FFB" } }
      }
    };
    if (c.EmaSeries is { Count: > 0 } ema)
    {
      seriesList.Add(new {
        name = "EMA (6)",
        type = "line",
        data = timestamps.Zip(ema, (x, e) => new object[] { x, e }).ToArray(),
        stroke = new { width = 1, dashArray = 4 },
        markers = new { size = 0 }
      });
    }

    return new
    {
      chart = new { id = ElementId, type = "line", height = 350, toolbar = new { show = true }, zoom = new { enabled = true } },
      dataLabels = new { enabled = false },
      xaxis = new { type = "datetime", tickAmount = 8, labels = new { format = "dd MMM HH:mm", rotate = -45, hideOverlappingLabels = true, datetimeUTC = false, style = new { fontSize = "12px" } }, title = new { text = c.XLabel, style = new { fontSize = "14px" } } },
      yaxis = new { title = new { text = c.YLabel, style = new { fontSize = "14px" } }, labels = new { style = new { fontSize = "12px" } } },
      series = seriesList.ToArray()
    };
  }

  private object BuildHeatmapOptions(ChartDataCache c)
  {
    var xCats = c.Data.Select(d => (string)d[0]).Distinct().ToArray();
    var yCats = c.Data.Select(d => (string)d[1]).Distinct().ToArray();
    var values = c.Data.Select(d => Convert.ToDecimal(d[2])).ToArray();
    var minVal = (double)values.Min();
    var maxVal = (double)values.Max();
    var series = yCats.Select(y => new {
      name = y,
      data = xCats.Select(x => new {
        x,
        y,
        value = (double)(c.Data.FirstOrDefault(d => (string)d[0] == x && (string)d[1] == y)?[2] ?? 0m)
      }).ToArray()
    }).ToArray();
    var chartHeight = Math.Max(350, yCats.Length * 30);

    return new
    {
      chart = new { type = "heatmap", height = chartHeight, toolbar = new { show = true } },
      plotOptions = new {
        heatmap = new {
          shadeIntensity = 0.5,
          radius = 0,
          colorScale = new { ranges = new[] { new { from = minVal, to = maxVal, name = "", color = "#008FFB" } } }
        }
      },
      dataLabels = new { enabled = false },
      stroke = new { width = 0 },
      xaxis = new { type = "category", categories = xCats, labels = new { rotate = -45, style = new { fontSize = "12px" } } },
      yaxis = new { type = "category", labels = new { style = new { fontSize = "12px" } } },
      series = series
    };
  }

  private object BuildScatterOptions(ChartDataCache c)
  {
    var points = c.Data.Select(d => new object[] {
      DateTime.Parse((string)d[0]).ToUniversalTime().ToUnixTimeMilliseconds(),
      Convert.ToDecimal(d[1])
    }).ToArray();

    return new
    {
      chart = new
      {
        id     = ElementId,
        type   = "scatter",
        height = 350,
        toolbar = new { show = true },
        zoom    = new { enabled = true }
      },
      markers = new { size = 6, colors = new[] { "#FF4560" } },
      dataLabels = new { enabled = false },
      xaxis = new
      {
        type = "datetime",
        tickAmount = 8,
        labels = new
        {
          format = "dd MMM HH:mm",
          rotate = -45,
          hideOverlappingLabels = true,
          datetimeUTC = false,
          style = new { fontSize = "12px" }
        },
        title = new { text = c.XLabel, style = new { fontSize = "14px" } }
      },
      yaxis = new
      {
        title = new { text = c.YLabel, style = new { fontSize = "14px" } },
        labels = new { style = new { fontSize = "12px" } }
      },
      series = new[]
      {
        new { name = Definition.Title, data = points }
      }
    };
  }
}

```
