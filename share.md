You're on the right track—the fact that the `XLabel` and `YLabel` are visible in the JSON means your backend parsing is solid. Now it's all about correctly passing those values into your ApexCharts options and ensuring the chart renders with axis titles visible.

---

## ✅ Step 1: Update your `Build…Options` in the Razor component

Make sure each chart options builder includes `xaxis.title.text` and `yaxis.title.text`. Here’s a refined version for the bar chart:

```csharp
private object BuildBarOptions(ChartDataCache c) => new
{
    chart = new { id = elementId, type = "bar", toolbar = new { show = true }, zoom = new { enabled = false } },
    title = new { text = Definition.Title, align = "left" },
    subtitle = new { text = c.Summary, align = "left" },
    xaxis = new
    {
        categories = c.Rows.Select(r => r.Label).ToArray(),
        title = new { text = c.XLabel }
    },
    yaxis = new
    {
        title = new { text = c.YLabel }
    },
    series = new[] { new { name = Definition.ChartId, data = c.Rows.Select(r => r.Value).ToArray() } }
};
```

### Mirror this in your other builders as well:

* **Line charts**
* **Scatter/datetime charts** (scatter also benefits from `xaxis.title.text`, even if using timestamps)

---

## 💡 Why this matters:

* `xaxis.title.text` and `yaxis.title.text` are the proper options for axis titles per ApexCharts docs ([apexcharts.com][1]).
* If these aren't present or mis-typed, the axis titles won't appear—even if your JSON contains the correct labels!

---

## ✅ Step 2: Check that `apexInterop.renderChart` correctly applies the options

Ensure your JavaScript side doesn’t filter out `xaxis.title` or `yaxis.title`. The full shape should be passed as-is. You may also test:

```js
console.log(options); // inspect xaxis.title and yaxis.title before rendering
```

---

## 🔧 Step 3: Verify for specific chart types

For scatter charts:

```csharp
xaxis = new
{
    type = "datetime",
    title = new { text = c.XLabel },
    labels = new { format = "dd MMM HH:mm" }
},
yaxis = new { title = new { text = c.YLabel } },
```

---

## 🧪 Troubleshooting tips

* Confirm your JSON still includes correct properties after update.
* Open DevTools and inspect the DOM/SVG: axis title text elements have classes like `.apexcharts-xaxis-title` or `.apexcharts-yaxis-title`.
* No need for CSS hacks—ApexCharts renders those title elements automatically when configured .

---

If you apply these changes and the axes still lack labels, share the rendered HTML snippet or your JS options object, and I’ll help debug further!

[1]: https://www.apexcharts.com/docs/options/xaxis/?utm_source=chatgpt.com "xaxis - ApexCharts.js"
