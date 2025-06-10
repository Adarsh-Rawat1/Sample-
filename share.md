You’ll need to update **two main parts** of your codebase (and optionally adjust one JSON file). Here’s what to change:

---

### 1. **`ChartService.cs`** (inside `RefreshChartAsync`)

This is where you’ll parse the SQL header, extract labels/summary/category, strip the trailing semicolon, and include the metadata in your returned `ChartDataCache`.
You should replace the part where you're currently just reading `rawSql` with the full snippet provided earlier:

```csharp
// Read the whole SQL file
string content = await File.ReadAllTextAsync(sqlPath);
string[] lines = content.Split(new[] { "\r\n", "\n" }, StringSplitOptions.None);

// Parse header (first line)
var header = lines[0].Split(',');
string xLabel = header.ElementAtOrDefault(0)?.Trim() ?? "";
string yLabel = header.ElementAtOrDefault(1)?.Trim() ?? "";
string category = header.ElementAtOrDefault(2)?.Trim() ?? "";

// Parse optional summary (second line)
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

// Reconstruct SQL and strip exactly one trailing semicolon
string rawSql = string.Join("\n", lines.Skip(sqlStart)).TrimEnd();
if (rawSql.EndsWith(";"))
    rawSql = rawSql.Substring(0, rawSql.Length - 1).TrimEnd();

// Then use `rawSql` in the OracleCommand and build rows as before…

// Finally, include the metadata in your ChartDataCache:
var cache = new ChartDataCache {
    ChartId = chartId,
    LastUpdatedUtc = DateTime.UtcNow,
    Rows = rows,
    XLabel = xLabel,
    YLabel = yLabel,
    Category = category,
    Summary = summary
};
```

---

### 2. **`ChartDataCache.cs`** (or wherever it’s defined)

Add properties to hold the values parsed from the SQL header:

```csharp
public class ChartDataCache
{
    public string ChartId { get; set; } = "";
    public DateTime LastUpdatedUtc { get; set; }
    public List<ChartDataRow> Rows { get; set; } = new();

    // Add these new properties:
    public string XLabel { get; set; } = "";
    public string YLabel { get; set; } = "";
    public string Category { get; set; } = "";
    public string? Summary { get; set; }
}
```

---

### 3. (Optional) **`chart-definitions.json`**

If you're no longer defining `Category` in your JSON, update the schema to remove that field. Your JSON entries can now look like:

```json
[
  {
    "ChartId": "trade-volumes",
    "SqlFile": "TradeVols.sql",
    "ChartType": "Line",
    "RefreshInterval": 300
  },
  // ...
]
```

---

### ✅ Summary

| File                                | Changes                                                                      |
| ----------------------------------- | ---------------------------------------------------------------------------- |
| `ChartService.cs`                   | Add code to parse header & summary, strip `;`, and populate `ChartDataCache` |
| `ChartDataCache.cs`                 | Add `XLabel`, `YLabel`, `Category`, `Summary` properties                     |
| `chart-definitions.json` (optional) | Remove `Category` field from JSON if no longer needed                        |

Let me know if you'd like help wiring these values into your UI or graph components!
