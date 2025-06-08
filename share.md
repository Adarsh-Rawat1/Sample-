```

@code {
  // … your other fields and LoadDataAsync …

  protected override async Task OnAfterRenderAsync(bool firstRender)
  {
      // Always log entry (in server console)
      Console.WriteLine($"[ChartBlock] OnAfterRenderAsync firstRender={firstRender}, needsRender={_needsRender}");

      // Also log into the browser console
      await JS.InvokeVoidAsync("console.log", $"[ChartBlock] firstRender={firstRender}, needsRender={_needsRender}");

      // Only attempt to render once we're past prerender and have new data
      if (!firstRender && _needsRender && _pendingOptions is not null)
      {
          Console.WriteLine($"[ChartBlock] Invoking apexInterop.renderChart for '{elementId}'");
          await JS.InvokeVoidAsync("console.log", $"[ChartBlock] calling apexInterop.renderChart('{elementId}')");

          try
          {
              await JS.InvokeVoidAsync("apexInterop.renderChart", elementId, _pendingOptions);
              Console.WriteLine($"[ChartBlock] renderChart succeeded for '{elementId}'");
              await JS.InvokeVoidAsync("console.log", $"[ChartBlock] renderChart succeeded for '{elementId}'");

              _needsRender = false;
          }
          catch (Exception ex)
          {
              Console.WriteLine($"[ChartBlock] renderChart **ERROR** for '{elementId}': {ex.Message}");
              await JS.InvokeVoidAsync("console.error", $"[ChartBlock] renderChart ERROR for '{elementId}': {ex.Message}");
          }
      }
  }
}
```
