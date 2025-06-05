Below is a concrete example showing how to put **one chart on the “Product” page** and **one chart on the “Trade” page** using the architecture described earlier. We’ll:

1. Create two SQL files (one for “Markets” under Product, one for “Tools” under Trade).
2. Append their definitions to `chart‐definitions.json`, assigning `"Page": "Product"` for the first and `"Page": "Trade"` for the second.
3. Show exactly how those two charts will appear (and only those two) at the routes `/charts/Product` and `/charts/Trade`.

> **Reminder**: We assume you have already wired up:
>
> * `ChartService` (reads JSON + SQL folder, caches results).
> * `ChartPollingService` (polls every `RefreshIntervalSeconds`).
> * A single generic Razor page `ChartPage.razor` with route `/charts/{pageName}` that renders *all* charts whose `"Page"` field matches `pageName`.

---

## 1. Place the two SQL files under `ChartDefinitions/Queries/`

### 1.1. `ChartDefinitions/Queries/ProductMarkets.sql`

```sql
-- ProductMarkets.sql
-- “Where Markets are set in the last month”
SELECT
  r.feature AS "Markets",
  COUNT(1) AS "Times used"
FROM star_action_audit r
WHERE r.mod_dt > TRUNC(SYSDATE) - 30
  AND feature_type IN ('MARKET')
GROUP BY r.feature
ORDER BY "Times used" DESC;
```

Make sure this file lives at:

```
StarTrendsDashboard/
└── ChartDefinitions/
    └── Queries/
        └── ProductMarkets.sql
```

### 1.2. `ChartDefinitions/Queries/TradeTools.sql`

```sql
-- TradeTools.sql
-- “Tools used last month”
SELECT
  r.feature AS "Tools",
  COUNT(1) AS "Times used"
FROM star_action_audit r
WHERE r.mod_dt > TRUNC(SYSDATE) - 30
  AND feature_type IN ('TOOLS')
GROUP BY r.feature
ORDER BY "Times used" DESC;
```

Place it here:

```
StarTrendsDashboard/
└── ChartDefinitions/
    └── Queries/
        └── TradeTools.sql
```

> **Note**: The filenames (`ProductMarkets.sql` and `TradeTools.sql`) should match exactly what you put into JSON’s `"SqlFile"` field.

---

## 2. Update `ChartDefinitions/chart‐definitions.json`

Open (or create) the file `ChartDefinitions/chart‐definitions.json` and ensure it contains exactly these **two** JSON objects:

```jsonc
[
  {
    "ChartId": "ProductMarkets",
    "Page": "Product",
    "Title": "Markets Set in Last 30 Days",
    "ChartType": "Bar",
    "SqlFile": "ProductMarkets.sql",
    "RefreshIntervalSeconds": 300
  },
  {
    "ChartId": "TradeTools",
    "Page": "Trade",
    "Title": "Tools Used in Last 30 Days",
    "ChartType": "Bar",
    "SqlFile": "TradeTools.sql",
    "RefreshIntervalSeconds": 300
  }
]
```

* **ChartId**: a unique ID (we use `"ProductMarkets"` and `"TradeTools"`).
* **Page**:

  * `"Product"` → means this chart will live on `/charts/Product`.
  * `"Trade"` → means this chart will live on `/charts/Trade`.
* **Title**: what the chart will display above.
* **ChartType**: we chose `"Bar"` (so both render as bar charts).
* **SqlFile**: must exactly match the filename under `ChartDefinitions/Queries/`.
* **RefreshIntervalSeconds**: 300 sec = 5 minutes between automatic polls.

Your final folder should look like:

```
StarTrendsDashboard/
├── ChartDefinitions/
│   ├── chart‐definitions.json   ← contains exactly those two entries
│   └── Queries/
│       ├── ProductMarkets.sql
│       └── TradeTools.sql
…
```

---

## 3. How `ChartPage.razor` will render them

We assume your `ChartPage.razor` looks like this (see prior instructions). The key part is that it reads `pageName` from the URL and fetches all definitions whose `"Page"` = `pageName`. In our case:

