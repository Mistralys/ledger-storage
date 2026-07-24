# Plan

## Summary

Five low-priority debt items were carried forward from the Jump To Episode feature (synthesis dated 2026-03-09). This rework plan addresses all of them in a single cleanup pass, touching only `js/viewer.js` and `assets/classes/Page/ReaderPage.php`. The changes are confined, mechanical, and require no new classes, no schema changes, and no autoload regeneration.

---

## Architectural Context

All debt items live in two files:

| File | Relevant context |
|---|---|
| `js/viewer.js` | Custom jQuery-based viewer class. No bundler; served directly. Has AJAX methods `SavePosition()`, `ToggleBookmark()`, `ToggleImageBookmark()`, `UseAsCover()`, `SetFlag()`, `JumpTo()`, and `ShowJumpToModal()`. |
| `assets/classes/Page/ReaderPage.php` | Reader page class. `renderImages()` (line 278) computes `$currentPageNumber` via `array_search()` and emits it into the `<script>` block and modal HTML. |

PHPStan level 5 with baseline must continue to produce 0 new errors. PHPUnit 60/60 must continue to pass (no PHP changes that affect tested logic are involved in this plan).

---

## Approach / Architecture

Four of the five items are pure JavaScript cleanup inside `viewer.js`. One is a PHP defensive hardening in `ReaderPage.php`. All changes are surgical line-level edits. No new files, no new classes, no dependency changes.

---

## Rationale

- **`'failure'` → `'error'` (jQuery AJAX):** `'failure'` is not a recognised jQuery AJAX callback key. The network-error handler silently never fires, leaving users without error feedback on network failures for four actions. The `'error'` key is the correct jQuery AJAX shorthand.  
- **`bootstrap.Modal.getOrCreateInstance()`:** Bootstrap's own documentation recommends this method over `new bootstrap.Modal()` for elements that may already have a Modal instance attached. Using `new bootstrap.Modal()` on repeat opens is harmless today but is non-idiomatic and could cause memory/event-listener leakage as Bootstrap version evolves.  
- **Dead `const viewer = this;` in `JumpTo()`:** Dead code: `viewer` is assigned but never referenced inside `JumpTo()`'s callbacks. Removing it eliminates lint noise and future reader confusion.  
- **Fragile Go button selector:** `$('#jumpToModal .btn-primary')` will break silently if a second `.btn-primary` is ever added to the modal footer. A dedicated `id="btn-jump-go"` eliminates that coupling.  
- **`$currentPageNumber = 0` fallback:** Showing "Page 0 of N" in the episode header degrades the UX when the edge case (episode not in fetched list) occurs, even though HTML5 validation prevents submission of `0`. Defaulting to `1` produces a coherent display in the unlikely fallback path.

---

## Detailed Steps

### Step 1 — Fix `'failure'` → `'error'` in all jQuery AJAX calls (`js/viewer.js`)

Four existing AJAX methods use `'failure'` as the network-error callback key:

- `SetFlag()` — line 76
- `SavePosition()` — line 176
- `ToggleBookmark()` — line 212
- `ToggleImageBookmark()` — line 257
- `UseAsCover()` — line 300

In each case, rename `'failure'` to `'error'`. The callback signature `(jqxhr, textStatus, error)` is correct for the `error` key — no further changes needed.

### Step 2 — Switch `ShowJumpToModal()` to `bootstrap.Modal.getOrCreateInstance()` (`js/viewer.js`)

Current code (line 141):
```js
const modal = new bootstrap.Modal(document.getElementById('jumpToModal'));
```

Replace with:
```js
const modal = bootstrap.Modal.getOrCreateInstance(document.getElementById('jumpToModal'));
```

The event listener attached immediately after (the `shown.bs.modal` handler with `{ once: true }`) is correct and unchanged.

### Step 3 — Remove dead `const viewer = this;` from `JumpTo()` (`js/viewer.js`)

Current code at approx. line 117:
```js
const viewer = this;
```

Remove this line. The `success` and `error` callbacks inside `JumpTo()` do not reference `viewer`; they use `$()` directly. No other change to `JumpTo()` is needed.

### Step 4 — Harden the Go button selector in `JumpTo()` (`js/viewer.js`)

Current selector (appears three times in `JumpTo()`):
```js
$('#jumpToModal .btn-primary')
```

**PHP side** — Add `id="btn-jump-go"` to the Go button in `ReaderPage.php renderImages()`. The button currently reads:
```html
<button type="button" class="btn btn-primary"
        onclick="viewer.JumpTo()">
```
Change to:
```html
<button type="button" id="btn-jump-go" class="btn btn-primary"
        onclick="viewer.JumpTo()">
```

