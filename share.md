@using StarTrendsDashboard.Shared
@using StarTrendsDashboard.Services
@inject IChartService ChartService
@inject IJSRuntime JS

<div class="card mb-4 shadow-sm">
    <div class="card-header d-flex justify-content-between align-items-center">
        <h5 class="card-title mb-0">@Definition.Title</h5>
        <button class="btn btn-outline-primary btn-sm"
                @onclick="RefreshAsync"
                disabled="@isLoading">
            @if (isLoading)
            {
                <span class="spinner-border spinner-border-sm me-1"></span> Loading…
            }
            else
            {
                <i class="bi bi-arrow-clockwise me-1"></i> Refresh
            }
        </button>
    </div>

    <div class="card-body">
        @if (isLoading)
        {
            <div class="d-flex justify-content-center my-5">
                <div class="spinner-border text-primary" role="status">
                    <span class="visually-hidden">Loading…</span>
                </div>
            </div>
        }
        else if (loadError)
        {
            <div class="alert alert-danger text-center">
                <i class="bi bi-exclamation-triangle-fill me-2"></i>
                Error loading data. Please try again.
            </div>
        }
        else if (hasNoData)
        {
            <div class="alert alert-warning text-center">
                <i class="bi bi-inbox-fill me-2"></i>
                No data available for this chart.
            </div>
        }
        else
        {
            <div id="@ElementId" style="min-height:350px;"></div>
            <p class="text-end text-muted small mt-3">@LastUpdatedText</p>
        }
    </div>
</div>

@code {
    [Parameter] public ChartDefinition Definition { get; set; } = default!;

    private bool isLoading;
    private bool hasNoData;
    private bool loadError;
    private bool needsRender;
    private object? chartOptions;
    private string LastUpdatedText { get; set; } = string.Empty;
    private string ElementId => $"chart_{Definition.ChartId}";

    protected override async Task OnInitializedAsync() => await RefreshAsync();

    private async Task RefreshAsync()
    {
        isLoading   = true;
        loadError   = false;
        hasNoData   = false;
        needsRender = false;
        StateHasChanged();

        try
        {
            var cache = await ChartService.RefreshChartAsync(Definition.ChartId);

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
                needsRender     = true;
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
            await JS.InvokeVoidAsync("apexInterop.renderChart", ElementId, chartOptions);
            needsRender = false;
        }
    }

    private object BuildBarOptions(ChartDataCache c) => new
    {
        chart = new { id = ElementId, type = "bar", toolbar = new { show = true }, zoom = new { enabled = false } },
        xaxis = new { categories = c.Rows.Select(r => r.Label).ToArray() },
        title = new { text = Definition.Title, align = "left" },
        series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
    };

    private object BuildLineOptions(ChartDataCache c) => new
    {
        chart = new { id = ElementId, type = "line", toolbar = new { show = true }, zoom = new { enabled = true } },
        xaxis = new { categories = c.Rows.Select(r => r.Label).ToArray() },
        title = new { text = Definition.Title, align = "left" },
        series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
    };

    private object BuildScatterOptions(ChartDataCache c) => new
    {
        chart = new { id = ElementId, type = "scatter", toolbar = new { show = true }, zoom = new { enabled = true } },
        xaxis = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
        title = new { text = Definition.Title, align = "left" },
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












@using StarTrendsDashboard.Services
@inject IChartService ChartService

<div class="d-flex">
  <aside class="bg-white shadow-sm" style="width: 240px; min-height: 100vh;">
    <div class="p-3 border-bottom">
      <a class="text-decoration-none d-flex align-items-center" href="/">
        <span class="fs-4 fw-bold">StarTrends</span>
      </a>
    </div>
    <nav class="nav nav-pills flex-column px-2 py-3">
      @foreach (var cat in ChartService
                            .GetAllDefinitions()
                            .Select(d => d.Page)
                            .Distinct())
      {
        <NavLink class="nav-link mb-1 rounded" 
                 href="@($"/charts/{cat}")" 
                 Match="NavLinkMatch.Prefix">
          @cat
        </NavLink>
      }
    </nav>
  </aside>

  <main class="flex-grow-1 p-4">
    @Body
  </main>
</div>







