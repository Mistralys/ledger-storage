# Plan

## Summary

Close the four carry-over debt items flagged in the
`2026-03-05-fetch-caching` synthesis. Two medium-priority defects affect
correctness under adverse conditions (`Comic::getCacheFolder()` returns the
wrong path; `FetchCache::put()` silently ignores write failures). Two
low-priority cleanups tighten the public API (`FetchCache::get()` accepts an
unused `$url` parameter; `FetchCache::isValid()` does not verify that the HTML
file actually exists). All four must be resolved together because they share the
same two source files and tests, and a single PHPStan + PHPUnit run should
remain green on completion.

---

## Architectural Context

The caching system introduced by the previous plan lives in three files:

| File | Role |
|---|---|
| `assets/classes/Comics/FetchCache.php` | Persistent cache class (all four issues touch this file) |
| `assets/classes/Comic.php` | `getCacheFolder()` helper — wrong path (Issue 1) |
| `assets/classes/Fetcher.php` | Wires `Comic` → `FetchCache`; call site for `get()` (Issues 1, 3) |

Tests are in:

| File | Role |
|---|---|
| `tests/TestSuites/FetchCacheTests.php` | 19 unit tests for `FetchCache` (Issues 1, 2, 3, 4) |
| `tests/TestSuites/FetcherCacheTests.php` | 10 integration tests for `Fetcher` cache wiring (Issues 1, 3) |

Path relationships that drive Issue 1:

```
Comic::getFolder()          → storage/comics/{alias}/          (FolderInfo)
Comic::getStorageFolder()   → storage/comics/{alias}/downloaded/   (string)
Comic::getCacheFolder()     → storage/comics/{alias}/cache/         (WRONG — uses getFolder())
Actual cache on disk        → storage/comics/{alias}/downloaded/cache/
FetchCache::__construct()   → receives parent folder, appends /cache internally
Fetcher passes              → $this->folder  (= getStorageFolder())  → correct today, but
                               bypasses getCacheFolder() entirely
```

Manifest documentation that must be cleaned up after the fixes:
- `docs/agents/project-manifest/api-surface.md` — `FetchCache::get()` signature, constructor param, new constant, `isValid()` note
- `docs/agents/project-manifest/constraints.md` — known-debt callout block for Issues 1 & 2
- `docs/agents/project-manifest/storage.md` — `getCacheFolder()` discrepancy warning

---

## Approach / Architecture

### Issue 1 — `getCacheFolder()` path mismatch (Medium)

**Chosen approach: option (b) from synthesis — "single source of truth".**

1. Fix `Comic::getCacheFolder()` to return `$this->getStorageFolder() . '/cache'` — the
   correct, fully-qualified path to the cache directory.
2. Simultaneously change `FetchCache::__construct()` to accept the **full cache
   folder path** (remove the internal `/cache` appension; rename parameter to
   `$cacheFolderPath`). This makes the constructor semantics unambiguous: the
   caller decides the root, the class does not silently append a subdirectory.
3. Change `Fetcher::__construct()` to construct `FetchCache` with
   `$this->comic->getCacheFolder()` instead of `$this->folder` — `getCacheFolder()`
   becomes the single authoritative source of the cache path.
4. Update `FetchCacheTests::createCache()` to pass `$this->tempDir . '/cache'`
   instead of `$this->tempDir` — all existing assertions that reference
   `$this->tempDir . '/cache/html/...'` and `$this->tempDir . '/cache/cache-manifest.json'`
   remain valid without further changes.

> **Why not option (a)?** Option (a) would fix `getCacheFolder()` while leaving
> `Fetcher` passing the parent folder directly, so `getCacheFolder()` would still
> not be used by any I/O code. That leaves the latent trap open. Option (b)
> makes the wiring explicit and testable.

### Issue 2 — `put()` unguarded write (Medium)

Guard `file_put_contents()` and throw `WebcomicsBuilderException` on failure
before updating the manifest. Add a new error constant on `FetchCache` itself
(following the `Comic::ERROR_NO_SUCH_COMIC` pattern of per-class constants).
Add `WebcomicsBuilderException` to the `use` block in `FetchCache.php`.
Add one new test to `FetchCacheTests` that verifies the exception is thrown when
the HTML directory is not writable.

### Issue 3 — Unused `$url` in `get()` (Low)

Remove the `string $url` parameter from `FetchCache::get()`. Update the single
call site in `Fetcher::fetchHTML()`. Remove the `$url` argument from all
`FetchCacheTests` calls to `get()` (approximately seven occurrences).

