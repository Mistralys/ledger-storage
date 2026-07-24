# Synthesis — Fetch Caching Rework (2026-03-05-fetch-caching-rework-1)

**Date:** 2026-03-05  
**Status:** COMPLETE  
**Work Packages:** 5 / 5 COMPLETE  
**PHPUnit final:** 60 tests, 120 assertions, 1 skipped (Windows, expected), 0 failures  
**PHPStan final:** No errors (138 files analysed, baseline applied)

---

## 1. Project Goal

Close four carry-over debt items flagged in the `2026-03-05-fetch-caching` synthesis:

| # | Priority | Description |
|---|---|---|
| 1 | Medium | `Comic::getCacheFolder()` returned wrong path (used `getFolder()` instead of `getStorageFolder()`) |
| 2 | Medium | `FetchCache::put()` silently ignored `file_put_contents()` write failures |
| 3 | Low | `FetchCache::get()` accepted an unused `$url` parameter |
| 4 | Low | `FetchCache::isValid()` did not verify the HTML file exists on disk |

All four issues shared the same two source files (`FetchCache.php`, `Comic.php`) and test suites, so they were resolved in a single coordinated effort to keep PHPStan and PHPUnit green throughout.

---

## 2. What Was Built / Changed

### WP-001 — Fix `getCacheFolder()` path and wire `FetchCache` correctly

**Core architectural change.** Applied the "single source of truth" option from the previous synthesis plan:

- `Comic::getCacheFolder()` now returns `getStorageFolder() . '/cache'`, yielding the correct path `storage/comics/{alias}/downloaded/cache`.
- `FetchCache::__construct()` parameter renamed from `$comicStorageFolder` to `$cacheFolderPath`; the internal `/cache` append was **removed** — the caller provides the complete cache root.
- `Fetcher::__construct()` now passes `$this->comic->getCacheFolder()` to `FetchCache`, making `getCacheFolder()` the single authoritative source used by all I/O code.
- Test factories updated to reflect the new constructor semantics.

**Files modified:** `assets/classes/Comics/FetchCache.php`, `assets/classes/Comic.php`, `assets/classes/Fetcher.php`, `tests/TestSuites/FetchCacheTests.php`, `tests/TestSuites/FetcherCacheTests.php`

### WP-002 — Guard `put()` against write failures

- Added `public const int ERROR_WRITE_FAILED = 260301` to `FetchCache`.
- `put()` now checks the `file_put_contents()` return value and throws `WebcomicsBuilderException(ERROR_WRITE_FAILED)` on failure **before** calling `saveManifest()`, preserving manifest integrity.
- Added `testPutThrowsOnWriteFailure` with `#[RequiresOperatingSystem('Linux')]` (chmod-based test; correctly skipped on Windows).
- QA fixed a pre-existing type-safety issue in the `forceManifestAge()` test helper (unguarded `file_get_contents()` / `json_decode()` return values; 3 PHPStan level-9 errors corrected).

**Files modified:** `assets/classes/Comics/FetchCache.php`, `tests/TestSuites/FetchCacheTests.php`

### WP-003 — Remove unused `$url` from `get()` and fix `isValid()` file-existence check

- `FetchCache::get(string $type, ?int $ttl = null)` — `$url` parameter removed from signature and docblock.
- Updated the single call site in `Fetcher::fetchHTML()` and all ~8 `get()` occurrences in `FetchCacheTests`.
- `isValid()` now returns `false` when the HTML file does not exist on disk, even if a valid manifest entry is present within TTL.
- Added `testIsValidReturnsFalseWhenHtmlFileDeleted` test.

**Files modified:** `assets/classes/Comics/FetchCache.php`, `assets/classes/Fetcher.php`, `tests/TestSuites/FetchCacheTests.php`

### WP-004 — Integration verification gate

Verified the combined result of WP-001—003:

