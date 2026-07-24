# Plan — Items-per-Page Selector Rework

## Summary

Address all actionable findings from the `2026-03-14-items-per-page-selector` synthesis report. Six items were flagged (four medium, two low priority). This rework covers the five that target the `application-datagrids` library codebase; the sixth (dev workflow pre-commit guard) is deferred as non-actionable in this context.

The work resolves a Template Method violation in `Bootstrap5Renderer`, adds a stale-value guard to `resolveItemsPerPage()`, creates renderer-level pagination tests, adds an `aria-label` for accessibility, strengthens GET parameter validation, and removes the temporary path repository from `webcomics-builder/composer.json`.

## Architectural Context

- **Renderer hierarchy:** `GridRendererInterface` → `BaseGridRenderer` (abstract) → `DefaultRenderer` / `Bootstrap5Renderer`. The base class follows the **Template Method** pattern: `renderPaginationRow()` handles guards then delegates to `createPaginationRow()`. Subclasses should override the `create*` hooks, not the top-level `render*` methods.
- **Items-per-page resolution:** `GridPagination::resolveItemsPerPage()` implements a three-level priority chain (`$_GET` → `GridSettings` → default). The result is cached in `$resolvedItemsPerPage` for the object's lifetime.
- **Test infrastructure:** PHPUnit 12, test suites in `tests/` subdirectories, `InMemoryStorage` helper in `tests/TestClasses/`. No renderer-level tests currently exist.
- **HTML generation:** All HTML output uses `HTMLTag` from `application-utils-core` — never raw string concatenation.
- **Coding conventions:** `declare(strict_types=1)`, fluent setters returning `self`/`$this`, trait+interface pairing, classmap autoloading (requires `composer dump-autoload` after adding files).

### Key files

| File | Role |
|---|---|
| `src/Grids/Renderer/BaseGridRenderer.php` | Abstract renderer with Template Method pagination |
| `src/Grids/Renderer/Types/Bootstrap5Renderer.php` | Bootstrap 5 concrete renderer |
| `src/Grids/Pagination/GridPagination.php` | Pagination state, IPP resolution, URL templates |
| `tests/Pagination/GridPaginationTest.php` | Existing 36 pagination tests |
| `tests/TestClasses/InMemoryStorage.php` | Zero-I/O storage for tests |
| `webcomics-builder/composer.json` | Contains temporary path repository block |

## Approach / Architecture

Each finding maps to one discrete, independently deliverable step. The steps are sequenced to put the structural refactor first (so subsequent work builds on the clean hierarchy), then correctness fixes, then coverage, then polish.

1. **Template Method refactor** — Eliminate `Bootstrap5Renderer::renderPaginationRow()` entirely. Move `createBootstrapPaginationRow()` logic into a `createPaginationRow()` override, letting the base class `renderPaginationRow()` handle all guard logic.
2. **IPP settings re-validation** — Add an `in_array` whitelist check in `resolveItemsPerPage()` on the value returned from `GridSettings`, falling through to the default when the stored value is no longer in the current options list.
3. **Renderer pagination tests** — Create a new `RendererPaginationTest` class that tests the HTML output of both `BaseGridRenderer` (via `DefaultRenderer`) and `Bootstrap5Renderer` under all combinatorial pagination states.
4. **Accessibility: aria-label** — Add `aria-label="Items per page"` to the `<select>` element built in `BaseGridRenderer::createItemsPerPageSelector()` (inherited by Bootstrap5).
5. **Strict GET validation** — Replace `(int)$_GET[$this->ippParam]` with `filter_var(..., FILTER_VALIDATE_INT)` and treat `false` as invalid.
6. **Housekeeping: remove path repository** — Remove the `"repositories"` block and `"_repositories_comment"` key from `webcomics-builder/composer.json`.

## Rationale

- **Step 1 first:** The refactor changes the method that steps 3 and 4 test/modify. Doing it first avoids double-work.
- **`in_array` guard (step 2):** A settings file can outlive an options-list change. Without re-validation, the selector renders with no `selected` option — a silent UI bug.
- **`filter_var` (step 5):** While the whitelist already blocks exploitation, `(int)` casting silently truncates strings like `'20abc'` to `20`, which could match a valid option. `filter_var` rejects malformed input outright, producing cleaner semantics.
- **Test class (step 3):** Renderer output is currently verified only by manual inspection. Snapshot-style assertions prevent regressions when renderer code changes.
- **aria-label (step 4):** Applied at the base class level so all renderers inherit it.

## Detailed Steps

### Step 1 — Refactor Bootstrap5Renderer pagination (Template Method fix)

