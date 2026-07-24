# Plan

## Summary

Add pagination to the `EpisodeListPage` using the `application-datagrids` library's `ArrayPagination` provider, defaulting to 20 episodes per page. The episode list currently renders all episodes in a single grid, which becomes unwieldy for comics with hundreds of episodes. This plan integrates server-side pagination with the existing `DataGrid` and `ArrayPagination` API, keeping the existing "refresh details" form action functional.

## Architectural Context

### Episode List Page

- File: `assets/classes/Page/EpisodeListPage.php`
- Extends `BaseComicPage` → `BasePage`
- The `_render()` method creates a `DataGrid` with a Bootstrap 5 renderer
- Episodes are loaded via `$this->comic->createFetcher()->getAll()` which returns `Episode[]`
- Each `Episode` object is converted to a row array with columns: `id`, `thumbnail`, `title`, `chapter`, `ignored`, `fetched`
- A custom form includes `episodes[]` checkboxes plus a `grid-action` select for "refresh details"
- Hidden vars `page=EpisodeList` and `comicID=...` are set on the grid form

### Datagrids Pagination API

- `ArrayPagination` (in `AppUtils\Grids\Pagination\Types\ArrayPagination`): Takes a full array, items-per-page count, optional current page, and a configurable page parameter name; slices the array for the current page
- `GridPagination` (accessed via `$grid->pagination()`): Wraps a `PaginationInterface` provider; automatically renders pagination controls in the grid footer
- `ArrayPagination::getPageURL()` rewrites `$_SERVER['REQUEST_URI']` query string, preserving existing parameters while adding/replacing the page parameter
- Bootstrap 5 pagination rendering is automatic when a provider is set and total pages > 1

### Parameter Conflict

The webcomics-builder application uses `?page=EpisodeList` for page routing (`Pages::getActivePage()` reads `$_GET['page']`). The `ArrayPagination` constructor defaults to reading `$_GET['page']` for the current page number. **These conflict.** The pagination parameter must be renamed (e.g., `ep_page`).

## Approach / Architecture

1. **Collect all episodes** as before using `$this->comic->createFetcher()->getAll()` → `Episode[]`
2. **Create `ArrayPagination`** with the full episode array, 20 items per page, and a custom page parameter `ep_page` to avoid conflicting with the routing `page` parameter
3. **Slice episodes** via `$pagination->getSlicedItems()` → `Episode[]` for the current page only
4. **Attach the provider** to the grid: `$grid->pagination()->setProvider($pagination)`
5. **Iterate only the sliced episodes** when building grid rows — identical row construction logic, just over fewer items
6. **Remove the custom form actions HTML** below the grid and migrate to the grid's built-in `GridActions` system (the grid already wraps everything in a `<form>`, using the built-in actions and selection column is cleaner and pagination-compatible)
7. **Preserve the `_init()` action handler** for `refresh-details` — it reads `$_POST['episodes']` which maps to the selection field

This is a minimal-change approach: only `EpisodeListPage::_render()` needs modification. No new classes, no new files, no new routes.

## Rationale

- **`ArrayPagination`** is the correct provider because the episode list is loaded entirely into memory (there is no database query to push `LIMIT`/`OFFSET` to). `ArrayPagination` handles the slicing and URL generation.
- **Custom page parameter `ep_page`**: Avoids the hard conflict with the application's `?page=` routing parameter. This is the simplest fix; the alternative (changing the app-wide routing parameter) would be far more disruptive.
- **Using the DataGrid's built-in actions/selection system**: The page currently manually renders checkboxes in the `id` column and a custom `<select>` below the grid. Migrating to `$grid->actions()` with a value column simplifies the template, keeps the `<form>` consistent with pagination URL parameters, and aligns with how the datagrids library is designed to work. However, this migration is **optional** — the existing manual checkboxes will continue to work within the grid's `<form>`, and the action handler reads `$_POST` directly. We can keep the current approach to minimize changes.
- **20 per page default**: Reasonable balance between scroll length and page loads for a local tool.

## Detailed Steps

### Step 1 — Modify `_render()` to use `ArrayPagination`

In `assets/classes/Page/EpisodeListPage.php`:

1. Add `use AppUtils\Grids\Pagination\Types\ArrayPagination;` import.
2. Before the grid column definitions, fetch all episodes:
   ```php
   $allEpisodes = $this->comic->createFetcher()->getAll();
   ```
3. Create the pagination provider with custom page parameter:
   ```php
   $pagination = new ArrayPagination($allEpisodes, 20, null, 'ep_page');
   ```
4. After creating the grid and defining columns, attach the provider:
   ```php
   $grid->pagination()->setProvider($pagination);
   ```
5. Replace the current loop `foreach($this->comic->createFetcher()->getAll() as $episode)` with:
   ```php
   foreach($pagination->getSlicedItems() as $episode)
   ```
   The `getSlicedItems()` returns `Episode[]` (same type as `getAll()`) because `ArrayPagination` stores the original array and slices it.

### Step 2 — Preserve the `grid-action` form handling

The `_init()` method reads `$this->request->getParam('grid-action')` and `$this->request->getParam('episodes')`. The grid's `<form>` wraps the entire table and the custom action HTML below it. Since pagination URLs are GET-based and the form action is POST-based, these do not interfere with each other.