* Visiting **`/charts/Product`** → `pageName = "Product"` → it finds the `"ProductMarkets"` definition and renders exactly that one chart.
* Visiting **`/charts/Trade`** → `pageName = "Trade"` → it finds the `"TradeTools"` definition and renders exactly that one chart.

Below is the minimal version of `ChartPage.razor` (you may already have something very similar):

```razor
@page "/charts/{pageName}"
@inject IChartService ChartService
@using BlazorApexCharts

<h3>Charts for “@pageName”</h3>

@if (_definitions == null || !_definitions.Any())
{
    <p class="text-muted">No charts configured for “@pageName”.</p>
}
else
{
    @foreach (var def in _definitions)
    {
        <div class="mb-5">
            <h4>@def.Title</h4>
            <button class="btn btn-sm btn-outline-primary mb-2"
                    @onclick="() => ManualRefresh(def.ChartId)">
                Refresh Now
            </button>

            @if (_cacheMap.TryGetValue(def.ChartId, out var cache) == false
                  || cache.Rows.Count == 0)
            {
                <p><em>Loading data…</em></p>
            }
            else
            {
                <ApexChart TItem="object"
                           Width="100%"
                           Height="350px"
                           ChartOptions="GetOptions(def, cache)"
                           Series="GetSeries(def, cache)">
                </ApexChart>
                <p class="text-muted">
                    Last updated: @cache.LastUpdatedUtc.ToLocalTime().ToString("g")
                </p>
            }
        </div>
    }
}

@code {
    [Parameter]
    public string pageName { get; set; }

    private List<ChartDefinition> _definitions = new();
    private readonly Dictionary<string, ChartDataCache> _cacheMap 
        = new(StringComparer.OrdinalIgnoreCase);

    protected override async Task OnInitializedAsync()
    {
        // 1) Load only the definitions whose "Page" matches pageName
        _definitions = ChartService.GetDefinitionsByPage(pageName).ToList();

        // 2) For each definition, ensure there's at least one cached fetch
        foreach (var def in _definitions)
        {
            var existing = ChartService.GetCachedData(def.ChartId);
            if (existing == null || existing.Rows.Count == 0)
            {
                var loaded = await ChartService.RefreshChartAsync(def.ChartId);
                _cacheMap[def.ChartId] = loaded;
            }
            else
            {
                _cacheMap[def.ChartId] = existing;
            }
        }
    }

    private ApexChartOptions<object> GetOptions(ChartDefinition def, ChartDataCache cache)
    {
        var rows = cache.Rows;
        // For “Bar” type, put labels on X and values on Y
        if (def.ChartType.Equals("Bar", StringComparison.OrdinalIgnoreCase))
        {
            var labels = rows.Select(r => r.Label).ToArray();
            var values = rows.Select(r => r.Value).ToArray();

            return new ApexChartOptions<object>
            {
                Chart = new ApexChart
                {
                    Type = ChartType.Bar,
                    Toolbar = new ApexToolbar { Show = true },
                    Zoom = new ApexZoom { Enabled = false }
                },
                Xaxis = new ApexXAxis { Categories = labels },
                Title = new ApexTitle { Text = def.Title, Align = ApexTitleAlign.Left }
            };
        }

        // (If you need other chart types, add branches here.)
        // Fallback to a simple bar chart
        {
            var labels = rows.Select(r => r.Label).ToArray();
            var values = rows.Select(r => r.Value).ToArray();
            return new ApexChartOptions<object>
            {
                Chart = new ApexChart
                {
                    Type = ChartType.Bar,
                    Toolbar = new ApexToolbar { Show = true },
                    Zoom = new ApexZoom { Enabled = false }
                },
                Xaxis = new ApexXAxis { Categories = labels },
                Title = new ApexTitle { Text = def.Title, Align = ApexTitleAlign.Left }
            };
        }
    }

    private IEnumerable<ChartSeries<object>> GetSeries(ChartDefinition def, ChartDataCache cache)
    {
        var rows = cache.Rows;
        if (def.ChartType.Equals("Bar", StringComparison.OrdinalIgnoreCase))
        {
            var values = rows.Select(r => r.Value).ToArray();
            return new[]
            {
                new ChartSeries<object>
                {
                    Name = def.ChartId,
                    Data = values.Cast<object>()
                }
            };
        }

        // Fallback: bar
        var fallbackValues = rows.Select(r => r.Value).ToArray();
        return new[]
        {
            new ChartSeries<object>
            {
                Name = def.ChartId,
                Data = fallbackValues.Cast<object>()
            }
        };
    }

    private async Task ManualRefresh(string chartId)
    {
        var updated = await ChartService.RefreshChartAsync(chartId);
        if (updated != null)
        {
            _cacheMap[chartId] = updated;
            StateHasChanged();
        }
    }
}
```

