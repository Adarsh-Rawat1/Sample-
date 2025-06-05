window.apexInterop = {
  renderChart: function (elementId, config) {
    console.log(`[apexInterop] renderChart called for id='${elementId}', config=`, config);
    const elem = document.querySelector(`#${elementId}`);
    if (!elem) {
      console.warn(`[apexInterop] Cannot find container #${elementId}`);
      return;
    }
    // if there’s an existing chart, destroy it
    if (ApexCharts.getChartByID(elementId)) {
      ApexCharts.getChartByID(elementId).destroy();
    }
    const chart = new ApexCharts(elem, config);
    chart.render();
  },
  updateSeries: function (elementId, newSeries) {
    console.log(`[apexInterop] updateSeries called for id='${elementId}', newSeries=`, newSeries);
    const chart = ApexCharts.getChartByID(elementId);
    if (chart) {
      chart.updateSeries(newSeries, true);
    } else {
      console.warn(`[apexInterop] No existing chart instance for #${elementId}`);
    }
  }
};
