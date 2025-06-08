```
@page "/charts/{category}"
@using StarTrendsDashboard.Shared
@inject IChartService ChartService

@code {
    [Parameter] public string category { get; set; } = "";
}

@if (!string.IsNullOrWhiteSpace(category))
{
    foreach (var def in ChartService
                         .GetAllDefinitions()
                         .Where(d => d.Page.Equals(category, StringComparison.OrdinalIgnoreCase)))
    {
        <section class="mb-5">
            <h4>@def.Title</h4>
            <div id="chart_@def.ChartId" style="min-height:350px"></div>
            <script>
              (function() {
                var chartId = '@def.ChartId';
                fetch('/ChartCache/' + chartId + '.json')
                  .then(function(response) {
                    if (!response.ok) throw new Error('No cache');
                    return response.json();
                  })
                  .then(function(cache) {
                    if (!cache.Rows || cache.Rows.length === 0) return;
                    var options = {
                      chart: { id: 'chart_' + chartId, type: '@def.ChartType' },
                      xaxis: { categories: cache.Rows.map(function(r) { return r.Label; }) },
                      series: [{ name: cache.ChartId, data: cache.Rows.map(function(r) { return r.Value; }) }]
                    };
                    var container = document.getElementById('chart_' + chartId);
                    ApexCharts.getChartByID('chart_' + chartId)?.destroy();
                    new ApexCharts(container, options).render();
                  })
                  .catch(function(err) {
                    console.warn('Chart load failed for', chartId, err);
                  });
              })();
            </script>
        </section>
    }
}
else
{
    <div class="alert alert-warning">No category specified.</div>
}


```
