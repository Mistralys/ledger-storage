# Plan

## Summary

Add two features to the comic list pages: (1) a full-text search filter that matches against comic label and synopsis, integrated into the existing filter bar used by both the card list and detailed list pages; (2) pagination for the detailed list page using `ArrayPagination` from the `application-datagrids` package, following the same patterns established by the episode list page.

## Architectural Context

### Filter System
- `ComicsFilter` ([assets/classes/Comics/Filter/ComicsFilter.php](assets/classes/Comics/Filter/ComicsFilter.php)) — stateless filter with `selectStatus()`, `setOrderBy()`, `setOrderDir()`, and `getMatches()`. The `isMatch()` method applies per-comic filtering; `getMatches()` also handles sorting. Currently only filters by status.
- `SessionComicsFilter` ([assets/classes/Comics/Filter/SessionComicsFilter.php](assets/classes/Comics/Filter/SessionComicsFilter.php)) — extends `ComicsFilter` with session persistence. Imports/exports values via `ArrayDataCollection`. Stores state in `Session::set('comicsFilter', ...)`.

### Page Hierarchy
- `DetailedListPage` ([assets/classes/Page/DetailedListPage.php](assets/classes/Page/DetailedListPage.php)) — renders the filter bar in `displayFilters()` and an HTML table of comics in `generateComicList()`. The `_render()` method calls both and is shared by its subclass.
- `CardListPage` ([assets/classes/Page/CardListPage.php](assets/classes/Page/CardListPage.php)) — extends `DetailedListPage`, overrides only `generateComicList()` to render cards instead of a table. Inherits `_render()` and `displayFilters()` from parent.

### Pagination Pattern
- `EpisodeListPage` ([assets/classes/Page/EpisodeListPage.php](assets/classes/Page/EpisodeListPage.php)) — established pagination pattern using `ArrayPagination` with custom `ep_page` parameter and `DataGrid` for rendering.
- Convention: custom page parameter name (`<context>_page`) to avoid conflict with the `?page=PageID` router parameter.
- Default page size: 20 items per page.
- `ArrayPagination` (from `application-datagrids`, `AppUtils\Grids\Pagination\Types`) — accepts full array, items-per-page, optional current page, and custom query parameter. Reads `$_GET[$pageParam]` automatically. Provides `getSlicedItems()`, `getPageURL(int $page)`, `getCurrentPage()`, `getTotalItems()`.

### Comic Entity
- `Comic::getLabel(): string` — the comic's display name.
- `Comic::getSynopsis(): string` — the comic's description/synopsis.
- Both are available in the filtered `Comic[]` array returned by `ComicsFilter::getMatches()`.

## Approach / Architecture

### Feature 1: Full-Text Search Filter

Add a search text property to `ComicsFilter` with session persistence via `SessionComicsFilter`. The search performs a case-insensitive substring match against the comic's label and synopsis within `ComicsFilter::isMatch()`. A text input is added to the filter form in `DetailedListPage::displayFilters()`, making it available on both the card list and detailed list pages.

### Feature 2: Detailed List Pagination

Add `ArrayPagination` to `DetailedListPage::_render()` with a `cl_page` custom parameter (following the `<context>_page` convention). Since `CardListPage` inherits `_render()` from `DetailedListPage`, both pages get pagination automatically with zero code duplication. Pagination controls are rendered as standard Bootstrap 5 markup after the comic list, using `ArrayPagination::getPageURL()` for link generation.

## Rationale

- **Search in `ComicsFilter::isMatch()`**: This is the natural extension point — all filtering logic lives here, keeping the filter composable and testable in isolation.
- **Session persistence**: Matches the existing pattern where all filter state is saved to the session and restored on page load.
- **Centralized pagination in `_render()`**: Since `CardListPage` inherits `_render()` from `DetailedListPage`, both pages get pagination automatically with zero code duplication. No guard method needed.
- **Manual Bootstrap 5 pagination HTML**: The project uses inline PHP rendering (no template engine). The DataGrid's built-in pagination renderer requires a full `DataGrid` instance. Since the detailed list uses a manual HTML table (not DataGrid), rendering pagination controls directly is simpler and avoids creating a throwaway DataGrid instance. The HTML follows standard Bootstrap 5 pagination markup.
- **`cl_page` parameter name**: Follows the established `<context>_page` convention from `constraints.md` to avoid conflict with the `?page=` router parameter.
- **20 items per page**: Matches the default established by `EpisodeListPage`.

## Detailed Steps

### Step 1: Add search text property to `ComicsFilter`

**File:** `assets/classes/Comics/Filter/ComicsFilter.php`

1. Add constant `FILTER_SEARCH = 'search'`.
2. Add property `protected string $searchText = ''`.
3. Add method `setSearchText(string $text): self` that trims and stores the value.
4. Update `reset()` to clear `$searchText` to `''`.
5. Update `isMatch(Comic $comic): bool` to return `false` when `$searchText` is non-empty and neither `$comic->getLabel()` nor `$comic->getSynopsis()` contain the search text (case-insensitive via `mb_stripos()`).

### Step 2: Add search text persistence to `SessionComicsFilter`

**File:** `assets/classes/Comics/Filter/SessionComicsFilter.php`

1. Update `importValues()` to call `$this->setSearchText($values->getString(self::FILTER_SEARCH))` (note: `FILTER_SEARCH` is inherited from `ComicsFilter`).
2. Update `getValues()` to include `self::FILTER_SEARCH => $this->searchText` in the returned array.

### Step 3: Add search text input to the filter form

