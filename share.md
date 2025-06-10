Below are the two final files—**NavMenu.razor** and **MainLayout.razor**—fully wired up for a responsive, collapsible sidebar on mobile and a fixed sidebar on desktop. No extra comments, just the essentials.

---

## NavMenu.razor

```razor
@using StarTrendsDashboard.Services
@inject IChartService ChartService

<div class="offcanvas offcanvas-start d-md-flex flex-column vh-100 bg-white shadow-sm"
     tabindex="-1" id="sidebar">
  <div class="offcanvas-header d-md-none">
    <h5 class="offcanvas-title">StarTrends</h5>
    <button type="button" class="btn-close" data-bs-dismiss="offcanvas"></button>
  </div>
  <div class="offcanvas-body p-0">
    <nav class="nav nav-pills flex-column px-2 py-3">
      @foreach (var cat in ChartService.GetAllDefinitions().Select(d => d.Page).Distinct())
      {
        <NavLink class="nav-link mb-1 rounded"
                 href="@($"/charts/{cat}")"
                 Match="NavLinkMatch.Prefix"
                 data-bs-dismiss="offcanvas">
          @cat
        </NavLink>
      }
    </nav>
  </div>
</div>
```

---

## MainLayout.razor

```razor
@inherits LayoutComponentBase

<nav class="navbar navbar-light bg-light shadow-sm d-md-none">
  <div class="container-fluid">
    <button class="btn" type="button" data-bs-toggle="offcanvas" data-bs-target="#sidebar">
      <span class="navbar-toggler-icon"></span>
    </button>
    <span class="navbar-brand mb-0 h1">StarTrends</span>
  </div>
</nav>

<div class="d-flex">
  <NavMenu />

  <main class="flex-grow-1 p-4 ms-md-5">
    @Body
  </main>
</div>
```

---

**Checklist**

1. **Bootstrap JS** must be included in `_Host.cshtml`:

   ```html
   <script src="lib/bootstrap/js/bootstrap.bundle.min.js" defer></script>
   ```
2. On **mobile (<768px)** you’ll see a top navbar with a hamburger. Tapping it slides in the sidebar.
3. On **desktop (≥768px)** the sidebar is always visible and the toggle bar is hidden.

That’s it—fully responsive, collapsible nav with no extra fluff.
