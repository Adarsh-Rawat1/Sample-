```
@page "/charts/{category}"
@using StarTrendsDashboard.Shared
@inject IChartService ChartService

@code {
    [Parameter]
    public string Category { get; set; } = string.Empty;

    // we'll hold only the definitions for this page
    private List<ChartDefinition> Definitions { get; set; } = new();

    protected override async Task OnParametersSetAsync()
    {
        // assume you add this async method to your service:
        // Task<IEnumerable<ChartDefinition>> GetAllDefinitionsAsync();
        var allDefs = await ChartService.GetAllDefinitionsAsync();

        Definitions = allDefs
            .Where(d => d.Page.Equals(Category, StringComparison.OrdinalIgnoreCase))
            .ToList();
    }
}

@if (Definitions.Any())
{
    @foreach (var def in Definitions)
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
                        return response.json();      // JS-side JSON parsing is fine
                    })
                    .then(function(cache) {
                        if (!cache.Rows || cache.Rows.length === 0) return;
                        var options = {
                            chart: { id: 'chart_' + chartId, type: '@def.ChartType' },
                            xaxis: { categories: cache.Rows.map(r => r.Label) },
                            series: [{ name: cache.ChartId, data: cache.Rows.map(r => r.Value) }]
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
    <div class="alert alert-warning">
        No charts found for “@Category”
    </div>
}


```



Task<IEnumerable<ChartDefinition>> GetAllDefinitionsAsync();