**File:** `assets/classes/Page/DetailedListPage.php`

1. In `displayFilters()`, add a text input field for the search text before the status dropdown. The field uses `ComicsFilter::FILTER_SEARCH` as its `name` attribute, with the current value from `$values->getString(ComicsFilter::FILTER_SEARCH)`.
2. Add a placeholder like `t('Search...')` for the input.

### Step 4: Add pagination to `DetailedListPage::_render()`

**File:** `assets/classes/Page/DetailedListPage.php`

1. Add import for `AppUtils\Grids\Pagination\Types\ArrayPagination`.
2. In `_render()`, after getting `$comics = $this->filters->getMatches()`:
   - Create `$pagination = new ArrayPagination($comics, 20, null, 'cl_page')`.
   - Pass `$pagination->getSlicedItems()` to `generateComicList()` instead of the full array.
   - After `generateComicList()`, call a new method `renderPagination($pagination)`.
   - Since `CardListPage` inherits `_render()`, both pages get pagination automatically.

### Step 5: Render Bootstrap 5 pagination controls

**File:** `assets/classes/Page/DetailedListPage.php`

1. Add private method `renderPagination(ArrayPagination $pagination): void`.
2. Calculate total pages: `ceil(totalItems / itemsPerPage)`. If total pages ≤ 1, return early (no controls needed).
3. Render a `<nav aria-label="..."><ul class="pagination justify-content-center">` with:
   - Previous page link (disabled when on page 1).
   - Page number links with `active` class on the current page. For large page counts, show ellipsis gaps (e.g., 1 2 ... 5 6 7 ... 10 11). A simple windowed approach: show first 2, last 2, and 2 around the current page.
   - Next page link (disabled when on last page).
4. Each page link uses `$pagination->getPageURL($pageNumber)`.

## Dependencies

- `AppUtils\Grids\Pagination\Types\ArrayPagination` from `mistralys/application-datagrids` (already a dependency).
- `mb_stripos()` PHP function (requires `mbstring` extension, standard in PHP 8.4).

## Required Components

### Modified Files
- `assets/classes/Comics/Filter/ComicsFilter.php` — search text property & matching logic (Steps 1)
- `assets/classes/Comics/Filter/SessionComicsFilter.php` — search text persistence (Step 2)
- `assets/classes/Page/DetailedListPage.php` — search input, pagination logic & rendering (Steps 3, 4, 5)

### Manifest Updates
- `docs/agents/project-manifest/api-surface.md` — new constant `FILTER_SEARCH`, new method `setSearchText()`
- `docs/agents/project-manifest/data-flows.md` — update "Comic List & Filtering" flow to mention search text and pagination
- `docs/agents/project-manifest/constraints.md` — document `cl_page` parameter name for comic list pagination

## Assumptions

- `mb_stripos()` is available (standard in PHP 8.4 with `mbstring`).
- Pagination applies to **both** card list and detailed list pages (centralized in `_render()`).
- The full-text search applies to **both** card list and detailed list pages (shared filter form).
- Search matches against comic label and synopsis only.
- 20 items per page matches the existing convention.

## Constraints

- Custom page parameter `cl_page` must be used (not `page`) per `constraints.md`.
- Filter form submission resets to page 1 (the redirect to `$this->getURL()` naturally omits `cl_page`).
- No database — filtering and pagination operate on in-memory arrays.
- No template engine — rendering is inline PHP with output buffering.

## Out of Scope

- Converting the detailed list table to use `DataGrid` (would be a separate refactoring task).
- Search by source type, alias, or other comic metadata beyond label and synopsis.
- Debounced/live search (JavaScript) — this is a form-submit search.
- Highlighting matched search terms in results.

## Acceptance Criteria

- [ ] A text input field appears in the filter bar on both card list and detailed list pages.
- [ ] Typing a search term and clicking "Filter" filters comics to those whose label or synopsis contains the search text (case-insensitive).
- [ ] The search text persists in the session across page loads (same as status and sort filters).
- [ ] The search text input shows the current search value after filtering.
- [ ] Clearing the search text and filtering shows all comics (matching other active filters).
- [ ] Both card list and detailed list pages show pagination controls when there are more than 20 matching comics.
- [ ] Pagination controls are hidden when there are 20 or fewer matching comics.
- [ ] Clicking a pagination link navigates to the correct page of results.
- [ ] Applying a new filter resets pagination to page 1.
- [ ] PHPStan analysis passes with zero new errors.

## Testing Strategy

- **Manual testing**: Navigate to both list pages, apply search terms, verify filtering works. Navigate pagination on the detailed list. Verify the card list has no pagination.
- **PHPStan**: Run `composer analyze` to confirm zero new static analysis errors.
- **Recommended unit tests** (future):
  - `ComicsFilter::isMatch()` with search text against mock comics with known labels and synopses.
  - `SessionComicsFilter` round-trip: import values with search text, export, verify persistence.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`mb_stripos()` not available** | PHP 8.4 includes `mbstring` by default; verify with `composer analyze`. |
| **Pagination URL conflicts** | Using `cl_page` custom parameter per established convention avoids the `?page=` router conflict. |
| **Pagination on card list** | Desired — both pages share the centralized `_render()` implementation. |
| **Search on large synopsis text is slow** | With ~50 comics, in-memory `mb_stripos()` is negligible. No performance concern at this scale. |
| **Filter reset doesn't clear pagination** | `applyFilters()` redirects to `$this->getURL()` which omits `cl_page`, naturally resetting to page 1. |