**JS side** — Replace all three occurrences of `$('#jumpToModal .btn-primary')` in `JumpTo()` with `$('#btn-jump-go')`.

### Step 5 — Harden `$currentPageNumber` fallback in `renderImages()` (`assets/classes/Page/ReaderPage.php`)

Current fallback (line 281–287):
```php
$currentPageNumber = array_search($episode, $fetchedEpisodes, true);
if ($currentPageNumber !== false) {
    $currentPageNumber += 1; // convert 0-based to 1-based
} else {
    $currentPageNumber = 0; // episode not in fetched list (fallback)
}
```

Change to default to `1` instead of `0`:
```php
$currentPageNumber = array_search($episode, $fetchedEpisodes, true);
if ($currentPageNumber !== false) {
    $currentPageNumber += 1; // convert 0-based to 1-based
} else {
    $currentPageNumber = 1; // episode not in fetched list (safe fallback: show as first page)
}
```

This produces "Page 1 of N" in the edge case rather than "Page 0 of N", and the modal input pre-fills with a valid in-range value.

---

## Dependencies

- No new Composer packages.
- No new JavaScript libraries.
- No class additions → no `composer dump-autoload` needed.

---

## Required Components

**Modified files only:**

- `js/viewer.js`
- `assets/classes/Page/ReaderPage.php`

---

## Assumptions

- Bootstrap 5.3 is in use (confirms `bootstrap.Modal.getOrCreateInstance()` is available — it was introduced in Bootstrap 5.0).
- The `.btn-primary` Go button in the modal footer has no other `id` attribute currently.
- All five debt items are addressed in a single work package; there is no benefit to splitting them across separate WPs.

---

## Constraints

- PHPStan must remain at 0 new errors after the PHP change.
- PHPUnit 60/60 pass must be maintained (no tested logic is altered).
- No new user-visible strings are introduced (no localization update needed).
- `'failure'` → `'error'` rename must be applied to **all five** AJAX calls in `viewer.js`, including `SetFlag()` (sharing the same pre-existing pattern), not just the four named explicitly in the synthesis.

---

## Out of Scope

- Refactoring other AJAX methods in `viewer.js` beyond the `'failure'` → `'error'` fix.
- Adding a PHPUnit test for the `array_search` fallback path (no PHP test infrastructure for reader page rendering exists yet; a recommendation to add one is noted but out of scope for this cleanup pass).
- Reviewing other JS files (`fetcher.js`, `builder.js`, `toolbar.js`) for similar `'failure'` patterns.

---

## Acceptance Criteria

- `'failure'` does not appear as an AJAX callback key anywhere in `js/viewer.js`.
- `ShowJumpToModal()` calls `bootstrap.Modal.getOrCreateInstance()`.
- `const viewer = this;` is absent from `JumpTo()`.
- All occurrences of `$('#jumpToModal .btn-primary')` in `JumpTo()` are replaced with `$('#btn-jump-go')`.
- The Go button in the modal HTML has `id="btn-jump-go"`.
- The `$currentPageNumber` fallback in `renderImages()` is `1` rather than `0`.
- PHPStan: 0 new errors.
- PHPUnit: 60/60 pass.

---

## Testing Strategy

**JavaScript:** Manual smoke-test in the browser:
1. Open any fully-fetched comic in the reader.
2. Click the Jump To toolbar button — modal opens, input is focused and selected.
3. Click Go with a valid page number — navigation occurs.
4. Click Go with an invalid page number (out of range) — error alert shown, Go button re-enables.
5. Close and reopen the modal — confirm it opens correctly on second use (exercises `getOrCreateInstance`).
6. Simulate a network failure (DevTools → offline) while on the reader — confirm an error alert is shown for `SavePosition`, `ToggleBookmark`, `ToggleImageBookmark`, and `UseAsCover` actions (exercises the `'error'` key fix).

**PHP:** Run PHPStan and PHPUnit:
```
vendor/bin/phpstan analyse
vendor/bin/phpunit
```

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **`id="btn-jump-go"` conflicts with another element** | The ID is unique within the modal and the reader page; grep confirms no other element uses this ID in the codebase. |
| **`getOrCreateInstance` unavailable in older Bootstrap build** | Confirmed available since Bootstrap 5.0; the project uses 5.3.x. No risk. |
| **Changing `$currentPageNumber` default from 0 to 1 affects the JS `const`** | The JS constant `currentPageNumber` is purely used to set the modal input default. `1` is within `min=1 max=N` range; no JS logic branches on it. No risk. |
