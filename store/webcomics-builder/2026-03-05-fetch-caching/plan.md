# Plan — Fetch Caching System

## Summary

Implement a persistent HTML caching layer for the `Fetcher` class that avoids redundant HTTP requests to comic websites. A new `FetchCache` class will manage per-comic disk caches with layered TTL expiry (short for index pages, long for episode pages). The caching integrates transparently at the `Fetcher::fetchHTML()` chokepoint, requiring no changes to source drivers. The existing `temp/` session-cache mechanism is retired in favour of the new persistent cache.

## Architectural Context

The application fetches HTML from comic websites in two workflows:

1. **Index fetching** — synchronous PHP: `FetchPage` → `Fetcher::fetchIndex()` → `BaseComicSource::_fetchIndex()` → one or more `Fetcher::fetchHTML()` calls (e.g., KillSixBillionDemonsSource makes ~30 calls across book archives and paginated sub-pages).
2. **Episode fetching** — AJAX-driven with JS throttling: `fetcher.js` → `FetchPage::ajax_fetchEpisode()` → `Fetcher::fetchEpisode()` → `BaseComicSource::processEpisode()` → `Fetcher::fetchHTML()` for the episode page.

Both workflows funnel through `Fetcher::fetchHTML()` ([Fetcher.php](../../../../assets/classes/Fetcher.php#L75-L87)), which is the sole HTTP entry point for HTML content. The method currently has two caching layers:

- **In-memory**: static `self::$helpers` map keyed by `md5(type-url)` — deduplicates within a single PHP process.
- **Temp-file**: `downloadHTMLFile()` writes to `storage/comics/{alias}/temp/{type}.html` — persists within a session but is wiped by `clearCachedDownloads()` before every "Refresh Index" and "Clear Cache" action in [FetchPage.php](../../../../assets/classes/Page/FetchPage.php#L72-L98).

The temp-file approach means every "Refresh Index" click re-downloads all pages from scratch, even when the archive content hasn't changed. The proposed `FetchCache` replaces this with TTL-based persistent caching.

Key files involved:

| File | Role |
|---|---|
| [assets/classes/Fetcher.php](../../../../assets/classes/Fetcher.php) | HTTP fetch orchestration; integration point |
| [assets/classes/Page/FetchPage.php](../../../../assets/classes/Page/FetchPage.php) | UI actions: Refresh Index, Clear Cache |
| [assets/classes/Comics/Sources/BaseComicSource.php](../../../../assets/classes/Comics/Sources/BaseComicSource.php) | Base source driver (calls `fetchHTML` indirectly) |
| [assets/classes/Comics/Sources/ComicSourceInterface.php](../../../../assets/classes/Comics/Sources/ComicSourceInterface.php) | Driver contract |
| [assets/classes/Comic.php](../../../../assets/classes/Comic.php) | Comic entity (folder/temp folder methods) |

## Approach / Architecture

### New Class: `FetchCache`

A new class `FetchCache` in `assets/classes/Comics/` encapsulates all persistent caching logic. Each `Fetcher` instance owns a `FetchCache` instance scoped to its comic.

```
FetchCache
├── Storage: storage/comics/{alias}/cache/
│   ├── html/                          # Cached HTML files (one per type)
│   │   ├── index.html
│   │   ├── book-1.html
│   │   ├── book-1-archive-2.html
│   │   ├── episode-42.html
│   │   └── ...
│   └── cache-manifest.json            # Manifest tracking all entries
│
├── Constants:
│   ├── TTL_INDEX   = 86400    (1 day)
│   ├── TTL_EPISODE = 2592000  (30 days)
│
├── Public Methods:
│   ├── get(string $type, string $url, ?int $ttl = null): ?string
│   ├── put(string $type, string $url, string $html): void
│   ├── invalidate(string $type): void
│   ├── invalidateAll(): void
│   ├── isValid(string $type, ?int $ttl = null): bool
│   └── getManifest(): array
```

### Cache Manifest Format (`cache-manifest.json`)

```json
{
    "index": {
        "url": "https://killsixbilliondemons.com/submit/",
        "cachedAt": 1741187400
    },
    "book-1-archive-2": {
        "url": "https://killsixbilliondemons.com/chapter/ksbd/page/2/",
        "cachedAt": 1741187405
    },
    "episode-42": {
        "url": "https://killsixbilliondemons.com/comic/ksbd-42/",
        "cachedAt": 1741100000
    }
}
```

The `cachedAt` field is a Unix timestamp (integer). The manifest is loaded lazily and written on every `put()` or `invalidate()` call. URL is stored for diagnostics/debugging but not used for cache matching — matching is by `$type` key only (same as the existing temp-file system).

### Integration into `Fetcher::fetchHTML()`

The modified flow:

```php
public function fetchHTML(string $type, string $url, ?Closure $htmlFilter=null) : DOMHelper
{
    // 1. In-memory cache (existing, unchanged)
    $key = md5($type.'-'.$url);
    if(isset(self::$helpers[$key])) {
        return self::$helpers[$key];
    }

    // 2. Persistent disk cache (NEW)
    if(!$this->bypassCache) {
        $ttl = $this->resolveTTL($type);
        $cached = $this->cache->get($type, $url, $ttl);
        if($cached !== null) {
            $helper = new DOMHelper($cached);
            self::$helpers[$key] = $helper;
            return $helper;
        }
    }

    // 3. HTTP download (existing logic, formerly in downloadHTMLFile)
    $source = $this->cleanHTML(FileHelper::downloadFile($url));
    if($htmlFilter) {
        $source = $htmlFilter($source);
    }

    // 4. Store in persistent cache (NEW)
    $this->cache->put($type, $url, $source);

    // 5. Build DOMHelper and store in memory cache (existing)
    $helper = new DOMHelper($source);
    self::$helpers[$key] = $helper;
    return $helper;
}
```

### TTL Resolution

A private method `resolveTTL(string $type): int` on `Fetcher` determines the TTL based on the `$type` prefix:
- Types starting with `episode-` → `FetchCache::TTL_EPISODE` (30 days)
- All other types (index, book-N, book-N-archive-N, chapters, etc.) → `FetchCache::TTL_INDEX` (1 day)

### Cache Bypass

A new `bool $bypassCache` property on `Fetcher` (default: `false`) with a setter `setBypassCache(bool): self`. When `true`, the persistent cache is skipped for reads (but writes still occur, so the cache is refreshed).

### FetchPage Changes

| Current Action | Current Behaviour | New Behaviour |
|---|---|---|
| "Refresh Index" (`REQUEST_PARAM_FETCH`) | `clearCachedDownloads()` then `fetchIndex()` | `setBypassCache(true)` then `fetchIndex()` — fresh downloads that also update the cache |
| "Clear Cache" (`REQUEST_PARAM_REFRESH`) | `clearCachedDownloads()` (deletes temp HTML) | `cache->invalidateAll()` — clears persistent cache |
| Normal page load | Downloads if temp files missing | Serves from cache if valid TTL |

### Retiring the Temp System

The existing `temp/` directory and `downloadHTMLFile()` private method in `Fetcher` are replaced:
- `downloadHTMLFile()` is removed; its logic is inlined into the modified `fetchHTML()`.
- `clearCachedDownloads()` is updated to delegate to `FetchCache::invalidateAll()` for backward compatibility, then also clear any remaining temp files (migration safety).
- `Comic::getTempFolder()` remains unchanged (it may be used for other purposes).

## Rationale

- **Single integration point.** All HTML fetching goes through `Fetcher::fetchHTML()`, so caching can be added in one place without touching any of the 30+ source drivers.
- **Layered TTL matches content volatility.** Index pages change when new episodes are published (days/weeks apart); episode pages are effectively immutable. Different TTLs reflect this.
- **Encapsulated in a dedicated class.** `FetchCache` handles all file I/O, manifest management, and TTL validation. `Fetcher` delegates to it, keeping its own complexity low.
- **No database.** The cache uses JSON + filesystem conventions, consistent with the project's JSON-only persistence constraint.
- **Bypass mode over invalidation for refresh.** Setting `bypassCache=true` during "Refresh Index" means fresh HTML is always downloaded and stored, updating the cache atomically. This is simpler and more correct than invalidate-then-fetch (which would require two steps).
- **No conditional HTTP requests in v1.** Many comic sites don't set proper `ETag`/`Last-Modified` headers. TTL-based caching provides reliable, server-independent behaviour. The manifest format has room for these fields in a future enhancement.

## Detailed Steps

### Step 1: Create the `FetchCache` class

Create [assets/classes/Comics/FetchCache.php](../../../../assets/classes/Comics/FetchCache.php) with:

- Namespace: `WebcomicsBuilder\Comics`
- Constructor: `__construct(string $comicStorageFolder)` — receives the comic's storage folder path, derives `{folder}/cache/` as the cache root and `{folder}/cache/html/` as the HTML directory.
- Creates the cache directories on construction if they don't exist.
- `get(string $type, string $url, ?int $ttl = null): ?string` — Loads the manifest, checks if an entry for `$type` exists, verifies TTL against `cachedAt`, returns the HTML file content or `null`.
- `put(string $type, string $url, string $html): void` — Writes the HTML to `cache/html/{$type}.html`, updates the manifest entry with `url` and `cachedAt` (current timestamp), saves the manifest.
- `isValid(string $type, ?int $ttl = null): bool` — Returns `true` if the entry exists and hasn't expired.
- `invalidate(string $type): void` — Removes the HTML file and the manifest entry.
- `invalidateAll(): void` — Deletes all HTML files in `cache/html/` and resets the manifest to `{}`.
- `getManifest(): array` — Returns the parsed manifest array (for diagnostics).
- Private `loadManifest(): void` / `saveManifest(): void` — Reads/writes `cache/cache-manifest.json`. Lazily loaded on first access.
- The manifest JSON file uses `JSONFile` from `application-utils` for consistency with the rest of the codebase (or `file_get_contents`/`file_put_contents` with `json_encode`/`json_decode` if simpler).

### Step 2: Add TTL constants to `FetchCache`

```php
public const int TTL_INDEX = 86400;       // 1 day
public const int TTL_EPISODE = 2592000;   // 30 days
```

### Step 3: Integrate `FetchCache` into `Fetcher`

Modify [assets/classes/Fetcher.php](../../../../assets/classes/Fetcher.php):

1. Add a `private FetchCache $cache` property.
2. In the constructor, instantiate `FetchCache` with the comic's storage folder: `$this->cache = new FetchCache($this->folder)`.
3. Add `private bool $bypassCache = false` property.
4. Add `public function setBypassCache(bool $bypass): self` method.
5. Add `public function getCache(): FetchCache` accessor.
6. Add `private function resolveTTL(string $type): int` that returns `FetchCache::TTL_EPISODE` for types starting with `episode-`, and `FetchCache::TTL_INDEX` otherwise.
7. Update `fetchHTML()` to use the cache as described in the integration section above.
8. Remove the `downloadHTMLFile()` private method (its logic is now split between `FetchCache` and the updated `fetchHTML()`).
9. Update `clearCachedDownloads()` to call `$this->cache->invalidateAll()` in addition to (or instead of) deleting temp files.

### Step 4: Update `FetchPage` to use cache bypass

Modify [assets/classes/Page/FetchPage.php](../../../../assets/classes/Page/FetchPage.php):

1. In the "Refresh Index" block (`REQUEST_PARAM_FETCH`): Replace `$this->fetcher->clearCachedDownloads()` with `$this->fetcher->setBypassCache(true)`. This ensures fresh pages are downloaded but also stored in the cache.
2. In the "Clear Cache" block (`REQUEST_PARAM_REFRESH`): Replace `$this->fetcher->clearCachedDownloads()` with `$this->fetcher->getCache()->invalidateAll()`. Optionally also call `clearCachedDownloads()` for backward compatibility during migration.

### Step 5: Add `getCacheFolder()` to `Comic`

Add a convenience method to [assets/classes/Comic.php](../../../../assets/classes/Comic.php):

```php
public function getCacheFolder() : string
{
    return sprintf('%s/cache', $this->getFolder());
}
```

This parallels the existing `getTempFolder()` method and can be used by `FetchCache` or future code.

### Step 6: Run `composer dump-autoload`

The new `FetchCache` class needs to be registered in the classmap autoloader.

### Step 7: Write unit tests for `FetchCache`

Create a test suite [tests/TestSuites/FetchCacheTests.php](../../../../tests/TestSuites/FetchCacheTests.php):

- **Test `put()` + `get()` round-trip**: Store HTML, retrieve it, verify content matches.
- **Test TTL expiry**: Store an entry, manipulate the manifest's `cachedAt` to be older than the TTL, verify `get()` returns `null`.
- **Test `invalidate()`**: Store an entry, invalidate it, verify `get()` returns `null` and the HTML file is deleted.
- **Test `invalidateAll()`**: Store multiple entries, invalidate all, verify all are gone.
- **Test `isValid()`**: Verify it correctly reports validity based on TTL.
- **Test cache miss**: `get()` on a non-existent type returns `null`.
- **Test URL stored in manifest**: Verify the manifest contains the correct URL for diagnostics.
- Use a temporary directory for test isolation (clean up in `tearDown()`).

### Step 8: Write integration test for `Fetcher` caching behaviour

Create [tests/TestSuites/FetcherCacheTests.php](../../../../tests/TestSuites/FetcherCacheTests.php):

- Test that `resolveTTL()` returns the correct TTL for index vs. episode types.
- Test that `setBypassCache(true)` causes `fetchHTML()` to skip the cache on reads.
- Test that the cache is populated after a fresh download.
- These may require mocking `FileHelper::downloadFile()` or using a test fixture.

### Step 9: Run PHPStan

Verify the new code passes static analysis at level 5:

```bash
vendor/bin/phpstan analyse
```

### Step 10: Manual verification

Test with a real comic (e.g., Kill Six Billion Demons):

1. Click "Refresh Index" — all pages downloaded, cache populated.
2. Click "Refresh Index" again immediately — all pages served from cache, zero HTTP requests.
3. Wait > 24 hours (or manually lower TTL for testing) — stale cache entries are re-fetched.
4. Click "Clear Cache" — cache files deleted. Next "Refresh Index" downloads everything fresh.

### Step 11: Update project manifest documents

Update the following manifest files to reflect the new class and changed flows:

- [docs/agents/project-manifest/file-tree.md](../../../../docs/agents/project-manifest/file-tree.md) — Add `Comics/FetchCache.php` entry and `cache/` directory under per-comic storage.
- [docs/agents/project-manifest/api-surface.md](../../../../docs/agents/project-manifest/api-surface.md) — Add `FetchCache` class signature; update `Fetcher` with new methods (`setBypassCache`, `getCache`, `resolveTTL`).
- [docs/agents/project-manifest/data-flows.md](../../../../docs/agents/project-manifest/data-flows.md) — Update "Index Fetching" and "Episode Fetching" flows to include cache lookup/store steps.
- [docs/agents/project-manifest/storage.md](../../../../docs/agents/project-manifest/storage.md) — Document the `cache/` directory and `cache-manifest.json` format.
- [docs/agents/project-manifest/constraints.md](../../../../docs/agents/project-manifest/constraints.md) — Add note about cache TTL defaults and the respectful-use motivation.

## Dependencies

- `AppUtils\FileHelper` — for directory creation and file operations (already in use).
- `AppUtils\FileHelper\JSONFile` — for manifest I/O (already available via `application-utils`).
- No new Composer dependencies required.

## Required Components

| Component | Status | Location |
|---|---|---|
| `FetchCache` class | **New** | `assets/classes/Comics/FetchCache.php` |
| `Fetcher` class | Modified | `assets/classes/Fetcher.php` |
| `FetchPage` class | Modified | `assets/classes/Page/FetchPage.php` |
| `Comic` class | Modified (minor) | `assets/classes/Comic.php` |
| `FetchCacheTests` | **New** | `tests/TestSuites/FetchCacheTests.php` |
| `FetcherCacheTests` | **New** | `tests/TestSuites/FetcherCacheTests.php` |
| Manifest docs (5 files) | Modified | `docs/agents/project-manifest/` |

## Assumptions

- The `$type` parameter passed to `fetchHTML()` is unique per URL within a comic's scope. This is already the established convention (e.g., `index`, `book-1`, `book-1-archive-2`, `episode-42`), and no source driver reuses a type for a different URL.
- `FileHelper::downloadFile()` is the only mechanism for HTTP-fetching HTML. No other code path downloads HTML outside of `Fetcher::fetchHTML()`.
- The `temp/` folder is only used for HTML file caching by `Fetcher`. If other code writes to `temp/`, that code is unaffected by these changes (the `temp/` folder itself is not deleted, only `.html` files within it were being managed).
- Cache invalidation via "Clear Cache" is acceptable as a manual operation. There is no need for automatic cross-session invalidation (e.g., based on detected new episode counts).

## Constraints

- **No database.** Cache metadata is stored in `cache-manifest.json` (JSON file on disk).
- **PHP 8.4+.** Use typed properties, typed constants, match expressions where appropriate.
- **Respectful website usage.** The entire purpose of this feature is to reduce unnecessary HTTP requests. The implementation must guarantee that cached content is served without contacting the origin server.
- **Backward compatibility.** Source drivers must not require any changes. The `Fetcher::fetchHTML()` signature remains identical.
- **File naming.** The `$type` value is used directly as the filename (e.g., `{type}.html`). Since types like `book-1-archive-2` and `episode-42` contain only alphanumeric characters and hyphens, this is safe. No sanitization needed.

## Out of Scope

- **HTTP conditional requests** (ETag / If-Modified-Since). Deferred to a future iteration. The manifest format accommodates future `etag` and `lastModified` fields.
- **Per-comic configurable TTLs.** Defaults are sufficient for v1. Could be added to `settings.json` or `ComicSourceInterface` later.
- **Image caching.** Images are downloaded once per episode and stored permanently. No caching layer needed.
- **UI indicators for cache status.** Showing "Cached 2 hours ago" or a "Cached" badge on the fetch page is a UX enhancement for a later iteration.
- **Differential/smart refresh** (only check the latest archive page). This is a driver-level optimization that complements caching but is a separate feature.
- **Cache size management / eviction.** For local-only use with tens of comics, the cache will be small (a few MB at most). No size-based eviction is needed.

## Acceptance Criteria

1. **Cache hit avoids HTTP:** When `Fetcher::fetchHTML()` is called for a type whose cache entry exists and is within TTL, no HTTP request is made and the cached HTML is returned.
2. **Cache miss triggers download:** When no cache entry exists or the TTL has expired, the page is downloaded via HTTP and stored in the cache.
3. **Bypass mode works:** Setting `setBypassCache(true)` causes all `fetchHTML()` calls to skip cache reads, forcing fresh downloads, while still writing the results to cache.
4. **Clear cache works:** Calling `invalidateAll()` removes all cached HTML files and resets the manifest. Subsequent `fetchHTML()` calls trigger fresh downloads.
5. **Source drivers unchanged:** No modifications are needed to any source driver class. The caching is fully transparent.
6. **FetchPage actions updated:** "Refresh Index" uses bypass mode; "Clear Cache" clears the persistent cache.
7. **Episode caching works:** Episode page HTML is cached with a 30-day TTL. Subsequent fetches for the same episode (e.g., metadata refresh with `setDownloadImages(false)`) use the cached HTML.
8. **Tests pass:** Unit tests for `FetchCache` cover all core operations (get, put, invalidate, TTL expiry). PHPStan passes at level 5.
9. **Manifest documents updated:** All affected manifest files reflect the new class, changed methods, and updated data flows.

## Testing Strategy

### Unit Tests (`FetchCacheTests`)

Test the `FetchCache` class in isolation using a temporary directory:

- Round-trip: `put()` then `get()` returns the same HTML.
- TTL enforcement: Manipulate `cachedAt` in the manifest, verify `get()` returns `null` for expired entries.
- `invalidate()`: Single-entry removal.
- `invalidateAll()`: Full cache wipe.
- `isValid()`: Correctness for valid, expired, and missing entries.
- Manifest integrity: Verify `cache-manifest.json` contains expected structure after operations.

### Integration Tests (`FetcherCacheTests`)

Test the `Fetcher`-level integration:

- `resolveTTL()` returns correct values for `episode-*` vs. other types.
- `setBypassCache()` flag is respected in `fetchHTML()`.
- Verify that after a `fetchHTML()` call, a corresponding cache entry exists.

### Static Analysis

Run PHPStan at level 5 to catch type errors and missing methods.

### Manual Smoke Test

Use Kill Six Billion Demons (multi-page index) as the integration test comic:
1. First "Refresh Index": All ~30 HTTP requests made, cache files created.
2. Second "Refresh Index": 0 HTTP requests, instant completion, pages served from cache.
3. "Clear Cache" → "Refresh Index": All requests made again.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Stale cache serves outdated episode list** | 1-day default TTL for index pages is conservative. "Refresh Index" with bypass mode always fetches fresh. Users can also "Clear Cache" manually. |
| **Cache files accumulate over time** | Cache is per-comic and bounded by the number of index/episode pages. Even KSBD with ~30 archive pages produces only ~30 small HTML files. Not a storage concern. |
| **`$type` collision between different URLs** | The existing convention already ensures unique types per URL. No driver reuses a type for a different URL. The manifest also stores the URL for diagnostic verification. |
| **Manifest corruption** (e.g., interrupted write) | Use atomic writes via a temporary file + rename. `JSONFile::putData()` from `application-utils` already handles this safely. |
| **Backward compatibility with existing temp files** | During migration, `clearCachedDownloads()` clears both temp and cache directories. After migration stabilizes, the temp-file cleanup can be removed. |
| **Test isolation** | Tests use a temporary directory created in `setUp()` and deleted in `tearDown()`, avoiding interference with production cache directories. |
