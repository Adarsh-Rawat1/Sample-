public async Task<ChartDataCache> RefreshChartIfNeededAsync(string chartId)
{
    var def = _definitions
        .FirstOrDefault(d => d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
    if (def == null)
        return await RefreshChartAsync(chartId);  // fallback

    string outPath = Path.Combine(_cacheFolder, $"{chartId}.json");

    if (File.Exists(outPath))
    {
        try
        {
            var existing = JsonSerializer.Deserialize<ChartDataCache>(await File.ReadAllTextAsync(outPath))!;
            if ((DateTime.UtcNow - existing.LastUpdatedUtc).TotalSeconds < def.RefreshInterval)
            {
                _logger.LogInformation("Using cached data for '{chartId}'", chartId);
                return existing;
            }
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Failed reading cache for '{chartId}', regenerating", chartId);
        }
    }

    return await RefreshChartAsync(chartId);
}
