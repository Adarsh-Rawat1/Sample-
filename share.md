@page "/test-apex"
<div id="my_chart" style="height:350px"></div>
<button @onclick="RenderChart">Show Chart</button>
@code {
    [Inject] IJSRuntime JS { get; set; }
    async Task RenderChart()
    {
        var opts = new {
            chart = new { id = "my_chart", type = "bar" },
            xaxis = new { categories = new[] { "A", "B", "C" } },
            series = new[] { new { name = "Demo", data = new[] { 1, 2, 3 } } }
        };
        await JS.InvokeVoidAsync("apexInterop.renderChart", "my_chart", opts);
    }
}

