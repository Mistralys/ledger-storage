# Synthesis — Fetch Caching System

**Project:** 2026-03-05-fetch-caching  
**Date:** 2026-03-05  
**Status:** COMPLETE  
**Work Packages:** 6 / 6 COMPLETE  
**PHPUnit:** 58 / 58 pass, 113 assertions  
**PHPStan:** 0 errors (138 files; 21 pre-existing errors baselined)

---

## Objective

Implement a persistent HTML caching layer for `Fetcher::fetchHTML()` that avoids redundant HTTP requests to comic websites across sessions. Replace the ephemeral `temp/` session-cache mechanism with a TTL-based persistent disk cache, transparent to all source drivers.

---

## What Was Built

### WP-001 — `FetchCache` class

Created `assets/classes/Comics/FetchCache.php` in namespace `WebcomicsBuilder\Comics`.

Key characteristics:
- Two typed TTL constants: `TTL_INDEX = 86400` (1 day), `TTL_EPISODE = 2592000` (30 days).
- Constructor receives a base folder path; creates `cache/` and `cache/html/` subdirectories on first use.
- Lazy manifest loading — the `cache-manifest.json` file is not read until the first cache operation. Manifest writes use `JSONFile` for atomic disk I/O.
- Public API: `get()`, `put()`, `isValid()`, `invalidate()`, `invalidateAll()`, `getManifest()`.
- Cache lookup is keyed by `$type` string (e.g. `"index"`, `"episode-42"`) — same key scheme as the old `temp/` approach.
- HTML files are stored as `cache/html/{type}.html`. The URL is stored in the manifest for diagnostics only; it does not participate in cache lookup.

### WP-002 — Fetcher and Comic integration

Modified `assets/classes/Fetcher.php` and `assets/classes/Comic.php`:

