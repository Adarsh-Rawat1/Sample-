@page "/demo-chart"

<h3>Demo bar chart</h3>

<ApexChart TItem="MyData" Title="Monthly Profit"
           RenderMode="InteractiveServer">
  <ApexPointSeries TItem="MyData"
                   Items="Data"
                   Name="Net Profit"
                   SeriesType="SeriesType.Bar"
                   XValue="e => e.Category"
                   YValue="e => e.NetProfit" />
</ApexChart>

@code {
  List<MyData> Data = new();

  protected override void OnInitialized() {
    Data = new List<MyData> {
      new() { Category = "Jan", NetProfit = 12 },
      new() { Category = "Feb", NetProfit = 43 },
      new() { Category = "Mar", NetProfit = 112 },
    };
  }

  class MyData {
    public string Category { get; set; } = default!;
    public int NetProfit { get; set; }
  }
}