> In short:
>
> * **OnInitializedAsync**
>
>   1. Fetches exactly those chart definitions whose `"Page"` matches the URL segment.
>   2. Ensures each has at least one `RefreshChartAsync` call so that `_cacheMap[chartId]` is populated.
> * **UI loop**: For each `def` in `_definitions` (in our case, exactly one per page), it:
>
>   1. Renders a “Refresh Now” button that re-queries and updates the cache.
>   2. Once `cache.Rows` exists, it builds a bar chart with `Categories = all Label strings` and `Data = all Value numbers`.

---

## 4. How navigation works

* **Product page**: browse to

  ```
  https://<your-host>/charts/Product
  ```

  Since our JSON has one definition with `"Page": "Product"`, that page will show exactly the “Markets Set in Last 30 Days” bar chart.

* **Trade page**: browse to

  ```
  https://<your-host>/charts/Trade
  ```

  Since our JSON has one definition with `"Page": "Trade"`, that page will show exactly the “Tools Used in Last 30 Days” bar chart.

No other pages (e.g. `/charts/OtherWhatever`) will show charts, because JSON does not define any `"Page": "OtherWhatever"`.

---

## 5. Recap of everything you need to drop in place

1. **SQL files** under `ChartDefinitions/Queries/`:

   * `ProductMarkets.sql` (Markets in last 30 days)
   * `TradeTools.sql` (Tools used in last 30 days)

2. **chart‐definitions.json** under `ChartDefinitions/`:

   ```jsonc
   [
     {
       "ChartId": "ProductMarkets",
       "Page": "Product",
       "Title": "Markets Set in Last 30 Days",
       "ChartType": "Bar",
       "SqlFile": "ProductMarkets.sql",
       "RefreshIntervalSeconds": 300
     },
     {
       "ChartId": "TradeTools",
       "Page": "Trade",
       "Title": "Tools Used in Last 30 Days",
       "ChartType": "Bar",
       "SqlFile": "TradeTools.sql",
       "RefreshIntervalSeconds": 300
     }
   ]
   ```

3. **ChartService & ChartPollingService** (as described previously) already in place.

4. **ChartPage.razor** (generic `/charts/{pageName}`) already in place.

5. **NavMenu** links (optional):

   ```razor
   <NavLink class="nav-link" href="charts/Product">Product</NavLink>
   <NavLink class="nav-link" href="charts/Trade">Trade</NavLink>
   ```

Once you have those two SQL files and that JSON file in place, simply **restart** the Blazor Server app:

```bash
dotnet run
```

* The moment the app starts, `ChartPollingService` will see `"ProductMarkets"` and `"TradeTools"` in the in-memory definitions.
* It will schedule each for an immediate refresh, so their caches populate right away.
* Navigate to `/charts/Product` → you see a bar chart of “Markets (Label) vs Times used (Value)”.
* Navigate to `/charts/Trade` → you see a bar chart of “Tools (Label) vs Times used (Value)”.

---

### That is how you get exactly **one chart on the Product page** and **one chart on the Trade page**.

No extra code is needed beyond:

* Placing the SQL files in `ChartDefinitions/Queries/`.
* Adding each entry to `chart‐definitions.json` with `"Page": "Product"` or `"Page": "Trade"`.
* Relying on the generic `/charts/{pageName}` Razor page to show only that page’s charts.
