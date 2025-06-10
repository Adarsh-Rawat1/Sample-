You're absolutely right – the nav bar currently builds menu items from the old `Page` property in your `ChartDefinition`. Since you're now parsing `category` directly from the `.sql` files, the navbar should also switch to use that dynamic value.

---

## 🧩 Update #1: Parse `category` earlier

In **`ChartService`**, add logic to extract `category` from each SQL in `LoadDefinitions()`. For instance:

```csharp
private void LoadDefinitions()
{
    // ... existing load code ...

    foreach (var def in _definitions)
    {
        var sqlPath = Path.Combine(_queryFolder, def.SqlFile);
        if (File.Exists(sqlPath))
        {
            string firstLine = File.ReadLines(sqlPath).FirstOrDefault() ?? "";
            var parts = firstLine.Split(',');
            def.Category = parts.ElementAtOrDefault(2)?.Trim() ?? "";
        }
    }
}
```

Ensure your `ChartDefinition` class now includes:

```csharp
public string Category { get; set; } = "";
```

---

## ✅ Update #2: Change Razor Nav to use `Category`

Then update your nav component to:

```razor
<nav class="nav flex-column">
  @foreach (var cat in ChartService
                        .GetAllDefinitions()
                        .Select(d => d.Category)
                        .Distinct())
  {
    <NavLink class="nav-link"
             href="@($"/charts/{cat}")"
             Match="NavLinkMatch.Prefix">
      @cat
    </NavLink>
  }
</nav>
```

---

### 🔎 What this does

* **Loads `category`** right after definitions are refreshed.
* **Nav pulls menu titles** from your SQL’s metadata — not static definitions.
* **Displays intuitive menu items** based on actual data categories from the `.sql` files.

---

### ✅ Summary

| File                 | Change                                                         |
| -------------------- | -------------------------------------------------------------- |
| `ChartService.cs`    | Parse `category` from SQL headers into definitions             |
| `ChartDefinition.cs` | Add `public string Category` property                          |
| Nav Razor Component  | Replace `.Select(d => d.Page)` with `.Select(d => d.Category)` |

Once implemented, your nav menu will automatically reflect the categories defined inside your SQL files. Neat, centralized, and future-proof. Let me know if you'd like help wiring routing by category too!

