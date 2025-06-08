```
@page "/charts/{category}"
@using StarTrendsDashboard.Shared
@inject IChartService ChartService

@foreach (var def in ChartService.GetAllDefinitions()
              .Where(d => d.Page.Equals(category, StringComparison.OrdinalIgnoreCase)))
{
  <section class="mb-5">
    <h4>@def.Title</h4>
    <div id="chart_@def.ChartId" style="min-height:350px"></div>
    <script>
      (function() {
        fetch('/ChartCache/@def.ChartId.json')
          .then(response => {
            if (!response.ok) throw new Error('No cache');
            return response.json();
          })
          .then(cache => {
            if (!cache.Rows || cache.Rows.length === 0) return;
            // Build options
            const options = {
              chart: { id: 'chart_@def.ChartId', type: '@def.ChartType' },
              xaxis: { categories: cache.Rows.map(r => r.Label) },
              series: [{ name: cache.ChartId, data: cache.Rows.map(r => r.Value) }]
            };
            // Render
            const container = document.querySelector('#chart_@def.ChartId');
            ApexCharts.getChartByID('chart_@def.ChartId')?.destroy();
            new ApexCharts(container, options).render();
          })
          .catch(err => {
            console.warn('Chart load failed for @def.ChartId:', err);
          });
      })();
    </script>
  </section>
}

```
