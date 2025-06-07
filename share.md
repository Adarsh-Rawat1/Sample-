@using System.Linq
@using System.Net.Http.Json
@using StarTrendsDashboard.Shared
@inject HttpClient Http
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
    <div class="alert alert-danger">
      Error loading data. Try again.
    </div>
  }
  else if (hasNoData)
  {
    <div class="alert alert-warning">
      No data available.
    </div>
  }
  else
  {
    <div id="@elementId" style="min-height:300px"></div>
    <p class="text-muted">@LastUpdatedText</p>
  }
</div>

@code {
  [Parameter] public ChartDefinition Definition { get; set; } = default!;

  private bool isLoading;
  private bool hasNoData;
  private bool loadError;
  private string LastUpdatedText = "";
  private string elementId => $"chart_{Definition.ChartId}";

  protected override async Task OnInitializedAsync()
  {
    await Load();
  }

  private async Task Load()
  {
    isLoading = true;
    hasNoData = false;
    loadError = false;
    StateHasChanged();

    try
    {
      var cache = await Http.GetFromJsonAsync<ChartDataCache>($"api/chartdata/{Definition.ChartId}");
      if (cache == null || cache.Rows == null || cache.Rows.Count == 0)
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
      loadError = true;
    }
    finally
    {
      isLoading = false;
      StateHasChanged();
    }
  }

  private object BuildBarOptions(ChartDataCache cache)
  {
    var labels = cache.Rows.Select(r => r.Label).ToArray();
    var values = cache.Rows.Select(r => r.Value).ToArray();

    return new
    {
      chart   = new { id = elementId, type = "bar", toolbar = new { show = true }, zoom = new { enabled = false } },
      xaxis   = new { categories = labels },
      title   = new { text = Definition.Title, align = "left" },
      series  = new[] { new { name = Definition.ChartId, data = values } }
    };
  }

  private object BuildLineOptions(ChartDataCache cache)
  {
    var labels = cache.Rows.Select(r => r.Label).ToArray();
    var values = cache.Rows.Select(r => r.Value).ToArray();

    return new
    {
      chart   = new { id = elementId, type = "line", toolbar = new { show = true }, zoom = new { enabled = true } },
      xaxis   = new { categories = labels },
      title   = new { text = Definition.Title, align = "left" },
      series  = new[] { new { name = Definition.ChartId, data = values } }
    };
  }

  private object BuildScatterOptions(ChartDataCache cache)
  {
    var points = cache.Rows
      .Select(r => new object[] {
        DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(),
        r.Value
      })
      .ToArray();

    return new
    {
      chart   = new { id = elementId, type = "scatter", toolbar = new { show = true }, zoom = new { enabled = true } },
      xaxis   = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
      title   = new { text = Definition.Title, align = "left" },
      series  = new[] { new { name = Definition.ChartId, data = points } }
    };
  }
}
