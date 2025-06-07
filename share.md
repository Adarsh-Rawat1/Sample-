protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (_needsRender && _pendingOptions is not null)
    {
        try
        {
            // Attempt JS interop—will throw during prerender, but succeed once interactive
            await JS.InvokeVoidAsync("apexInterop.renderChart", elementId, _pendingOptions);
            _needsRender = false;  // only draw once
        }
        catch (InvalidOperationException)
        {
            // prerendering blocked the call—ignore and retry on next render
        }
    }
}
