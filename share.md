Here are two things going on:

1. **Your empty-category link**
   You’re doing

   ```csharp
   .GetAllDefinitions()
   .Select(d => d.Category)
   .Distinct()
   ```

   If any of those definitions has `Category == ""` or `null`, you end up with a `<NavLink href="/charts/">…</NavLink>` which will take you to `/charts` (and light up “Trade” if that was the first real non-empty one).

2. **Prefix matching**
   By default you set

   ```razor
   Match="NavLinkMatch.Prefix"
   ```

   which means **any** URL starting with `/charts` will mark that link active.

---

### ✂️ Fix

Update your `NavMenu.razor` to:

```razor
@using StarTrendsDashboard.Services
@inject IChartService ChartService

<aside class="position-fixed top-0 start-0 vh-100 bg-white shadow-sm">
  <div class="p-3 border-bottom">
    <a class="fs-4 fw-bold text-decoration-none" href="/">StarTrends</a>
  </div>
  <nav class="nav nav-pills flex-column px-2 py-3">
    @foreach (var cat in ChartService
      .GetAllDefinitions()
      // 1) drop any blank categories
      .Where(d => !string.IsNullOrWhiteSpace(d.Category))
      .Select(d => d.Category)
      .Distinct())
    {
      <NavLink 
        class="nav-link"
        href="@($"/charts/{cat}")"
        // 2) only match when the URL is exactly /charts/{cat}
        Match="NavLinkMatch.All">
        @cat
      </NavLink>
    }
  </nav>
</aside>
```

* **`.Where(...)`** removes that stray empty string so you won’t get a bare `/charts` link.
* **`Match="NavLinkMatch.All"`** makes “Trade” only light up when you’re exactly on `/charts/Trade`, not on any child URL.

That will kill both the unwanted blue underline under “Trade” and the click-through to the generic `/charts` page.
