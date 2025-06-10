var options = definition.ChartType.ToLower() switch
{
    "bar" => new
    {
        chart = new { id = elementId, type = "bar", toolbar = new { show = true }, zoom = new { enabled = false } },
        xaxis = new { categories = cache.Rows.Select(r => r.Label).ToArray() },
        series = new[] { new { name = definition.Title, data = cache.Rows.Select(r => r.Value).ToArray() } }
    },

    "scatter" => new
    {
        chart = new { id = elementId, type = "scatter", toolbar = new { show = true }, zoom = new { enabled = true } },
        xaxis = new { type = "datetime", labels = new { format = "dd MMM HH:mm" } },
        series = new[]
        {
            new
            {
                name = definition.Title,
                data = cache.Rows
                           .Select(r => new object[]
                           {
                               DateTimeOffset.Parse(r.Label).ToUnixTimeMilliseconds(),
                               r.Value
                           })
                           .ToArray()
            }
        }
    },

    // fallback for any other chart type
    _ => throw new InvalidOperationException($"Unknown chart type '{definition.ChartType}'")
};