**File:** `src/Grids/Renderer/Types/Bootstrap5Renderer.php`

1. **Delete** the `renderPaginationRow()` method override entirely (lines 71–81). The base class `BaseGridRenderer::renderPaginationRow()` already contains identical guard logic and calls `createPaginationRow()`.
2. **Rename** the private method `createBootstrapPaginationRow(GridPagination)` to `createPaginationRow(GridPagination)` and change its visibility from `private` to `protected` so it correctly overrides the base class hook.
3. **Verify** that `DefaultRenderer` does not override `renderPaginationRow()` (it doesn't — confirmed via api-surface.md).
4. Run `composer analyze` and `composer test` to confirm no regressions.

### Step 2 — Re-validate GridSettings IPP value against current options

**File:** `src/Grids/Pagination/GridPagination.php`, method `resolveItemsPerPage()`

1. After the Priority 2 comment (`// Priority 2: GridSettings`), retrieve the stored value from `GridSettings::getItemsPerPage()`.
2. Before assigning to `$this->resolvedItemsPerPage`, check whether the stored value is in `$this->itemsPerPageOptions` using `in_array($storedValue, $this->itemsPerPageOptions, true)`.
3. If the stored value is **not** in the whitelist, fall through to `$fallback`.
4. Add a test in `GridPaginationTest` that:
   - Configures options `[10, 25, 50]`, manually writes value `100` to `GridSettings`, then asserts `resolveItemsPerPage()` returns the default (25), not 100.

### Step 3 — Create RendererPaginationTest

**New file:** `tests/Renderer/RendererPaginationTest.php`

Create a test class covering these combinatorial states for **both** `DefaultRenderer` and `Bootstrap5Renderer`:

| # | State | Expected behavior |
|---|---|---|
| 1 | No provider set | `renderPaginationRow()` returns `''` |
| 2 | Provider set, totalPages ≤ 1, no IPP options | Returns `''` |
| 3 | Provider set, totalPages ≤ 1, IPP options configured | Returns non-empty HTML containing `<select>` but no `<nav>` / `<ul class="pagination">` |
| 4 | Provider set, totalPages > 1, no IPP options | Returns HTML with page navigation but no `<select>` |
| 5 | Provider set, totalPages > 1, IPP options configured | Returns HTML with both page navigation and `<select>` |

Each test should:
- Create a `DataGrid` with `InMemoryStorage`.
- Set up a minimal `PaginationInterface` stub (anonymous class, like existing tests).
- Call `renderPaginationRow()` directly on the renderer instance.
- Assert against the rendered HTML string using `assertStringContainsString` / `assertStringNotContainsString` / `assertEmpty`.

Also register a new test suite `Renderer` in `phpunit.xml`.

After creating the file, run `composer dump-autoload` (classmap) then `composer test`.

### Step 4 — Add aria-label to IPP select element

**File:** `src/Grids/Renderer/BaseGridRenderer.php`, method `createItemsPerPageSelector()`

1. After the `$select = HTMLTag::create('select')` line, add `->attr('aria-label', 'Items per page')`.
2. This propagates to `Bootstrap5Renderer` since it calls `parent::createItemsPerPageSelector()`.
3. Add an assertion in `RendererPaginationTest` (step 3) that the rendered HTML contains `aria-label="Items per page"`.

### Step 5 — Replace (int) cast with filter_var for GET parameter

**File:** `src/Grids/Pagination/GridPagination.php`, method `resolveItemsPerPage()`

1. Replace:
   ```php
   $value = (int)$_GET[$this->ippParam];
   ```
   with:
   ```php
   $value = filter_var($_GET[$this->ippParam], FILTER_VALIDATE_INT);
   ```
2. Add a `$value !== false` guard before the `in_array` check (since `filter_var` returns `false` for invalid integers).
3. Add a test in `GridPaginationTest` that sets `$_GET['ipp'] = '20abc'` with option `20` in the whitelist, and asserts `resolveItemsPerPage()` does **not** return 20 (falls through to default). This verifies the stricter validation rejects truncated-cast matches.

### Step 6 — Remove path repository from webcomics-builder

**File:** `webcomics-builder/composer.json`

1. Remove the `"_repositories_comment"` key (line 4).
2. Remove the entire `"repositories"` array (lines 5–11).
3. Run `composer update --no-install` in the webcomics-builder directory to confirm clean resolution.

## Dependencies

- Step 1 must complete before Step 3 (tests should target the refactored method structure).
- Step 4 should complete before or alongside Step 3 (so the aria-label assertion can be included).
- Steps 2 and 5 are independent of each other but both target `resolveItemsPerPage()` — apply sequentially.
- Step 6 is fully independent.

Sequencing: **1 → 4 → 5 → 2 → 3 → 6**

## Required Components

| Component | Status | Location |
|---|---|---|
| `Bootstrap5Renderer` | Existing — modify | `src/Grids/Renderer/Types/Bootstrap5Renderer.php` |
| `BaseGridRenderer` | Existing — modify | `src/Grids/Renderer/BaseGridRenderer.php` |
| `GridPagination` | Existing — modify | `src/Grids/Pagination/GridPagination.php` |
| `GridPaginationTest` | Existing — extend | `tests/Pagination/GridPaginationTest.php` |
| `RendererPaginationTest` | **New** | `tests/Renderer/RendererPaginationTest.php` |
| `InMemoryStorage` | Existing — reuse | `tests/TestClasses/InMemoryStorage.php` |
| `phpunit.xml` | Existing — add suite | `phpunit.xml` |
| `webcomics-builder/composer.json` | Existing — modify | `../personal/webcomics-builder/composer.json` |

## Assumptions

- The existing anonymous `PaginationInterface` stub pattern from `GridPaginationTest` can be reused in `RendererPaginationTest`.
- `HTMLTag::render()` or `(string)$tag` produces the full HTML string needed for assertions.
- `InMemoryStorage` supports writing arbitrary keys (confirmed by its use in `GridSettingsTest`).
- The webcomics-builder path repository is not needed by any other active development branch.

## Constraints

- All new files must include `declare(strict_types=1)`.
- All new setter methods must return `self`/`$this` (fluent API).
- HTML output must use `HTMLTag` — no raw string concatenation.
- `composer dump-autoload` must be run after adding `RendererPaginationTest`.
- PHPStan level 6 must pass with 0 errors after every step.
- The existing 112 tests must continue to pass throughout.

## Out of Scope

- **Synthesis item 6 (pre-commit guard for path repository):** This is a developer workflow concern outside the library codebase. Deferred.
- **Localization of "Items per page" / aria-label strings:** The existing implementation uses hardcoded English strings; localization is a separate concern.
- **Additional renderer types** (e.g., testing `DefaultRenderer::createItemsPerPageSelector()` independently) — covered as part of step 3's dual-renderer test matrix.
- **Changing the `GridSettings` storage key format** — out of scope; existing format works correctly.

## Acceptance Criteria

1. `Bootstrap5Renderer` no longer overrides `renderPaginationRow()`. The base class guard is the single source of truth.
2. `resolveItemsPerPage()` returns the default when `GridSettings` contains a value not in the current options list.
3. `resolveItemsPerPage()` rejects `$_GET` values like `'20abc'` even when `20` is a valid option.
4. The IPP `<select>` element includes `aria-label="Items per page"` in all renderers.
5. A new `RendererPaginationTest` class covers at least 5 combinatorial states × 2 renderers = 10 test methods.
6. `webcomics-builder/composer.json` contains neither `"repositories"` nor `"_repositories_comment"`.
7. PHPStan level 6: 0 errors. All tests pass (target: ~122+ tests).

## Testing Strategy

| Area | Approach |
|---|---|
| Template Method refactor (step 1) | Existing 112 tests + new `RendererPaginationTest` confirming Bootstrap5 pagination output is unchanged |
| IPP re-validation (step 2) | New test in `GridPaginationTest`: stale settings value falls back to default |
| Renderer pagination HTML (step 3) | New `RendererPaginationTest` with 10+ test methods covering all combinatorial states |
| aria-label (step 4) | Assertion in `RendererPaginationTest` that HTML contains `aria-label="Items per page"` |
| Strict GET validation (step 5) | New test in `GridPaginationTest`: `'20abc'` with option 20 returns default |
| Path repository removal (step 6) | Manual: `composer update --no-install` succeeds in webcomics-builder |

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **Template Method refactor breaks Bootstrap5 pagination output** | The logic in `createBootstrapPaginationRow()` is unchanged — only the method name, visibility, and call path change. Full test coverage (step 3) validates output. |
| **`filter_var` changes behavior for edge-case GET values** | The whitelist check is the true safety net. `filter_var` only rejects values that `(int)` would silently truncate — strictly safer. |
| **Stale settings re-validation changes existing behavior** | Only affects the case where stored value ∉ current options — previously a silent UI bug (no `selected` option). Now falls back to default, which is the correct behavior. |
| **RendererPaginationTest is brittle against HTML structure changes** | Use `assertStringContainsString` on semantic markers (tag names, class names, aria attributes) rather than exact HTML snapshots. |
