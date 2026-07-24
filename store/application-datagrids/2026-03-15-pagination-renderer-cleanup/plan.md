# Plan

## Summary

Bundle four low-priority gold nuggets from the `2026-03-14-items-per-page-selector-rework-1` synthesis into a single cleanup/refactor cycle. The anchor item extracts a `createPaginationNavContent()` hook to eliminate duplication between `BaseGridRenderer` and `Bootstrap5Renderer`. Three micro-fixes ride along: GridSettings self-healing write-back on stale IPP detection, explicit `setItemsPerPageURLTemplate()` in `RendererPaginationTest` setup, and removal of a redundant `?? $fallback` null-coalescing tail.

## Architectural Context

### Pagination rendering (current state)

Two renderer classes contain pagination row logic:

- [src/Grids/Renderer/BaseGridRenderer.php](src/Grids/Renderer/BaseGridRenderer.php) — `createPaginationRow()` builds a `<tr><td>` containing up to three blocks: **nav links**, **page jump input**, and **IPP selector**. The nav block uses generic `<nav>` + `<span>`/`<a>` elements.
- [src/Grids/Renderer/Types/Bootstrap5Renderer.php](src/Grids/Renderer/Types/Bootstrap5Renderer.php) — overrides `createPaginationRow()` with an identical 3-block skeleton, but the nav block builds a `<nav><ul class="pagination"><li>` structure using Bootstrap-specific helper methods.

Both methods share identical structure:

```
1. Create <td> with colspan
2. If totalPages > 1:
   a. Build nav content (ONLY THIS DIFFERS)
   b. Append to <td>
   c. If page jump enabled → append page jump input
3. If IPP options → append IPP selector
4. Wrap <td> in <tr> and return
```

The only divergence is step 2a — the nav content markup. Everything else (the conditional skeleton, page jump, IPP selector, `<tr>` wrapping) is duplicated.

### Items-per-page resolution

[src/Grids/Pagination/GridPagination.php](src/Grids/Pagination/GridPagination.php) — `resolveItemsPerPage()` handles a 3-tier priority chain: `$_GET` → `GridSettings` → default. When a stale persisted value is detected (no longer in the options whitelist), it falls back to the default but does not write the corrected value back to storage.

### GridSettings contract

[src/Grids/Settings/GridSettings.php](src/Grids/Settings/GridSettings.php) — `getItemsPerPage(?int $default): ?int` returns `$default` when nothing is stored (via storage fallback). `setItemsPerPage()` enforces positive integers, so `null` can never be persisted. This makes the `?? $fallback` tail in `resolveItemsPerPage()` unreachable.

### Test setup

[tests/Renderer/RendererPaginationTest.php](tests/Renderer/RendererPaginationTest.php) — IPP tests call `setItemsPerPageOptions()` but never call `setItemsPerPageURLTemplate()`, causing the JS in the rendered `<select>` to contain `null.replace(...)`. Harmless in unit tests but silently incomplete setup.

## Approach / Architecture

### Step A — Extract `createPaginationNavContent()` hook

Introduce a `protected` method `createPaginationNavContent(GridPagination): ?HTMLTag` in `BaseGridRenderer` that encapsulates the default nav-links block (the `<nav>` with `<span>`/`<a>` page links, previous/next links). Return `null` when `totalPages <= 1`.

Refactor `createPaginationRow()` in `BaseGridRenderer` to call this hook instead of inlining the nav construction. The page-jump and IPP-selector blocks remain in `createPaginationRow()`.

Override `createPaginationNavContent()` in `Bootstrap5Renderer` to produce the Bootstrap `<nav><ul class="pagination">...</ul></nav>` markup. Delete the `createPaginationRow()` override in `Bootstrap5Renderer` entirely — it inherits the base skeleton.

### Step B — GridSettings self-healing write-back

In `GridPagination::resolveItemsPerPage()`, after the stale-value detection branch falls through to `$fallback`, call `$this->grid->settings()->setItemsPerPage($fallback)` to persist the corrected value. This eliminates repeated whitelist-miss cycles on subsequent requests.

### Step C — Test setup completeness

In `RendererPaginationTest`, add `$pagination->setItemsPerPageURLTemplate('/items?ipp={IPP}')` to the IPP test helper setup (or directly in each IPP test) so the rendered `onchange` JS contains a real URL template instead of `null`.

### Step D — Remove `?? $fallback` redundancy

In `GridPagination::resolveItemsPerPage()`, change:

```php
$storedValue = $this->grid->settings()->getItemsPerPage($fallback) ?? $fallback;
```

to:

