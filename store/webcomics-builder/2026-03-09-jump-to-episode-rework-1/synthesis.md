# Synthesis — Jump To Episode Rework 1 (2026-03-09-jump-to-episode-rework-1)

**Date:** 2026-03-09  
**Status:** COMPLETE  
**Work Packages:** 1 / 1 COMPLETE  
**PHPUnit final:** 60 tests, 0 failures, 1 pre-existing skip, 1 pre-existing deprecation (both unrelated)  
**PHPStan final:** No errors (138 files analysed, baseline applied)

---

## 1. Project Goal

Close five low-priority debt items flagged in the `2026-03-09-jump-to-episode` synthesis. All items were in two files (`js/viewer.js`, `assets/classes/Page/ReaderPage.php`) and were addressed in a single cleanup pass.

| # | Priority | Description |
|---|---|---|
| 1 | Low | `'failure'` is not a recognised jQuery AJAX callback key; correct key is `'error'`. Affected all five AJAX methods. |
| 2 | Low | `ShowJumpToModal()` used `new bootstrap.Modal()` on every call; `getOrCreateInstance()` is the Bootstrap-recommended safe pattern for repeated opens. |
| 3 | Low | `JumpTo()` declared `const viewer = this;` which was never referenced in any callback — dead code. |
| 4 | Low | Go button selector `$('#jumpToModal .btn-primary')` was fragile; a dedicated `id="btn-jump-go"` eliminates the coupling. |
| 5 | Low | `$currentPageNumber` fallback was `0`, producing "Page 0 of N" in the episode header on an unreachable edge case; fallback changed to `1`. |

---

## 2. What Was Changed

### WP-001 — Jump-to-Episode Debt Cleanup

All five debt items resolved in a single pass across two files.

**`js/viewer.js`**

- `'failure'` → `'error'` in the AJAX error callback key of all five methods: `SetFlag()` (line 76), `SavePosition()` (line 172), `ToggleBookmark()` (line 208), `ToggleImageBookmark()` (line 248), `UseAsCover()` (line 283). The callback signature `(jqxhr, textStatus, error)` was already correct for the `error` key — only the property name was changed.
- `ShowJumpToModal()`: `new bootstrap.Modal(document.getElementById('jumpToModal'))` replaced with `bootstrap.Modal.getOrCreateInstance(document.getElementById('jumpToModal'))`. The `shown.bs.modal` listener with `{ once: true }` was unchanged.
- `JumpTo()`: removed dead `const viewer = this;` declaration (line ~117); the success and error callbacks use `$()` directly and never referenced this alias.
- `JumpTo()`: all three occurrences of `$('#jumpToModal .btn-primary')` replaced with `$('#btn-jump-go')` (disable on entry, re-enable on non-success response, re-enable on network error).

**`assets/classes/Page/ReaderPage.php`**

- `renderImages()`: Go button in the `#jumpToModal` footer given `id="btn-jump-go"` to support the stable JS selector.
- `renderImages()`: `$currentPageNumber` fallback branch changed from `$currentPageNumber = 0` to `$currentPageNumber = 1` — produces "Page 1 of N" rather than "Page 0 of N" in the unlikely edge case where the episode is absent from the fetched list.

---

## 3. Files Changed (complete list)

| File | Change |
|---|---|
| `js/viewer.js` | `'failure'` → `'error'` in 5 AJAX calls; `getOrCreateInstance()` in `ShowJumpToModal()`; dead `const viewer` removed from `JumpTo()`; Go button selector hardened to `#btn-jump-go` |
| `assets/classes/Page/ReaderPage.php` | `id="btn-jump-go"` added to Go button; `$currentPageNumber` fallback changed from `0` to `1` |

No manifest documents required updating — all changes were internal implementation fixes with no public API surface changes, no new classes, no new files, and no new storage fields.

---

## 4. Metrics

| Metric | Before | After |
|---|---|---|
| PHPUnit tests | 60 | 60 |
| Tests failing | 0 | 0 |
| PHPStan errors | 0 | 0 |
| Known-debt callouts (from prior synthesis) | 5 | 0 |

---

## 5. Observations

### Code-review notes (all low priority, no action required)

- `JumpTo()` uses unquoted shorthand method syntax for `success` and `error` keys, while the rest of the class quotes all AJAX keys (`'success'`, `'error'`). Both are valid JS; minor style inconsistency only.
- The `$('#btn-jump-go')` selector appears three times in `JumpTo()`. Caching it in a local variable would be marginally cleaner, but the function scope is small enough that this is negligible.
- `ShowJumpToModal()` calls `document.getElementById()` directly while the rest of the class uses jQuery selectors. This is functionally required — `bootstrap.Modal.getOrCreateInstance()` takes an `HTMLElement`, not a jQuery object. Minor stylistic inconsistency; correct as-is.
- The Go button (`id=btn-jump-go`) lives outside the `<form>` in the modal footer and is triggered via `onclick`; the form handles Enter via `onsubmit`. This split is correct and intentional but worth noting for future maintainers.
