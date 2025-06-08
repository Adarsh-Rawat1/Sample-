window.apexInterop = {
  renderChart: function (elementId, config) {
    console.log(`[apexInterop] renderChart id=${elementId}`, config);
    const el = document.getElementById(elementId);
    if (!el) {
      console.warn(`[apexInterop] Container '${elementId}' not found.`);
      return;
    }
    const existing = ApexCharts.getChartByID(elementId);
    if (existing) existing.destroy();
    new ApexCharts(el, config).render();
  },
  updateSeries: function (elementId, newSeries) {
    const chart = ApexCharts.getChartByID(elementId);
    if (chart) chart.updateSeries(newSeries, true);
  }
};