### Issue 4 — `isValid()` missing file check (Low)

Add a `file_exists()` check before the final `return true` in `isValid()` so
that a manifest entry without a corresponding HTML file on disk correctly returns
`false`. Add one test to `FetchCacheTests` verifying this behaviour.

---

## Rationale

- Fixing the `FetchCache` constructor to accept the full path eliminates the
  dual-path ambiguity and makes `getCacheFolder()` genuinely usable.
- Grouping all four fixes in one plan prevents partial states where manifests
  and api-surface docs are only half-updated.
- No changes to any comic source driver, page class, or AJAX endpoint are
  required — the entire diff is limited to two production classes, two test
  files, and three manifest documents.
- Error constant `260301` (March 2026, sequence 01) follows the project
  `YYMMNN` convention from `AGENTS.md`.

---

## Detailed Steps

### Step 1 — Fix `FetchCache::__construct()` (Issue 1)

File: `assets/classes/Comics/FetchCache.php`

- Rename parameter `$comicStorageFolder` → `$cacheFolderPath`.
- Change the first three assignments:
  ```php
  // Before
  $this->cacheFolder  = rtrim($comicStorageFolder, '/\\') . '/cache';

  // After
  $this->cacheFolder  = rtrim($cacheFolderPath, '/\\');
  ```
  `$this->htmlFolder` and `$this->manifestPath` continue to be derived from
  `$this->cacheFolder` as before — no further changes there.
- Update the constructor docblock to reflect the new semantics.

### Step 2 — Fix `Comic::getCacheFolder()` (Issue 1)

File: `assets/classes/Comic.php`

- Change the method body:
  ```php
  // Before
  return sprintf('%s/cache', $this->getFolder());

  // After
  return $this->getStorageFolder() . '/cache';
  ```

### Step 3 — Wire `Fetcher` to use `getCacheFolder()` (Issue 1)

File: `assets/classes/Fetcher.php`

- In `__construct()`, change:
  ```php
  // Before
  $this->cache = new FetchCache($this->folder);

  // After
  $this->cache = new FetchCache($this->comic->getCacheFolder());
  ```

### Step 4 — Update `FetchCacheTests::createCache()` (Issue 1)

File: `tests/TestSuites/FetchCacheTests.php`

- Change `createCache()`:
  ```php
  // Before
  return new FetchCache($this->tempDir);

  // After
  return new FetchCache($this->tempDir . '/cache');
  ```
- All existing assertions against `$this->tempDir . '/cache/html/...'` and
  `$this->tempDir . '/cache/cache-manifest.json'` remain correct — no other
  test changes are needed for this step.

### Step 5 — Guard `put()` write and add error constant (Issue 2)

File: `assets/classes/Comics/FetchCache.php`

- Add import at the top of the file:
  ```php
  use WebcomicsBuilder\WebcomicsBuilderException;
  ```
- Add a class constant (after the two TTL constants):
  ```php
  public const int ERROR_WRITE_FAILED = 260301;
  ```
- In `put()`, replace the bare `file_put_contents()` call:
  ```php
  // Before
  file_put_contents($htmlFile, $html);

  // After
  if (file_put_contents($htmlFile, $html) === false) {
      throw new WebcomicsBuilderException(
          sprintf('Failed to write cache file: %s', $htmlFile),
          self::ERROR_WRITE_FAILED
      );
  }
  ```

### Step 6 — Add write-failure test (Issue 2)

File: `tests/TestSuites/FetchCacheTests.php`

- Add import `use WebcomicsBuilder\WebcomicsBuilderException;`
- Add one test method after the existing `put()`/`get()` group:
  ```php
  public function testPutThrowsOnWriteFailure(): void
  {
      // Create a FetchCache whose html/ directory is made unwritable.
      $cacheDir = $this->tempDir . '/cache';
      mkdir($cacheDir . '/html', 0777, true);
      chmod($cacheDir . '/html', 0444);   // read-only

      $cache = new FetchCache($cacheDir);

      $this->expectException(WebcomicsBuilderException::class);
      $this->expectExceptionCode(FetchCache::ERROR_WRITE_FAILED);

      try {
          $cache->put('index', 'https://example.com/', '<html></html>');
      } finally {
          // Restore permissions so tearDown() can delete the directory.
          chmod($cacheDir . '/html', 0777);
      }
  }
  ```
  > **Note:** On Windows, `chmod()` with `0444` does not enforce directory
  > write-protection the same way as on Linux. This test will pass on Linux/macOS
  > and should be marked `@requires OS Linux` or skipped on Windows if it causes
  > a false-negative. Add a `#[RequiresOperatingSystem('Linux')]` attribute
  > (PHPUnit 12) or a `markTestSkipped` guard inside the method.

