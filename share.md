<nav class="navbar navbar-expand-lg navbar-light bg-light shadow-sm mb-4">
  <div class="container-fluid">
    <a class="navbar-brand" href="/">StarTrends</a>
    <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navContent">
      <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="navContent">
      <ul class="navbar-nav me-auto mb-2 mb-lg-0">
        <li class="nav-item">
          <NavLink class="nav-link" href="/charts/product" Match="NavLinkMatch.Prefix">Product</NavLink>
        </li>
        <li class="nav-item">
          <NavLink class="nav-link" href="/charts/trade" Match="NavLinkMatch.Prefix">Trade</NavLink>
        </li>
      </ul>
    </div>
  </div>
</nav>






@using System.Net.Http.Json
@inject HttpClient Http
@inject IJSRuntime JS

<div class="card mb-4 shadow-sm">
  <div class="card-header d-flex justify-content-between align-items-center">
    <h5 class="mb-0">@Definition.Title</h5>
    <div>
      <button class="btn btn-sm btn-outline-primary" @onclick="OnRefresh">Refresh</button>
      @if (lastUpdated != null)
      {
        <small class="text-muted ms-2">Updated: @lastUpdated</small>
      }
    </div>
  </div>
  <div class="card-body p-0">
    @if (isLoading)
    {
      <div class="d-flex justify-content-center align-items-center" style="height:300px;">Loading...</div>
    }
    else if (hasNoData)
    {
      <div class="d-flex justify-content-center align-items-center text-muted" style="height:300px;">No data available</div>
    }
    else
    {
      <div id="@ElementId" style="height:300px;"></div>
    }
  </div>
</div>

@code {
    [Parameter] public ChartDefinition Definition { get; set; }

    bool isLoading;
    bool hasNoData;
    string? lastUpdated;
    ChartDataCache? cache;
    string ElementId => $"chart_{Definition.ChartId}";

    async Task OnRefresh()
    {
        cache = null;
        lastUpdated = null;
        hasNoData = false;
        await LoadAndRender();
    }

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
            await LoadAndRender();
    }

    async Task LoadAndRender()
    {
        isLoading = true;
        StateHasChanged();

        var resp = await Http.GetAsync($"ChartCache/{Definition.ChartId}.json");
        if (!resp.IsSuccessStatusCode)
        {
            isLoading = false;
            hasNoData = true;
            StateHasChanged();
            return;
        }

        if (resp.Content.Headers.LastModified.HasValue)
            lastUpdated = resp.Content.Headers.LastModified.Value
                .LocalDateTime.ToString("dd MMM yyyy HH:mm");

        cache = await resp.Content.ReadFromJsonAsync<ChartDataCache>();
        isLoading = false;
        hasNoData = cache?.Rows?.Any() != true;
        StateHasChanged();

        if (!hasNoData)
        {
            var opts = Definition.ChartType.ToLower() switch
            {
                "line" => new {
                    chart  = new { id = ElementId, type = "line", height = 300 },
                    xaxis  = new { categories = cache.Rows.Select(r => r.Label).ToArray() },
                    series = new[] { new { name = Definition.Title, data = cache.Rows.Select(r => r.Value).ToArray() } }
                },
                "scatter" => new {
                    chart  = new { id = ElementId, type = "scatter", height = 300 },
                    xaxis  = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
                    series = new[] {
                      new {
                        name = Definition.Title,
                        data = cache.Rows.Select(r =>
                          new object[] {
                            DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(),
                            r.Value
                          }).ToArray()
                      }
                    }
                },
                _ => new {
                    chart  = new { id = ElementId, type = "bar", height = 300 },
                    xaxis  = new { categories = cache.Rows.Select(r => r.Label).ToArray() },
                    series = new[] { new { name = Definition.Title, data = cache.Rows.Select(r => r.Value).ToArray() } }
                }
            };

            await JS.InvokeVoidAsync("apexInterop.renderChart", ElementId, opts);
        }
    }
}




