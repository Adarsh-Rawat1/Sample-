```
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using StarTrendsDashboard.Shared;
using Oracle.ManagedDataAccess.Client;
using System;
using System.Collections.Generic;
using System.Data;
using System.IO;
using System.Linq;
using System.Text.Json;
using System.Threading.Tasks;

namespace StarTrendsDashboard.Services
{
    public class ChartService : IChartService
    {
        private readonly string _definitionsJsonPath;
        private readonly string _queryFolder;
        private readonly string _connectionString;
        private readonly string _cacheFolder;
        private readonly ILogger<ChartService> _logger;
        private readonly List<ChartDefinition> _definitions = new();
        private DateTime _defsLastWriteTimeUtc;
        private readonly object _lock = new();

        public ChartService(IConfiguration config, ILogger<ChartService> logger)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            _definitionsJsonPath = config["ChartDefinitionsPath"] ?? throw new ArgumentNullException("ChartDefinitionsPath missing");
            _queryFolder = config["ChartQueryFolder"] ?? throw new ArgumentNullException("ChartQueryFolder missing");
            _cacheFolder = config["ChartCacheFolder"] ?? throw new ArgumentNullException("ChartCacheFolder missing");
            _connectionString = config.GetConnectionString("OracleDb") ?? throw new ArgumentNullException("OracleDb missing");
            Directory.CreateDirectory(_cacheFolder);
            LoadDefinitions();
        }

        private void LoadDefinitions()
        {
            var fi = new FileInfo(_definitionsJsonPath);
            if (!fi.Exists) throw new FileNotFoundException($"Cannot find {_definitionsJsonPath}");
            if (fi.LastWriteTimeUtc <= _defsLastWriteTimeUtc && _definitions.Count > 0) return;

            var json = File.ReadAllText(_definitionsJsonPath);
            var defs = JsonSerializer.Deserialize<List<ChartDefinition>>(json) ?? new List<ChartDefinition>();
            lock (_lock)
            {
                _definitions.Clear();
                _definitions.AddRange(defs);
                _defsLastWriteTimeUtc = fi.LastWriteTimeUtc;
            }

            // Extract category from header
            foreach (var def in _definitions)
            {
                var path = Path.Combine(_queryFolder, def.SqlFile);
                if (File.Exists(path))
                {
                    var parts = File.ReadLines(path).FirstOrDefault()?.Split(',') ?? Array.Empty<string>();
                    def.Category = parts.ElementAtOrDefault(def.Dimensions)?.Trim() ?? string.Empty;
                }
            }
            _logger.LogInformation("Loaded {Count} definitions", _definitions.Count);
        }

        public IReadOnlyList<ChartDefinition> GetAllDefinitions()
        {
            LoadDefinitions();
            lock (_lock) { return _definitions.ToList(); }
        }

        public async Task<ChartDataCache> RefreshChartIfNeededAsync(string chartId)
        {
            LoadDefinitions();
            var def = _definitions.FirstOrDefault(d => d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
            if (def == null) return await RefreshChartAsync(chartId);

            var outPath = Path.Combine(_cacheFolder, $"{chartId}.json");
            if (File.Exists(outPath))
            {
                try
                {
                    var existing = JsonSerializer.Deserialize<ChartDataCache>(await File.ReadAllTextAsync(outPath))!;
                    if ((DateTime.UtcNow - existing.LastUpdatedUtc).TotalSeconds < def.RefreshIntervalSeconds)
                    {
                        _logger.LogInformation("Using cache for '{chartId}'", chartId);
                        return existing;
                    }
                }
                catch (Exception ex)
                {
                    _logger.LogWarning(ex, "Cache read failed for '{chartId}', regenerating", chartId);
                }
            }
            return await RefreshChartAsync(chartId);
        }

        public async Task<ChartDataCache> RefreshChartAsync(string chartId)
        {
            LoadDefinitions();
            var def = _definitions.FirstOrDefault(d => d.ChartId.Equals(chartId, StringComparison.OrdinalIgnoreCase));
            if (def == null)
            {
                _logger.LogWarning("No definition for '{chartId}'", chartId);
                return new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };
            }

            var sqlPath = Path.Combine(_queryFolder, def.SqlFile);
            if (!File.Exists(sqlPath))
            {
                _logger.LogError("SQL file not found: {sqlPath}", sqlPath);
                return new ChartDataCache { ChartId = chartId, LastUpdatedUtc = DateTime.UtcNow };
            }

            var lines = (await File.ReadAllTextAsync(sqlPath))
                         .Split(new[] { "\r\n", "\n" }, StringSplitOptions.None);

            // Header parsing
            var hdr = lines[0].Split(',');
            var xLabel = hdr.ElementAtOrDefault(0)?.Trim() ?? string.Empty;
            var yLabel = hdr.ElementAtOrDefault(1)?.Trim() ?? string.Empty;
            var zLabel = def.Dimensions == 3 ? hdr.ElementAtOrDefault(2)?.Trim() : null;
            var category = hdr.ElementAtOrDefault(def.Dimensions)?.Trim() ?? string.Empty;

            // Optional summary
            string? summary = null;
            var start = 1;
            if (lines.Length > 1 && lines[1].Trim().StartsWith("{") && lines[1].Trim().EndsWith("}"))
            {
                summary = lines[1].Trim('{', '}').Trim();
                start = 2;
            }

            // SQL build & trim semicolon
            var rawSql = string.Join("\n", lines.Skip(start)).TrimEnd();
            if (rawSql.EndsWith(";")) rawSql = rawSql[..^1].TrimEnd();

            var data = new List<object[]>();
            using (var conn = new OracleConnection(_connectionString))
            {
                await conn.OpenAsync();
                using var cmd = conn.CreateCommand();
                cmd.CommandText = rawSql;
                using var reader = await cmd.ExecuteReaderAsync();
                while (await reader.ReadAsync())
                {
                    if (def.Dimensions == 3)
                    {
                        var x = reader.IsDBNull(0) ? string.Empty : reader.GetString(0);
                        var y = reader.IsDBNull(1) ? string.Empty : reader.GetString(1);
                        var z = reader.IsDBNull(2) ? 0m : Convert.ToDecimal(reader.GetValue(2));
                        data.Add(new object[] { x, y, z });
                    }
                    else
                    {
                        var x = reader.IsDBNull(0) ? string.Empty : reader.GetString(0);
                        var y = reader.IsDBNull(1) ? 0m : Convert.ToDecimal(reader.GetValue(1));
                        data.Add(new object[] { x, y });
                    }
                }
            }

            // Compute EMA if requested
            List<decimal>? emaSeries = null;
            if (def.ShowAverageLine && def.Dimensions == 2 && data.Count > 0)
            {
                emaSeries = new List<decimal>(data.Count);
                const int period = 6;
                var alpha = 2m / (period + 1);
                decimal? prev = null;
                foreach (var point in data)
                {
                    var val = Convert.ToDecimal(point[1]);
                    var ema = prev.HasValue ? (alpha * val + (1 - alpha) * prev.Value) : val;
                    emaSeries.Add(ema);
                    prev = ema;
                }
            }

            var cache = new ChartDataCache
            {
                ChartId = chartId,
                LastUpdatedUtc = DateTime.UtcNow,
                Data = data,
                XLabel = xLabel,
                YLabel = yLabel,
                ZLabel = zLabel,
                Category = category,
                Summary = summary,
                EmaSeries = emaSeries
            };

            var json = JsonSerializer.Serialize(cache, new JsonSerializerOptions { WriteIndented = true });
            await File.WriteAllTextAsync(Path.Combine(_cacheFolder, $"{chartId}.json"), json);

            _logger.LogInformation("Refreshed '{chartId}' → {Count} points, wrote cache", chartId, data.Count);
            return cache;
        }
    }
}

```


I’ve added:

* **`EmaSeries`** to `ChartDataCache` (holds the 6-period EMA if `ShowAverageLine` is set).
* EMA calculation in `RefreshChartAsync` (α = 2/(6+1)).
* Assigned `EmaSeries` instead of a flat average.

Next, update your **`BuildLineOptions`** to plot `c.EmaSeries`:

```csharp
if (c.EmaSeries != null)
{
    seriesList.Add(new
    {
        name    = "EMA (6)",
        type    = "line",
        data    = mainDates.Zip(c.EmaSeries, (x, e) => new object[] { x, e }).ToArray(),
        stroke  = new { width = 1, dashArray = 4 },
        markers = new { size = 0 }
    });
}
```

Drop the old `AverageValue` logic, and you’ll now see a true 6-period EMA line on your charts. Let me know when you want the updated chart block snippet too!