### Step 7 — Remove `$url` from `FetchCache::get()` (Issue 3)

File: `assets/classes/Comics/FetchCache.php`

- Remove the `string $url` parameter from `get()`.
- Update the docblock accordingly.

File: `assets/classes/Fetcher.php`

- Update the `fetchHTML()` call site:
  ```php
  // Before
  $cached = $this->cache->get($type, $url, $ttl);

  // After
  $cached = $this->cache->get($type, $ttl);
  ```

File: `tests/TestSuites/FetchCacheTests.php`

- Remove the second positional `$url` argument from every call to `$cache->get()`.
  Affected calls (all pass a URL string as the second argument):
  - `testPutAndGetRoundTrip`
  - `testGetReturnsNullOnCacheMiss`
  - `testGetReturnsNullWhenTTLExpired`
  - `testGetReturnsContentWhenWithinTTL`
  - `testGetWithNullTTLSkipsExpiryCheck`
  - `testInvalidateRemovesEntry`
  - `testInvalidateAllClearsAllEntries` (three calls)
  - `testInvalidateDeletesHTMLFile` (one call inside `testInvalidateRemovesEntry` via `get()`)

### Step 8 — Add `file_exists()` check to `isValid()` (Issue 4)

File: `assets/classes/Comics/FetchCache.php`

- In `isValid()`, before the final `return true;`, add:
  ```php
  $htmlFile = $this->htmlFolder . '/' . $type . '.html';
  if (!file_exists($htmlFile)) {
      return false;
  }
  ```
- Update the method docblock to remove (or update) any note about the asymmetry
  with `get()`, since both now check file existence.

### Step 9 — Add `isValid()` file-missing test (Issue 4)

File: `tests/TestSuites/FetchCacheTests.php`

- Add one test method after the existing `isValid()` group:
  ```php
  public function testIsValidReturnsFalseWhenHtmlFileDeleted(): void
  {
      $cache = $this->createCache();
      $cache->put('index', 'https://example.com/', '<html></html>');

      // Manually delete the HTML file but leave the manifest intact.
      $htmlFile = $this->tempDir . '/cache/html/index.html';
      $this->assertFileExists($htmlFile);
      unlink($htmlFile);

      // isValid() must now return false because the file is gone.
      $freshCache = $this->createCache();
      $this->assertFalse(
          $freshCache->isValid('index'),
          'isValid() must return false when the HTML file has been deleted externally.'
      );
  }
  ```

### Step 10 — Run PHPStan and PHPUnit

- Run `vendor/bin/phpstan analyse` — expect 0 errors (baseline unchanged).
- Run `vendor/bin/phpunit` — expect all tests pass; total count increases by 2
  (Steps 6 and 9); the `put()` write-failure test may be skipped on Windows.

### Step 11 — Update project manifest documentation

File: `docs/agents/project-manifest/api-surface.md`

- Update `FetchCache::__construct()` parameter name from `$comicStorageFolder`
  to `$cacheFolderPath` and revise its description.
- Add `public const int ERROR_WRITE_FAILED = 260301;` to the `FetchCache`
  constants block.
- Update `FetchCache::get()` signature: remove `string $url` parameter.
- Update `FetchCache::isValid()` note: remove the asymmetry warning; note that
  it now checks both manifest and file existence.

File: `docs/agents/project-manifest/constraints.md`

- Remove the known-debt callout block for `getCacheFolder()` path mismatch.
- Remove the known-debt callout for `put()` unguarded write.
- Remove any warning discouraging the use of `getCacheFolder()` for I/O.

File: `docs/agents/project-manifest/storage.md`

- Remove the `getCacheFolder()` discrepancy warning box.
- Update the `getCacheFolder()` description to reflect the corrected return value.

---

## Dependencies

- PHPUnit `>= 12.5` (already in `vendor/`; needed for `#[RequiresOperatingSystem]`
  or equivalent skip logic in Step 6).
- No new Composer packages required.

---

## Required Components