The grid form's hidden vars (`page`, `comicID`) ensure the POST submission routes back to the correct page. **No changes needed here.**

### Step 3 — Preserve current page in redirect URLs after action handling

In `handleRefreshDetails()`, the success/error redirects use `$this->comic->getURLEpisodeList()`, which returns a URL without the `ep_page` parameter. To keep the user on the same pagination page after a refresh-details action, pass the current page parameter through:

```php
$params = [];
$epPage = $this->request->getParam('ep_page');
if($epPage !== null) {
    $params['ep_page'] = (int)$epPage;
}

$this->redirectWithSuccessMessage(
    'Details fetched successfully.',
    $this->comic->getURLEpisodeList($params)
);
```

Apply the same pattern to the error redirect. The `ep_page` value comes from the form's hidden vars or the current URL; since the grid's `<form>` action URL is the current page (which includes `ep_page` in the query string), the POST will carry it through.

### Step 4 — Add `ep_page` as a hidden form variable

To ensure the current page number survives form POST submissions (the form action URL may not always include query parameters depending on the browser), add it as a hidden var on the grid form:

```php
$grid->form()->addHiddenVar('ep_page', $pagination->getCurrentPage());
```

This guarantees `$this->request->getParam('ep_page')` is available in `handleRefreshDetails()` regardless of how the browser handles `<form action>` query strings.

### Step 5 — Verify hidden vars are compatible with pagination

The grid already calls:
```php
$grid->form()->addHiddenVar('page', self::PAGE_NAME);
$grid->form()->addHiddenVar('comicID', $this->comic->getID());
```

These are emitted as `<input type="hidden">` inside the grid's `<form>`. Pagination links are regular `<a href>` elements (GET), so these hidden vars only affect form submission (POST). **No conflict.**

## Dependencies

- `mistralys/application-datagrids` (already a `dev-main` dependency in `composer.json`) — must include the pagination feature (`ArrayPagination`, `GridPagination`, renderer support)

## Required Components

- `assets/classes/Page/EpisodeListPage.php` — the only file that needs code changes

## Assumptions

- The `application-datagrids` library's pagination feature is fully functional and available in the `dev-main` branch currently installed
- `ArrayPagination` correctly handles arrays of `Episode` objects (it stores `array<mixed>` and slices without type constraints)
- `ArrayPagination::getPageURL()` correctly preserves the existing query string parameters (`page=EpisodeList`, `comicID=...`) when generating pagination links
- Comics in the collection have varying episode counts (from single-digit to hundreds), making pagination a meaningful UX improvement

## Constraints

- The `page` query parameter is reserved for application routing — pagination **must not** use `page` as its parameter name
- No database — all episodes are loaded into memory; `ArrayPagination` slices them in PHP
- No breaking changes to the existing "refresh details" form action workflow
- No changes to URL routes or other pages

## Out of Scope

- Migrating the manual checkbox/action system to the DataGrid's built-in `GridActions` API (can be done as a follow-up)
- Making the items-per-page count user-configurable (e.g., via a dropdown)
- Sorting support on the episode list columns
- Persistent pagination state across separate visits (the page resets to 1 on each fresh visit unless `ep_page` is in the URL)

## Acceptance Criteria

- The episode list renders at most 20 episodes per page by default
- Pagination controls appear in the grid footer when a comic has more than 20 episodes
- Pagination controls do not appear when a comic has 20 or fewer episodes
- Clicking pagination links navigates to the correct page slice while preserving `page=EpisodeList` and `comicID` in the URL
- The "refresh details" form action continues to work for selected episodes on the current page
- After a successful or failed "refresh details" action, the user is redirected back to the same pagination page they were on
- The page loads without errors for comics with 0 episodes, 1 episode, exactly 20, and more than 20

## Testing Strategy

- **Manual testing** against several comics with different episode counts:
  - A comic with 0 fetched episodes (empty grid, no pagination)
  - A comic with < 20 episodes (full list, no pagination)
  - A comic with exactly 20 episodes (full list, no pagination)
  - A comic with > 20 episodes (paginated, verify navigation)
  - A comic with 100+ episodes (verify multi-page navigation, ellipsis rendering)
- **Verify URL structure**: Confirm pagination URLs include `ep_page=N` alongside `page=EpisodeList` and `comicID=...`
- **Test form submission**: Select episodes on a paginated page, run "refresh details", verify it processes correctly and redirects back
- **PHPStan**: Run `phpstan` to ensure no new static analysis errors are introduced

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`page` parameter conflict** breaks routing | Use `ep_page` as the pagination parameter name; verify in manual testing that app routing is unaffected |
| **`ArrayPagination::getSlicedItems()` returns index-shifted array** | `array_slice` preserves values but resets integer keys; the iteration uses `foreach` on objects, so key shifts are irrelevant |
| **Pagination URLs lose `comicID` or `page`** | `ArrayPagination::getPageURL()` rewrites `$_SERVER['REQUEST_URI']` which includes all current query parameters; verified in source code |
| **Form POST with pagination**: browser may strip query-string params from the `<form action>` URL | `ep_page` is added as a hidden form variable, ensuring it is always available in `$_POST` for the redirect after the action |
