```
@using System.Linq
@using System.Net.Http.Json
@using Microsoft.JSInterop
@using StarTrendsDashboard.Shared
@inject HttpClient Http
@inject IChartService ChartService
@inject IJSRuntime JS

<div class="mb-5">
  <h4>@Definition.Title</h4>

  <!-- This button calls the Refresh() method below -->
  <button class="btn btn-outline-primary mb-2"
          @onclick="Refresh"
          disabled="@isRefreshing">
    @if (isRefreshing)
    {
      <span>Loading…</span>
    }
    else
    {
      <span>Refresh</span>
    }
  </button>

  @if (isRefreshing && !_hasRenderedCache)
  {
    <div class="spinner-border text-primary" role="status">
      <span class="visually-hidden">Loading…</span>
    </div>
  }
  else if (loadError)
  {
    <div class="alert alert-danger">Error loading data.</div>
  }
  else if (hasNoData && !_hasRenderedCache)
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

  // State flags
  bool isRefreshing;
  bool loadError;
  bool hasNoData;
  bool _hasRenderedCache;
  bool _needsRender;
  object? _pendingOptions;

  string LastUpdatedText = "";
  string ElementId        => $"chart_{Definition.ChartId}";

  // 1) On init, render any existing JSON cache immediately
  protected override async Task OnInitializedAsync()
  {
    await LoadCacheAsync();
    // 2) Then start the real refresh in background
    _ = LoadDataAsync();
  }

  // Handler for the Refresh button
  private async Task Refresh()
    => await LoadDataAsync();

  private async Task LoadCacheAsync()
  {
    try
    {
      var cache = await Http.GetFromJsonAsync<ChartDataCache>(
        $"ChartCache/{Definition.ChartId}.json"
      );

      if (cache?.Rows?.Any() ?? false)
      {
        _pendingOptions   = BuildOptions(cache);
        _needsRender      = true;
        LastUpdatedText   = $"Cached: {cache.LastUpdatedUtc.ToLocalTime():g}";
        _hasRenderedCache = true;
        StateHasChanged();
      }
    }
    catch
    {
      // no cache yet, ignore
    }
  }

  private async Task LoadDataAsync()
  {
    isRefreshing = true;
    loadError    = false;
    hasNoData    = false;
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
        _pendingOptions = BuildOptions(cache);
        _needsRender    = true;
        LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
      }
    }
    catch
    {
      loadError = true;
    }
    finally
    {
      isRefreshing = false;
      StateHasChanged();
    }
  }

  protected override async Task OnAfterRenderAsync(bool firstRender)
  {
    if (_needsRender && _pendingOptions is not null)
    {
      try
      {
        await JS.InvokeVoidAsync("apexInterop.renderChart", ElementId, _pendingOptions);
      }
      catch
      {
        // prerender or timing issue; will retry on next render
      }
      _needsRender = false;
    }
  }

  private object BuildOptions(ChartDataCache c) =>
    Definition.ChartType.ToLower() switch
    {
      "line"    => BuildLine(c),
      "scatter" => BuildScatter(c),
      _         => BuildBar(c)
    };

  private object BuildBar(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "bar", toolbar = new { show = true } },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    title  = new { text = Definition.Title, align = "left" },
    series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  private object BuildLine(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "line", toolbar = new { show = true } },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    title  = new { text = Definition.Title, align = "left" },
    series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  private object BuildScatter(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "scatter", toolbar = new { show = true } },
    xaxis  = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
    title  = new { text = Definition.Title, align = "left" },
    series = new[]
    {
      new
      {
        name = Definition.ChartId,
        data = c.Rows.Select(r =>
          new object[] { DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(), r.Value }
        ).ToArray()
      }
    }
  };
}

```
