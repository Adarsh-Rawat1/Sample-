To make the sidebar collapse on smaller viewports, you can leverage Bootstrap’s Collapse (or Offcanvas) component instead of a fixed sidebar. Here’s a responsive pattern that:

* Shows a toggle button in the top navbar on small screens
* Opens/closes the sidebar as an overlay (Offcanvas) or sliding panel
* Falls back to a static sidebar on medium+ screens

---

## 1. Add a top Navbar with a toggle

Place this in your **MainLayout.razor** (above or alongside your existing navbar):

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
```

* `d-md-none` hides this bar on medium (≥768px) and up.
* The button toggles an Offcanvas with ID `sidebar`.

---

## 2. Convert the sidebar to an Offcanvas

Update **NavMenu.razor** to wrap your `<aside>` in an Offcanvas container:

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

* `.offcanvas-start`: sidebar slides in from the left.
* `.d-md-flex`: on md+ it acts like a normal flex column (always visible).
* The header with close button only shows on mobile (`d-md-none`).
* Each link has `data-bs-dismiss="offcanvas"` so selecting a page auto-closes it on mobile.

---

## 3. Adjust main content margin

In **MainLayout.razor**, ensure your `<main>` has a left margin on md+ to sit beside the open sidebar:

```razor
<div class="d-flex">
  <NavMenu />

  <main class="flex-grow-1 p-4" style="margin-left: 0; margin-top: 0;"
        class="ms-md-5">
    @Body
  </main>
</div>
```

* `ms-md-5` adds left margin on md+ (you can tweak spacing).
* On mobile, it’s `margin-left: 0` so the offcanvas overlays the content.

---

## 4. Ensure Bootstrap JS is loaded

Your **\_Host.cshtml** must include the bundle:

```html
<script src="lib/bootstrap/js/bootstrap.bundle.min.js" defer></script>
```

---

With these changes:

* **Mobile (<768px)**: you see a top bar with hamburger; tapping it slides in the sidebar; selecting a link closes it.
* **Desktop (≥768px)**: the sidebar is fixed open on the left, and no toggle button appears.

This gives you a fully responsive, collapsible navigation without extra JavaScript.