```php
$storedValue = $this->grid->settings()->getItemsPerPage($fallback);
```

Since `getItemsPerPage($fallback)` returns `$fallback` (non-null int) when nothing is stored, and `setItemsPerPage()` prevents persisting `null`, the `?? $fallback` tail is unreachable. However, this depends on the `?int` return type — `getItemsPerPage` _can_ return `null` if `$default` is `null`. In our call site `$fallback` is always non-null, so the coalescing is dead code. To make this fully safe, also narrow the return type of the internal call by asserting: cast the result or keep the coalescing. **Recommendation:** simply remove the `?? $fallback` since the preceding whitelist check already handles the fallback path. Add a brief inline comment documenting why `null` is not possible here.

## Rationale

- **Step A** is the core value: it removes a copy-paste override and makes adding a third renderer trivial. The Template Method pattern is already the project convention (per the previous WP-001 fix); this completes the pattern for pagination nav content.
- **Steps B–D** are near-zero-risk one-liners that were explicitly flagged by the reviewer in cycle 1. Bundling them avoids the overhead of a separate plan for each.
- The existing `RendererPaginationTest` suite (12 tests) provides regression coverage for Step A. Steps B–D each have targeted test assertions already in place or need minor additions.

## Detailed Steps

1. **Create `createPaginationNavContent()` in `BaseGridRenderer`.**
   - Extract the nav-links block from `createPaginationRow()` into a new `protected function createPaginationNavContent(GridPagination $pagination): ?HTMLTag`.
   - When `$pagination->getTotalPages() <= 1`, return `null`.
   - Otherwise, build and return the `<nav>` element containing: previous link, page number links (with ellipsis), next link.
   - Update `createPaginationRow()` to call `createPaginationNavContent()` and conditionally append the result + page-jump input.

2. **Override `createPaginationNavContent()` in `Bootstrap5Renderer`.**
   - Move the Bootstrap nav-links logic from `Bootstrap5Renderer::createPaginationRow()` into `protected function createPaginationNavContent(GridPagination $pagination): ?HTMLTag`.
   - When `$pagination->getTotalPages() <= 1`, return `null`.
   - Otherwise, build and return the `<nav aria-label="Page navigation"><ul class="pagination">...</ul></nav>`.
   - **Delete** the `createPaginationRow()` override from `Bootstrap5Renderer` — it now inherits cleanly from the base class.

3. **Add self-healing write-back in `resolveItemsPerPage()`.**
   - In `GridPagination::resolveItemsPerPage()`, after the stale-detection branch sets `$storedValue = $fallback`, add: `$this->grid->settings()->setItemsPerPage($fallback);`

4. **Remove `?? $fallback` redundancy in `resolveItemsPerPage()`.**
   - Change `$this->grid->settings()->getItemsPerPage($fallback) ?? $fallback` to `$this->grid->settings()->getItemsPerPage($fallback)`.
   - Add a brief comment: `// $fallback is non-null, so getItemsPerPage() always returns a non-null int`

5. **Add `setItemsPerPageURLTemplate()` to `RendererPaginationTest` IPP tests.**
   - In each of the following test methods, add `$pagination->setItemsPerPageURLTemplate('/items?ipp={IPP}')` after the `setItemsPerPageOptions()` call:
     - `test_defaultRenderer_singlePageWithIpp_hasSelectNoNav`
     - `test_bootstrap5Renderer_singlePageWithIpp_hasSelectNoNav`
     - `test_defaultRenderer_multiPageWithIpp_hasBothNavAndSelect`
     - `test_bootstrap5Renderer_multiPageWithIpp_hasBothPaginationAndSelect`
     - `test_defaultRenderer_ippSelect_hasAriaLabel`
     - `test_bootstrap5Renderer_ippSelect_hasAriaLabel`

6. **Run full test suite** — `composer test` — verify 126 tests pass, 0 failures.

7. **Run PHPStan** — `composer analyze` — verify 0 errors at level 6.

8. **Update manifest documents:**
   - `api-surface.md` — Add `createPaginationNavContent()` to `BaseGridRenderer` and `Bootstrap5Renderer`; remove `createPaginationRow()` from `Bootstrap5Renderer`.
   - `data-flows.md` — Update the rendering pipeline section 8 to reflect the new hook.
   - `constraints.md` — Remove `?? $fallback` from any known-debt tracking if listed; note the self-healing behavior.

## Dependencies

- Steps 1–2 are tightly coupled (the `createPaginationRow()` refactor and `Bootstrap5Renderer` override deletion depend on the new hook existing).
- Steps 3–5 are independent of each other and of Steps 1–2.
- Steps 6–7 (validation) depend on all code changes being complete.
- Step 8 (documentation) depends on all code changes being finalized.

