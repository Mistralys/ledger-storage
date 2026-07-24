# Synthesis — Preserve Index Metadata: Post-Synthesis Rework

**Project path:** `docs/agents/plans/2026-03-04-preserve-index-metadata-rework-1`  
**Date completed:** 2026-03-05  
**Status:** COMPLETE — all 6 work packages passed all pipelines

---

## 1. Project Purpose

This plan addressed three follow-up items identified in the synthesis of the `2026-03-04-preserve-index-metadata` plan, plus one additional task (test helper cleanup) discovered during implementation:

1. **PHPStan cleanup** — Add type annotations to the new metadata-preservation methods in `BaseComicSource` to resolve 6 `--level=max` errors.
2. **`filterText()` deduplication** — Extract the identical `filterText()` implementations from `BaseComicSource` and `PopulateIndex` into a shared static utility class `MetadataFilter`.
3. **Stale episode ID test** — Add a test confirming snapshot entries for episodes no longer in the re-fetched index are silently dropped.
4. **Test helper cleanup** — Remove the now-redundant `callFilterText()` reflection wrapper from `TestableBaseComicSource` (tests can call `MetadataFilter::filterText()` directly).

---

## 2. Work Package Summary

| WP | Title | Outcome |
|---|---|---|
| WP-001 | Add PHPStan type annotations to `BaseComicSource` metadata methods | COMPLETE — PASS |
| WP-002 | Create `MetadataFilter` static utility class | COMPLETE — PASS |
| WP-003 | Update `BaseComicSource` to call `MetadataFilter::filterText()` | COMPLETE — PASS |
| WP-004 | Update `PopulateIndex` to call `MetadataFilter::filterText()` | COMPLETE — PASS |
| WP-005 | Add stale episode ID test + remove `callFilterText()` from test helper | COMPLETE — PASS |
| WP-006 | Update project manifest (`api-surface.md`, `file-tree.md`) | COMPLETE — PASS |

All WPs passed implementation, QA, code-review, and documentation pipelines.

---

## 3. Changes Delivered

### 3.1 New File

**`assets/classes/Comics/Sources/MetadataFilter.php`**

- Namespace: `WebcomicsBuilder\Comics\Sources`
- Single public static method: `public static function filterText(string $text): string`
- Trims whitespace, strips CR/LF characters, and returns an empty string for numeric-only values.
- Eliminates the code duplication that existed between `BaseComicSource` and `PopulateIndex`.

### 3.2 Modified Files

**`assets/classes/Comics/Sources/BaseComicSource.php`**
- Added `@var` type annotations in `snapshotIndexMetadata()` and `autoPopulateFromEpisodes()` to narrow `mixed` returns from `JSONFile::getData()` / `JSONFile::parse()`. Resolves all 6 `--level=max` PHPStan errors introduced by the previous plan.
- Removed the private `filterText()` method.
- Updated the call in `autoPopulateFromEpisodes()` to `MetadataFilter::filterText(...)` (same namespace — no import required).

**`assets/classes/Actions/Action/PopulateIndex.php`**
- Added `use WebcomicsBuilder\Comics\Sources\MetadataFilter;` import.
- Removed the private `filterText()` method.
- Updated the three call sites in `_run()` to `MetadataFilter::filterText(...)`.

**`tests/TestSuites/Comics/BaseComicSourceMetadataTests.php`**
- Added `testMergePreservedMetadataDropsStaleEpisodeIDs()` — verifies that the `!isset($preservedMetadata[$episodeID])` guard correctly drops snapshot entries for episode IDs no longer present in the re-fetched index.

**`tests/TestClasses/TestableBaseComicSource.php`**
- Removed the `callFilterText()` reflection wrapper (no longer needed; tests call `MetadataFilter::filterText()` directly).

### 3.3 Manifest Updates

**`docs/agents/project-manifest/api-surface.md`**
- Added `MetadataFilter` class entry with `public static filterText(string $text): string` signature.
- `filterText()` is NOT listed as a private method of `BaseComicSource` (it was removed from that class).

**`docs/agents/project-manifest/file-tree.md`**
- Added `MetadataFilter.php` under `assets/classes/Comics/Sources/` with correct annotation.

---

## 4. Verification Results

### PHPUnit
- **29 tests, 61 assertions — all PASS** (consistent across implementation, QA, and code-review verification runs).
- 1 pre-existing deprecation notice (not an error; pre-dates this plan).

### PHPStan
- **Level 5 (configured):** 0 errors on `MetadataFilter.php` and `PopulateIndex.php`.
- **Level max on `BaseComicSource.php`:** Reduced from 18 errors to 12 (the 6 new-method errors eliminated; 12 pre-existing errors remain, unchanged from the baseline before this plan).

---

## 5. Architectural Notes

- **Static utility over trait**: `filterText()` is pure, stateless text normalization. A static method in a dedicated class is idiomatic; a trait would add coupling with no benefit.
- **`Sources/` namespace scope**: The function is specific to comic index metadata (it blanks numeric-only values, not general whitespace normalisation). Placing it in the `Sources` namespace correctly communicates this scope.
- **`@var` annotations over library changes**: `JSONFile::getData()` returns `mixed`/`array<int|string, mixed>` (third-party library). Narrowing locally with `@var` is the standard, non-invasive PHPStan pattern and avoids cascading changes.
- **`callFilterText()` removal**: With `MetadataFilter` a public static class, tests no longer need reflection to access the method. The wrapper was removed to keep `TestableBaseComicSource` minimal.

---

## 6. No Outstanding Issues

All rework items from the prior synthesis have been resolved. No new issues were identified. The codebase is clean at PHPStan level 5 and all tests pass.
