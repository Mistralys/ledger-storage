# Synthesis — Jump To Episode Feature

**Project:** Jump To Episode  
**Date:** 2026-03-09  
**Status:** COMPLETE  
**Work Packages:** WP-001 (PHP + Modal), WP-002 (JS Rework)

---

## What Was Built

A "Jump To" feature for the comic reader toolbar. Users can now enter a 1-based page number into a Bootstrap 5 modal to navigate directly to any fetched episode, bypassing the need to know episode ID slugs.

---

## Files Modified

| File | Role |
|---|---|
| `assets/classes/Page/ReaderPage.php` | Added `ajax_resolvePageNumber()` handler; emitted JS constants; added modal HTML; added toolbar button; updated episode header; removed dead commented code |
| `assets/classes/UI/Icon.php` | Added `jumpTo()` icon method (`fa-arrow-right-to-bracket`) |
| `js/viewer.js` | Added `ShowJumpToModal()`; rewrote `JumpTo()` |
| `docs/agents/project-manifest/api-surface.md` | Documented `jumpTo()` icon and `ajax_resolvePageNumber()` handler |
| `docs/agents/project-manifest/data-flows.md` | Documented JS constants, modal, toolbar button, and full Jump To AJAX flow |

---

## Implementation Summary

### PHP side (WP-001)

- **`ajax_resolvePageNumber()`** added to `ReaderPage` after `ajax_useAsCover()`. Takes a 1-based `pageNumber` request parameter, validates it against `$fetcher->getFetched()`, and returns `sendAJAXSuccess(['viewURL' => ...])` or `sendAJAXError(...)`. Dispatched automatically by `BasePage::handleAJAX()` via the `ajax_` prefix convention — no registration step needed.
- **`renderImages()`** now computes `$fetchedEpisodes`, `$currentPageNumber` (1-based position of the current episode in the fetched list), and `$totalPages`, and emits `totalPages` and `currentPageNumber` as JS constants in the inline `<script>` block.
- **Episode header** updated to show `Episode #X — Page N of M`.
- **Bootstrap 5 `#jumpToModal`** rendered in the page body (not in the toolbar). Contains a number input (`#field-jumper`) pre-filled with `$currentPageNumber`, labelled with the valid range (`1–N`), and a Go button. Form `onsubmit` calls `event.preventDefault(); viewer.JumpTo()`.
- **Toolbar button** added calling `viewer.ShowJumpToModal()`, using the new `jumpTo()` icon.
- Old commented-out `addToolbarHTML()` block removed.
- All user-visible strings use `pt()` / `t()`.

### JavaScript side (WP-001 QA + WP-002)

Two bugs were found by QA in the WP-001 implementation and fixed in WP-002:

1. `ShowJumpToModal()` was absent — the toolbar button's onclick would have thrown a `TypeError` at runtime.
2. `JumpTo()` referenced an undeclared `baseURL` variable and used `&episode=` (not a recognised server parameter).

The final WP-002 implementation:

- **`ShowJumpToModal()`** calls `new bootstrap.Modal(el).show()` and attaches a `shown.bs.modal` listener with `{ once: true }` to focus and select `#field-jumper` after the modal animation completes.
- **`JumpTo()`** reads `#field-jumper` with `parseInt()`, disables the Go button, calls `this.ajaxBaseURL + '&ajax=resolvePageNumber&pageNumber=' + pageNumber`, navigates to `data.data.viewURL` on success, and shows an error alert + re-enables Go on both server-side and network errors.

---

## Test Results

- **PHPStan:** 0 errors (138 files) after all changes.
- **PHPUnit:** 60/60 pass (1 pre-existing unrelated deprecation, 1 pre-existing unrelated skip).

---

## Observations & Known Debt

### Carried Forward (no action required now)

| Priority | Location | Note |
|---|---|---|
| Low | `js/viewer.js ShowJumpToModal()` | Uses `new bootstrap.Modal(el)` on every call. `bootstrap.Modal.getOrCreateInstance(el)` is the canonical safe pattern for repeated opens. Functionally correct as-is. |
| Low | `js/viewer.js JumpTo()` | `const viewer = this;` is declared but never used inside callbacks. Dead variable; harmless. |
| Low | `assets/classes/Page/ReaderPage.php renderImages()` | If `array_search()` returns `false` (episode not in fetched list), `$currentPageNumber` is 0. Produces "Page 0 of N" in the header and pre-fills the input below `min=1`. HTML5 validation blocks submission; no crash. The reader guard makes this extremely rare, but the fallback could be hardened. |
| Low | `js/viewer.js` (pre-existing) | `SavePosition()`, `ToggleBookmark()`, `ToggleImageBookmark()`, `UseAsCover()` all use `'failure'` as the jQuery AJAX error callback key; the correct key is `'error'`. Not introduced by this feature — flagged for a future cleanup pass. |
| Low | `js/viewer.js JumpTo()` | Go button selector `$('#jumpToModal .btn-primary')` is slightly fragile if a second `.btn-primary` were added to the modal footer. Acceptable for the current single-action modal structure. |

---

## Architecture Notes

- The page-number → episode-URL mapping is resolved on the server via AJAX rather than embedding a full lookup table in page HTML. This avoids page bloat for comics with hundreds or thousands of episodes.
- The feature operates on the **fetched-only** episode list (not the full index), consistent with how the reader's navigation buttons (First / Previous / Next / Last) work.
- AJAX dispatch requires no registration — `BasePage::handleAJAX()` discovers `ajax_resolvePageNumber()` automatically via `method_exists` and the `ajax_` prefix.