## Required Components

### Modified files
- `src/Grids/Renderer/BaseGridRenderer.php` — new `createPaginationNavContent()` method; refactored `createPaginationRow()`
- `src/Grids/Renderer/Types/Bootstrap5Renderer.php` — new `createPaginationNavContent()` override; `createPaginationRow()` override **deleted**
- `src/Grids/Pagination/GridPagination.php` — `resolveItemsPerPage()` modified (self-healing + redundancy removal)
- `tests/Renderer/RendererPaginationTest.php` — IPP test methods updated with URL template setup

### Documentation files
- `docs/agents/project-manifest/api-surface.md`
- `docs/agents/project-manifest/data-flows.md`
- `docs/agents/project-manifest/constraints.md`

### No new files created.

## Assumptions

- The `createPageJumpInput()` method is shared between both renderers and does not need extraction (confirmed: `Bootstrap5Renderer` overrides it separately if needed, but both call it from the same position in the skeleton).
- `Bootstrap5Renderer::createItemsPerPageSelector()` override remains unchanged — it wraps the base selector in a Bootstrap `<div>` and is not part of the nav-content hook.
- The `?? $fallback` removal is safe because `GridSettings::setItemsPerPage()` enforces positive integers and `getItemsPerPage($fallback)` returns `$fallback` when nothing is stored. No code path can persist `null` to the `items_per_page` key.

## Constraints

- All HTML must be generated via `HTMLTag` — no string concatenation (per project conventions).
- `declare(strict_types=1);` required in all PHP files.
- Fluent API: new methods returning `self`/`$this` where applicable.
- `composer dump-autoload` is **not** needed — no files are added, renamed, or moved.

## Out of Scope

- Adding a third renderer type (this plan _enables_ it but does not implement one).
- Modifying the `renderPaginationRow()` guard logic or the `createItemsPerPageSelector()` method.
- Changing the IPP resolution priority chain beyond the self-healing write-back.
- Tagging a release or publishing to Packagist (operational task, not code).
- Running `composer update` in `webcomics-builder` (separate operational task after release tagging).

## Acceptance Criteria

- `Bootstrap5Renderer` no longer overrides `createPaginationRow()`.
- Both renderers produce identical HTML output before and after the refactor (verified by existing `RendererPaginationTest` suite).
- `createPaginationNavContent()` exists as a `protected` method on `BaseGridRenderer` and is overridden in `Bootstrap5Renderer`.
- Stale IPP values are written back to GridSettings storage on detection (verifiable via a new or extended test).
- `RendererPaginationTest` IPP tests include `setItemsPerPageURLTemplate()` — rendered output contains a real URL pattern instead of `null`.
- The `?? $fallback` coalescing tail is removed from `resolveItemsPerPage()`.
- `composer test` — all tests pass (≥ 126 tests, 0 failures).
- `composer analyze` — 0 errors at level 6.

## Testing Strategy

- **Regression:** The existing 12-method `RendererPaginationTest` suite covers all 5 combinatorial pagination states × 2 renderers. After the Step A refactor, all tests must produce identical results — same assertions, same HTML structure.
- **Self-healing (Step B):** Extend the existing stale-settings test in `GridPaginationTest` (or add a new method) to verify that after `resolveItemsPerPage()` detects a stale value, the corrected value is persisted to `GridSettings`.
- **URL template (Step C):** After adding `setItemsPerPageURLTemplate()`, optionally add an assertion in the IPP tests that the rendered `onchange` contains the expected URL pattern (e.g., `assertStringContainsString('/items?ipp=', $result)`).
- **Redundancy removal (Step D):** Covered by existing `resolveItemsPerPage` tests — behavior is unchanged.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Step A introduces subtle HTML differences** | Run the full `RendererPaginationTest` suite after refactoring. The tests assert exact structural markers (`<nav`, `class="pagination"`, `<select`, `aria-label`). Any regression will surface immediately. |
| **Self-healing write-back causes unexpected persistence** | The write-back only triggers when a stale value is detected (value not in current whitelist). This is the same code path that already falls back silently — persisting the corrected value is strictly more correct. Add a targeted test to verify the behavior. |
| **`?? $fallback` removal breaks edge case** | Reviewed `GridSettings::getItemsPerPage()` — returns `$default` when nothing stored, and `setItemsPerPage()` prevents persisting `null`. The `$fallback` argument is always non-null in this call site. Risk is near-zero; existing tests cover the resolution chain. |
