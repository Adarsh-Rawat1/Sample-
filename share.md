```

@using System.Linq
@using System.Net.Http.Json
@inject HttpClient Http
@inject IChartService ChartService
@inject IJSRuntime JS

<div class="mb-5">
  <h4>@Definition.Title</h4>
  <button class="btn btn-outline-primary mb-2" @onclick="Refresh" disabled="@isRefreshing">
    @(isRefreshing ? "Loading…" : "Refresh")
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
  [Parameter] public ChartDefinition Definition { get; set; }
  bool isRefreshing, loadError, hasNoData, _hasRenderedCache, _needsRender;
  object? _pendingOptions;
  string LastUpdatedText = "";
  string ElementId => $"chart_{Definition.ChartId}";

  protected override async Task OnInitializedAsync()
  {
    await LoadCacheAsync();
    _ = LoadDataAsync();
  }

  async Task Refresh() => await LoadDataAsync();

  async Task LoadCacheAsync()
  {
    try
    {
      var cache = await Http.GetFromJsonAsync<ChartDataCache>($"ChartCache/{Definition.ChartId}.json");
      if (cache?.Rows?.Any() ?? false)
      {
        _pendingOptions   = BuildOptions(cache);
        _needsRender      = true;
        LastUpdatedText   = $"Cached: {cache.LastUpdatedUtc.ToLocalTime():g}";
        _hasRenderedCache = true;
        StateHasChanged();
      }
    }
    catch { }
  }

  async Task LoadDataAsync()
  {
    isRefreshing = true; loadError = false; hasNoData = false;
    StateHasChanged();
    try
    {
      var cache = await ChartService.RefreshChartAsync(Definition.ChartId);
      if (cache.Rows?.Any() ?? false)
      {
        _pendingOptions = BuildOptions(cache);
        _needsRender    = true;
        LastUpdatedText = $"Last updated: {cache.LastUpdatedUtc.ToLocalTime():g}";
      }
      else hasNoData = true;
    }
    catch { loadError = true; }
    finally { isRefreshing = false; StateHasChanged(); }
  }

  protected override async Task OnAfterRenderAsync(bool firstRender)
  {
    if (_needsRender && _pendingOptions is not null)
    {
      try
      {
        await JS.InvokeVoidAsync("apexInterop.renderChart", ElementId, _pendingOptions);
      }
      catch { }
      _needsRender = false;
    }
  }

  object BuildOptions(ChartDataCache c) =>
    Definition.ChartType.ToLower() switch
    {
      "line"    => new { chart   = new { id = ElementId, type = "line" },   xaxis = new { categories = c.Rows.Select(r => r.Label).ToArray() }, series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } } },
      "scatter" => new { chart   = new { id = ElementId, type = "scatter" }, xaxis = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } }, series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => new object[] { DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(), r.Value }).ToArray() } } },
      _         => new { chart   = new { id = ElementId, type = "bar" },      xaxis = new { categories = c.Rows.Select(r => r.Label).ToArray() }, series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } } }
    };
}



```