- PHPStan: `[OK] No errors` (138 files, baseline applied).
- PHPUnit: 60 tests, 120 assertions, 1 skipped, 0 failures — meeting the ≥ 60 test threshold defined by the plan.
- QA removed a stale debt callout from `constraints.md` that had not been cleared during the Developer pass.

### WP-005 — Final manifest documentation audit

Confirmed all three manifest documents were in full conformance with implemented changes. No further edits were required; manifest maintenance had been kept current throughout the prior WPs.

**Manifest files verified:** `docs/agents/project-manifest/api-surface.md`, `docs/agents/project-manifest/constraints.md`, `docs/agents/project-manifest/storage.md`

---

## 3. Files Changed (complete list)

| File | Change |
|---|---|
| `assets/classes/Comic.php` | `getCacheFolder()` fixed to use `getStorageFolder()` |
| `assets/classes/Comics/FetchCache.php` | Constructor semantics (no suffix append), `ERROR_WRITE_FAILED` constant, write-failure guard in `put()`, `$url` removed from `get()`, `file_exists()` guard in `isValid()` |
| `assets/classes/Fetcher.php` | `FetchCache` constructed with `getCacheFolder()`; `get()` call site updated |
| `tests/TestSuites/FetchCacheTests.php` | Factory updated, new tests (`testPutThrowsOnWriteFailure`, `testIsValidReturnsFalseWhenHtmlFileDeleted`), `forceManifestAge()` type-safety fix, all `get()` calls updated |
| `tests/TestSuites/FetcherCacheTests.php` | `getCacheFolder()` stub added to `createFetcher()` |
| `docs/agents/project-manifest/api-surface.md` | `FetchCache` entry updated (constructor param, constant, `get()` signature, `isValid()` note, `get()` stale `$url` removed) |
| `docs/agents/project-manifest/constraints.md` | Stale debt callouts for Issues 1 & 2 removed |
| `docs/agents/project-manifest/storage.md` | `getCacheFolder()` discrepancy warning removed; construction note updated |
| `docs/agents/project-manifest/data-flows.md` | Fetch bootstrap line updated to reference `Comic::getCacheFolder()` |

---

## 4. Metrics

| Metric | Before | After |
|---|---|---|
| PHPUnit tests | 58 | 60 |
| PHPUnit assertions | ~113 | 120 |
| Tests skipped | 0 | 1 (Linux-only, expected) |
| Tests failing | 0 | 0 |
| PHPStan errors | 0 | 0 |
| Known-debt callouts (manifest) | 4 | 0 |

---

## 5. Open Observations (deferred)

These low-priority items were noted during review but are out of scope for this project:

| Source | Item |
|---|---|
| WP-001 code review | `FetchCacheTests` hardcodes the manifest path (`$this->tempDir . '/cache/cache-manifest.json'`). A `FetchCache::getManifestPath()` getter would allow tests to reference it authoritatively. |
| WP-001 code review | `FetchCache::put()` writes HTML with `file_put_contents()`, which is not atomic. A partial write followed by a crash could leave a truncated HTML file with a valid manifest entry. Pre-existing, not introduced by this work. |
| WP-002/WP-004 code review | `WebcomicsBuilderException` in `put()` passes `''` as `$details`; passing `null` would be more idiomatic. |
| WP-002 code review | `FetchCache::get()` returns `null` silently on `file_get_contents()` failure (correct cache-miss behaviour, but indistinguishable from "file missing" vs "file unreadable"). |
| WP-003 code review | `Fetcher::cleanHTML()` is missing a type hint for its `$html` parameter. Pre-existing. |
| WP-004 QA | `testPutThrowsOnWriteFailure` is Linux-only. If a Linux CI runner is ever added, the test will run automatically without changes. |

---

## 6. Conclusion

All four debt items from the prior synthesis are fully resolved. The caching subsystem (`FetchCache`, `Comic::getCacheFolder()`, `Fetcher`) is now correct, exception-safe, and has a clean public API. Project manifest documentation is fully synchronized with the implementation. The PHPStan and PHPUnit pipelines are green with no regressions.
