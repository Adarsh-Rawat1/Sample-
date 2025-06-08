```

@using System.Linq
@using System.Net.Http.Json
@inject HttpClient Http
@inject IJSRuntime JS

<div id="@ElementId" style="min-height:350px"></div>

@code {
  [Parameter] public ChartDefinition Definition { get; set; } = default!;

  bool _hasRendered;
  ChartDataCache? _cache;

  string ElementId => $"chart_{Definition.ChartId}";

  protected override async Task OnAfterRenderAsync(bool firstRender)
  {
    if (firstRender && !_hasRendered)
    {
      _hasRendered = true;

      // 1) load existing JSON cache
      _cache = await Http.GetFromJsonAsync<ChartDataCache>(
        $"ChartCache/{Definition.ChartId}.json"
      );

      if (_cache?.Rows?.Any() ?? false)
      {
        // 2) build options
        var opts = Definition.ChartType.ToLower() switch
        {
          "line"    => BuildLine(_cache),
          "scatter" => BuildScatter(_cache),
          _         => BuildBar(_cache)
        };

        // 3) render chart
        await JS.InvokeVoidAsync("apexInterop.renderChart", ElementId, opts);
      }
    }
  }

  object BuildBar(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "bar" },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    series = new[] { new { name = Definition.Title, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  object BuildLine(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "line" },
    xaxis  = new { categories = c.Rows.Select(r => r.Label).ToArray() },
    series = new[] { new { name = Definition.Title, data = c.Rows.Select(r => r.Value).ToArray() } }
  };

  object BuildScatter(ChartDataCache c) => new
  {
    chart  = new { id = ElementId, type = "scatter" },
    xaxis  = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
    series = new[]
    {
      new
      {
        name = Definition.Title,
        data = c.Rows.Select(r => new object[]
        {
          DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(),
          r.Value
        }).ToArray()
      }
    }
  };
}


```