### Modified files
| File | Changes |
|---|---|
| `assets/classes/Comics/FetchCache.php` | Constructor param rename; `ERROR_WRITE_FAILED`; guarded `put()`; `get()` signature; `isValid()` file check |
| `assets/classes/Comic.php` | `getCacheFolder()` corrected path |
| `assets/classes/Fetcher.php` | Constructor wiring; `get()` call site |
| `tests/TestSuites/FetchCacheTests.php` | `createCache()` path; `$url` args removed; two new tests |
| `docs/agents/project-manifest/api-surface.md` | Constructor param; constant; `get()`; `isValid()` |
| `docs/agents/project-manifest/constraints.md` | Remove debt callouts |
| `docs/agents/project-manifest/storage.md` | Remove discrepancy warning |

### No new files are required.

---

## Assumptions

- The project runs on PHP 8.4 on a local webserver; `chmod()` behaviour on
  Windows for the write-failure test is non-standard, so a skip guard is needed.
- `WebcomicsBuilderException` extends `AppUtils\BaseException`, whose constructor
  accepts `(string $message, int $code)` — consistent with all existing throws
  in the codebase.
- No external caller currently passes `getCacheFolder()` to `FetchCache` or
  relies on the wrong path it returned; confirmed by a codebase-wide search that
  found zero I/O use of `getCacheFolder()` in production code.
- `FetcherCacheTests` stubs `Comic::getStorageFolder()` directly and is unaffected
  by the `getCacheFolder()` change at the stub level; however the `Fetcher`
  constructor will now call `$comic->getCacheFolder()`, so the stub must also
  return a value for `getCacheFolder()`. Update `FetcherCacheTests::createFetcher()`
  to add:
  ```php
  $comic->method('getCacheFolder')->willReturn($this->tempDir . '/cache');
  ```

---

## Constraints

- No database, no framework, no new Composer packages (`constraints.md`).
- Error constant format: `YYMMNN` integer on the owning class (`AGENTS.md`).
- `FetchCache` constructor rename must update the class docblock's `@param` tag.
- The 60–190 s throttle in `fetcher.js` is unrelated and must not be touched.

---

## Out of Scope

- Fixing the 21 pre-existing PHPStan baseline errors.
- Changes to any comic source driver.
- UI or JavaScript changes.
- Adding a per-entry `isValid()` pre-flight guard to `Fetcher::fetchHTML()`.

---

## Acceptance Criteria

- `Comic::getCacheFolder()` returns `storage/comics/{alias}/downloaded/cache`
  (i.e. `getStorageFolder() . '/cache'`).
- `FetchCache::__construct()` receives the full cache folder path; `htmlFolder`
  and `manifestPath` are still derived from it correctly.
- `Fetcher` constructs `FetchCache` via `$this->comic->getCacheFolder()`.
- `FetchCache::put()` throws `WebcomicsBuilderException` (code `260301`) when
  `file_put_contents()` fails; the manifest is not updated in that case.
- `FetchCache::get()` no longer accepts a `$url` parameter.
- `FetchCache::isValid()` returns `false` when the HTML file has been deleted
  externally even if the manifest entry is present and valid.
- `vendor/bin/phpstan analyse` exits with 0 errors.
- `vendor/bin/phpunit` passes for all tests; total count ≥ 60 (58 existing + 2 new).
- All debt callouts in `constraints.md`, `storage.md`, and `api-surface.md` are
  removed or updated to reflect the resolved state.

---

## Testing Strategy

All changes are covered by the existing test harness (`FetchCacheTests`,
`FetcherCacheTests`) with two additions:

| Test | Covers |
|---|---|
| `testPutThrowsOnWriteFailure` (new) | Issue 2 — exception on write failure |
| `testIsValidReturnsFalseWhenHtmlFileDeleted` (new) | Issue 4 — `isValid()` file check |

The `FetcherCacheTests::createFetcher()` stub must be updated to stub
`getCacheFolder()` (Issue 1 / Assumption above), but no new `FetcherCacheTests`
test methods are needed.

Run order: PHPStan first (catches type errors from the signature change), then
PHPUnit.

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **`chmod(0444)` no-op on Windows** makes `testPutThrowsOnWriteFailure` a false-negative | Add `#[RequiresOperatingSystem('Linux')]` attribute or an `os_check` + `markTestSkipped` guard in Step 6 |
| **`FetcherCacheTests` stub missing `getCacheFolder()`** causes a fatal in `Fetcher::__construct()` | Explicitly add `$comic->method('getCacheFolder')` to the stub in `createFetcher()` (documented in Assumptions) |
| **Missed `get()` call sites** after removing the `$url` parameter | `vendor/bin/phpstan analyse` will surface any remaining calls immediately; resolve before committing |
| **Manifest left in inconsistent state on write failure** | Guaranteed by the guard: `saveManifest()` is only reached after a successful `file_put_contents()` |
