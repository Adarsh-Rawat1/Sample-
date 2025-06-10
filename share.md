```
<div class="mb-5">
  <div class="d-flex justify-content-between align-items-center mb-2">
    <h4 class="m-0">@Definition.Title</h4>

    <div class="d-flex align-items-center">
      <button class="btn btn-outline-primary me-3"
              @onclick="Refresh"
              disabled="@isLoading">
        @(isLoading ? "Loading…" : "Refresh")
      </button>
      
      @* only show last-updated when non-empty *@
      @if (!string.IsNullOrEmpty(LastUpdatedText))
      {
        <small class="text-muted">@LastUpdatedText</small>
      }
    </div>
  </div>

  @if (isLoading)
  {
    <div class="spinner-border text-primary" role="status">
      <span class="visually-hidden">Loading...</span>
    </div>
  }
  else if (loadError)
  {
    <div class="alert alert-danger">Error loading data. Try again.</div>
  }
  else if (hasNoData)
  {
    <div class="alert alert-warning">No data available.</div>
  }
  else
  {
    @* give it a fixed height instead of “very big” *@
    <div id="@elementId" style="height:350px; width:100%;"></div>
  }
</div>


```