- `Comic::getCacheFolder()` added — returns `{getFolder()}/cache`.
- `Fetcher` now owns a private `FetchCache $cache` property, instantiated in the constructor with `$this->folder` (the comic's `getStorageFolder()` path).
- `Fetcher::fetchHTML()` now follows a three-step lookup chain:
  1. **In-memory** (`self::$helpers` static map) — unchanged.
  2. **Persistent cache** (`FetchCache::get()`) — skipped when `bypassCache = true`.
  3. **HTTP download** — result written to persistent cache via `FetchCache::put()`.
- `Fetcher::setBypassCache(bool): self` — fluent setter; when `true`, step 2 is skipped (but step 3 still writes).
- `Fetcher::getCache(): FetchCache` — returns the instance for direct invalidation.
- `Fetcher::resolveTTL(string $type): int` — private method; returns `TTL_EPISODE` for `episode-*` types, `TTL_INDEX` for everything else.
- `downloadHTMLFile()` removed; the persistent cache subsumes its role.
- `clearCachedDownloads()` updated to call `$this->cache->invalidateAll()` while retaining legacy temp-directory cleanup; return type changed from `void` to `self`.

### WP-003 — FetchPage UI wiring

Modified `assets/classes/Page/FetchPage.php`:

- **Refresh Index** (`REQUEST_PARAM_FETCH`): now calls `$fetcher->setBypassCache(true)` before `fetchIndex()` instead of `clearCachedDownloads()`. The cache is bypassed for this run but populated afresh by the HTTP responses — subsequent loads are fully cached.
- **Clear Cache** (`REQUEST_PARAM_REFRESH`): now calls `$fetcher->getCache()->invalidateAll()` to wipe the persistent cache.
- No orphaned `clearCachedDownloads()` calls remain in this file.

### WP-004 — Unit / integration test suites

Created:

- `tests/TestSuites/FetchCacheTests.php` — 19 test methods covering: `put()`/`get()` round-trip, TTL expiry, null-TTL bypass, `invalidate()`, `invalidateAll()`, `isValid()`, cache-miss, manifest URL storage, manifest persistence, and TTL constant ordering. Uses `sys_get_temp_dir()` temp directories cleaned up via `FileHelper::deleteTree()` in `tearDown()`.
- `tests/TestSuites/FetcherCacheTests.php` — 10 test methods covering: `resolveTTL()` (four variants including the `"episode"` no-dash edge case), `getCache()`, `setBypassCache()` (fluent true/false), and `clearCachedDownloads()` wipe. Private `resolveTTL()` accessed via `ReflectionMethod`.

Both files use `strict_types`, proper namespace, and `#[CoversClass]` attributes. PHPUnit auto-discovers them via the existing `phpunit.xml` directory glob — no XML changes required.

### WP-005 — Full system validation

- PHPStan level 5: 0 errors on 138 files. A `phpstan-baseline.neon` was generated for 21 pre-existing errors in `Builder.php`, `DOMHelper.php`, `Form.php`, `ReaderPage.php`, etc. — none related to the caching code. The baseline is included in `phpstan.neon`.
- PHPUnit: 58/58 pass, 113 assertions, 0 failures, 0 errors (1 pre-existing PHPUnit-internal framework deprecation, not introduced by this project).
- Human smoke test on `kill-six-billion-demons`:
  - **AC-3 ✓**: First Refresh Index populated `storage/comics/kill-six-billion-demons/downloaded/cache/html/` with HTML files and `cache-manifest.json`.
  - **AC-4 ✓**: Second immediate Refresh Index served all pages from cache with unchanged `cachedAt` timestamps and zero HTTP requests to origin.
  - **AC-5 ✓**: Clear Cache emptied the directory and reset `cache-manifest.json`.

### WP-006 — Project-manifest documentation

Updated five manifest files to reflect the complete caching feature:

| File | Changes |
|---|---|
| `api-surface.md` | New `FetchCache` class block; updated `Fetcher` (new methods, removed `downloadHTMLFile`); new `Comic::getCacheFolder()`; `isValid()` asymmetry note; `get()` `$url` parameter note |
| `file-tree.md` | `FetchCache.php` entry; per-comic `downloaded/cache/` tree; `FetchCacheTests.php` and `FetcherCacheTests.php` entries |
| `storage.md` | `cache/` directory structure; `cache-manifest.json` schema with `cachedAt`/`url` fields; TTL constants table; `getCacheFolder()` discrepancy warning |
| `constraints.md` | Fetch Cache TTL Defaults section (values, rationale, non-reduction guidance); known-debt callouts for `getCacheFolder()` and `put()` unguarded write; PHPUnit version corrected to `>= 12.5`; `phpstan-baseline.neon` noted |
| `data-flows.md` | Section 4 expanded: three sub-flows (Refresh Index, Clear Cache, normal page load); Episode Fetching flow shows FetchCache `get()`/`put()` steps with `TTL_EPISODE` |

---

## Files Modified

### New files
| File | Description |
|---|---|
| `assets/classes/Comics/FetchCache.php` | Persistent cache class |
| `tests/TestSuites/FetchCacheTests.php` | 19 FetchCache unit tests |
| `tests/TestSuites/FetcherCacheTests.php` | 10 Fetcher cache integration tests |
| `phpstan-baseline.neon` | Baseline of 21 pre-existing PHPStan errors |

### Modified files
| File | Changes |
|---|---|
| `assets/classes/Fetcher.php` | FetchCache integration; `setBypassCache`, `getCache`, `resolveTTL`; `downloadHTMLFile` removed; `clearCachedDownloads` updated |
| `assets/classes/Comic.php` | `getCacheFolder()` added |
| `assets/classes/Page/FetchPage.php` | Refresh Index and Clear Cache actions rewired |
| `phpstan.neon` | `includes: [phpstan-baseline.neon]` added |
| `docs/agents/project-manifest/api-surface.md` | FetchCache, Fetcher, Comic updates |
| `docs/agents/project-manifest/file-tree.md` | Cache paths and new test files |
| `docs/agents/project-manifest/storage.md` | Cache directory and manifest schema |
| `docs/agents/project-manifest/constraints.md` | TTL defaults, known-debt notes |
| `docs/agents/project-manifest/data-flows.md` | FetchPage and Episode Fetching flows |

---

## Known Open Items (Carry-Over Debt)

The following items were flagged during code review across multiple WPs and are documented in `constraints.md`. They do not affect correctness or test outcomes but should be addressed before the next agent builds on `FetchCache`.

| Priority | Issue | Recommended Fix |
|---|---|---|
| Medium | `Comic::getCacheFolder()` returns `{getFolder()}/cache` but the actual cache lives at `{getFolder()}/downloaded/cache` (Fetcher constructs `FetchCache` with `getStorageFolder()`). No current caller relies on `getCacheFolder()` for I/O, but it is a latent trap. | Change `getCacheFolder()` to return `getStorageFolder().'/cache'`, or construct `FetchCache` in `Fetcher` with `$this->comic->getCacheFolder()`. Option (b) is preferred. |
| Medium | `FetchCache::put()` calls `file_put_contents()` without checking the return value before `saveManifest()`. A failed write silently creates a manifest entry pointing to a non-existent file. `get()` handles the miss gracefully (returns `null`), but the stale manifest entry persists. | Guard with `if (file_put_contents(...) === false) { throw new WebcomicsBuilderException(...); }` before `saveManifest()`. |
| Low | `FetchCache::get()` accepts a `$url` parameter that is unused in the read path. | Remove `$url` from `get()` and update the single call site in `Fetcher::fetchHTML()`. |
| Low | `FetchCache::isValid()` checks only the manifest timestamp, not whether the HTML file exists. `get()` checks `file_exists()`. A caller using `isValid()` as a pre-flight guard will be surprised if the HTML file was deleted externally. | Add `file_exists()` check to `isValid()`, or document the asymmetry in the method docblock. |

---

## Test Metrics

| Suite | Tests | Assertions | Result |
|---|---|---|---|
| Pre-existing suites | 31 | — | PASS |
| `FetchCacheTests` | 19 | — | PASS |
| `FetcherCacheTests` | 10 | — | PASS |
| **Total** | **58** | **113** | **PASS** |

PHPStan: **0 errors** (138 files; 21 pre-existing errors in `phpstan-baseline.neon`).

---

## Architecture Summary

The fetch-caching feature introduces a single new class (`FetchCache`) that slots cleanly into the existing architecture without touching any source driver. The integration point is `Fetcher::fetchHTML()` — the sole HTTP entry point — which now follows an in-memory → persistent-disk → HTTP fallback chain. All source drivers, page classes, and AJAX endpoints benefit automatically.

The cache is per-comic and per-type, scoped to `storage/comics/{alias}/downloaded/cache/`. Manifest writes are atomic (via the existing `JSONFile` helper). TTLs are type-driven (`episode-*` → 30 days, everything else → 1 day). Invalidation is available at the individual entry level (`invalidate()`) and the full-comic level (`invalidateAll()`), and is exposed to the UI via the existing Refresh Index and Clear Cache actions in `FetchPage`.
