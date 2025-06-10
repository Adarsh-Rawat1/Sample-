public async Task<ChartDataCache> RefreshChartAsync(string chartId)
{
    LoadDefinitions();

    var def = _definitions
        .FirstOrDefault(d => d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
    if (def == null) return new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };

    var sqlPath = Path.Combine(_queryFolder, def.SqlFile);
    if (!File.Exists(sqlPath)) return new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };

    // 1. Read raw content and split lines
    string content = await File.ReadAllTextAsync(sqlPath);
    string[] lines = content.Split(new[] { "\r\n", "\n" }, StringSplitOptions.None);

    // 2. Extract metadata from first line (a,b,c)
    var header = lines[0].Split(',');
    string xLabel = header.ElementAtOrDefault(0)?.Trim() ?? "";
    string yLabel = header.ElementAtOrDefault(1)?.Trim() ?? "";
    string category = header.ElementAtOrDefault(2)?.Trim() ?? "";

    // 3. Optional summary in second line {…}
    string? summary = null;
    int sqlStart = 1;
    if (lines.Length > 1)
    {
        var l2 = lines[1].Trim();
        if (l2.StartsWith("{") && l2.EndsWith("}"))
        {
            summary = l2.Trim('{', '}').Trim();
            sqlStart = 2;
        }
    }

    // 4. Reconstruct raw SQL and strip exactly one trailing semicolon
    string rawSql = string.Join("\n", lines.Skip(sqlStart))
        .TrimEnd();
    if (rawSql.EndsWith(";"))
        rawSql = rawSql[..^1].TrimEnd();

    // 5. Execute the query
    var rows = new List<ChartDataRow>();
    try
    {
        using var conn = new OracleConnection(_connectionString);
        await conn.OpenAsync();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = rawSql;
        cmd.CommandType = CommandType.Text;
        using var reader = await cmd.ExecuteReaderAsync();
        while (await reader.ReadAsync())
        {
            rows.Add(new ChartDataRow
            {
                Label = reader.IsDBNull(0) ? string.Empty : reader.GetString(0),
                Value = reader.IsDBNull(1) ? 0 : Convert.ToDecimal(reader.GetValue(1))
            });
        }
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error executing SQL for '{chartId}'", chartId);
        return new ChartDataCache
        {
            ChartId = chartId,
            LastUpdatedUtc = DateTime.UtcNow,
            Rows = rows
        };
    }

    // 6. Embed metadata into result object
    var cache = new ChartDataCache
    {
        ChartId = chartId,
        LastUpdatedUtc = DateTime.UtcNow,
        Rows = rows,
        XLabel = xLabel,
        YLabel = yLabel,
        Category = category,
        Summary = summary
    };

    // 7. Serialize and write cache file
    string jsonOut = JsonSerializer.Serialize(cache, new JsonSerializerOptions
    {
        WriteIndented = true
    });
    await File.WriteAllTextAsync(Path.Combine(_cacheFolder, $"{chartId}.json"), jsonOut);

    _logger.LogInformation("Refreshed '{chartId}' → {Count} rows, wrote {outPath}",
        chartId, rows.Count, Path.Combine(_cacheFolder, $"{chartId}.json"));

    return cache;
}

