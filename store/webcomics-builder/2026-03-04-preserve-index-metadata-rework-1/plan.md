# Plan — Preserve Index Metadata: Post-Synthesis Rework

## Summary

This plan addresses the three follow-up tasks identified in the synthesis of the `2026-03-04-preserve-index-metadata` plan:

1. **PHPStan cleanup** — Add type annotations to the new metadata-preservation methods in `BaseComicSource` to resolve the 6 `--level=max` errors they introduced.
2. **`filterText()` deduplication** — Extract the identical `filterText()` implementations from `BaseComicSource` and `PopulateIndex` into a shared static utility class.
3. **Stale episode ID test** — Add a missing test case confirming that snapshot entries for episodes no longer present in the re-fetched index are silently dropped.

---

## Architectural Context

### Relevant files

| File | Role |
|---|---|
| `assets/classes/Comics/Sources/BaseComicSource.php` | Abstract base for all comic source drivers. Contains the 4 new private metadata methods (`snapshotIndexMetadata`, `mergePreservedMetadata`, `autoPopulateFromEpisodes`, `filterText`) and the `fetchIndex()` orchestration. |
| `assets/classes/Actions/Action/PopulateIndex.php` | Action class that manually populates index metadata from episode info files. Contains a duplicate `filterText()` private method. |
| `assets/classes/Comics/Sources/` | Source driver subsystem directory. Already has a `Traits/` subdirectory for shared interface/trait pairs. |
| `tests/TestSuites/Comics/BaseComicSourceMetadataTests.php` | PHPUnit test suite (14 tests) covering the metadata-preservation feature. |
| `tests/TestClasses/TestableBaseComicSource.php` | Reflection-based test helper exposing `BaseComicSource` private methods. |
| `docs/agents/project-manifest/api-surface.md` | Manifest documenting public (and selected private) API signatures. |
| `docs/agents/project-manifest/file-tree.md` | Annotated directory structure. |
| `docs/agents/project-manifest/constraints.md` | Project conventions and rules. |
| `phpstan.neon` | PHPStan config — currently level 5. The errors targeted here appear at `--level=max`. |

### Current PHPStan situation

- PHPStan is configured at **level 5** (`phpstan.neon`). At this level, `BaseComicSource.php` is clean — zero errors.
- At `--level=max`, the file has **18 errors total**: 6 are directly attributable to the new metadata methods (lines 200, 206, 208, 214, 218, 299); the remaining 12 are pre-existing.
- The 6 new errors all stem from `JSONFile::getData()` returning `mixed`/`array<int|string, mixed>`, which PHPStan cannot narrow without explicit type assertions or `@var` annotations.

### `filterText()` duplication

Both copies are identical:

```php
private function filterText(string $text): string
{
    $text = trim($text);
    $text = str_replace(array("\r", "\n"), '', $text);

    if (is_numeric($text)) {
        return '';
    }

    return $text;
}
```

`BaseComicSource` calls it in `autoPopulateFromEpisodes()` (line 299). `PopulateIndex` calls it in `_run()` (lines 79, 80, 84).

### Existing traits directory

`assets/classes/Comics/Sources/Traits/` already contains shared interface/trait pairs (`CardNamePropertyInterface`/`Trait`, `DomainNamePropertyInterface`/`Trait`). However, `filterText()` is not a trait-style behavior (it's stateless text normalization), so a static utility class is more appropriate.

---

## Approach / Architecture

### Task 1: PHPStan Cleanup

Add `@var` type annotations (inline `@phpstan-var`) inside `snapshotIndexMetadata()` and `autoPopulateFromEpisodes()` to narrow the `mixed` returns from `JSONFile::getData()` and `JSONFile::parse()`. This approach:

- Uses the standard PHPStan pattern already used elsewhere in the codebase.
- Does not require changing `JSONFile`'s generic return type (which is a third-party library class).
- Is confined to the two methods, with no API or behavior changes.

The specific errors and their fixes:

| Line | Error | Fix |
|---|---|---|
| 200 | Cannot access offset `'episode'` on `mixed` | Add `@var array{episode?: string, url?: string} $entry` before the foreach in `snapshotIndexMetadata()` |
| 206 | Invalid type `mixed` for foreach | Same annotation as above (the `$entry` loop variable) |
| 208 | Invalid array key type `mixed` | Covered by narrowing `$entry` and `$key`/`$value` types |
| 214 | Invalid array key type `mixed` | Covered by the same narrowing |
| 218 | Return type mismatch `array<array<mixed>>` vs `array<string, array<string, mixed>>` | Assigning the `string` key to `$map` with the narrowed `$episodeID` resolves this |
| 299 | Cannot cast `mixed` to string | Add `@var array<string, mixed> $infoData` before the `parse()` call in `autoPopulateFromEpisodes()` |

### Task 2: `filterText()` Extraction

Create a new static utility class `WebcomicsBuilder\Comics\Sources\MetadataFilter` in `assets/classes/Comics/Sources/MetadataFilter.php`. This class will:

- Contain a single `public static function filterText(string $text): string` method.
- Be used by both `BaseComicSource::autoPopulateFromEpisodes()` and `PopulateIndex::_run()`.
- Remove the private `filterText()` method from both classes.

**Why a class in `Sources/`, not a trait?**  
- `filterText()` is pure, stateless text normalization — no instance state needed. A static method is idiomatic for this.  
- The `Sources/` namespace is where both callers reside conceptually (source-related metadata processing).  
- The existing `Traits/` directory is for interface/trait pairs that add properties to source drivers, not for standalone utilities.

**Why not a more generic location?**  
- The function is specific to comic index metadata (it blanks numeric-only values). It's not general-purpose text normalization. Keeping it in the `Sources` namespace communicates this scope.

### Task 3: Stale Episode ID Test

Add a single test to `BaseComicSourceMetadataTests.php` that:

1. Sets up index entries with two episodes (`ep001`, `ep002`).
2. Creates a preserved metadata snapshot containing three episodes (`ep001`, `ep002`, `ep003` — where `ep003` is "stale").
3. Calls `mergePreservedMetadata()`.
4. Asserts `ep001` and `ep002` metadata is restored.
5. Asserts the stale `ep003` metadata is silently dropped (not present in any index entry).

This tests the `!isset($preservedMetadata[$episodeID])` guard on line 241 of `BaseComicSource.php`.

### Task 4: Test Helper Update

The `TestableBaseComicSource` class currently exposes `callFilterText()` via reflection. After extraction to `MetadataFilter`, this wrapper is no longer needed — tests can call `MetadataFilter::filterText()` directly. The wrapper should be removed to keep the test helper minimal.

---

## Rationale

- **Type annotations over code changes for PHPStan**: The `getData()` return type is defined by a third-party library (`mistralys/application-utils`). Narrowing it locally with `@var` annotations is the standard, non-invasive approach. Changing the library's return type would cascade across the entire codebase.
- **Static utility over trait**: `filterText()` has no dependency on instance state. Using a trait would add coupling with no benefit. A static method preserves the same call semantics with better discoverability.
- **MetadataFilter naming**: Clearly communicates its purpose (metadata text filtering) and its scope (the comic sources subsystem).
- **Test for stale IDs**: The production code correctly handles this case (the `!isset + continue` in `mergePreservedMetadata`), but without a test, a future refactor could inadvertently break it. One targeted test closes this gap.

---

## Detailed Steps

### Step 1: Create `MetadataFilter` utility class

1. Create `assets/classes/Comics/Sources/MetadataFilter.php`:
   - Namespace: `WebcomicsBuilder\Comics\Sources`
   - Single public static method: `filterText(string $text): string`
   - Copy the implementation from either existing location (they are identical).
   - Add a class-level PHPDoc comment explaining that it normalizes metadata text values for index storage.

2. Run `composer dump-autoload` to register the new class.

### Step 2: Update `BaseComicSource` to use `MetadataFilter`

1. In `assets/classes/Comics/Sources/BaseComicSource.php`:
   - **Remove** the private `filterText()` method (lines 312–325).
   - **Update** the call in `autoPopulateFromEpisodes()` (line 299): replace `$this->filterText(...)` with `MetadataFilter::filterText(...)`.
   - No import needed — same namespace (`WebcomicsBuilder\Comics\Sources`).

### Step 3: Update `PopulateIndex` to use `MetadataFilter`

1. In `assets/classes/Actions/Action/PopulateIndex.php`:
   - **Remove** the private `filterText()` method (lines 97–107).
   - **Add** `use WebcomicsBuilder\Comics\Sources\MetadataFilter;` to the imports.
   - **Update** the three call sites in `_run()` (lines 79, 80, 84): replace `$this->filterText(...)` with `MetadataFilter::filterText(...)`.

### Step 4: Add PHPStan type annotations to `snapshotIndexMetadata()`

1. In `assets/classes/Comics/Sources/BaseComicSource.php`, inside `snapshotIndexMetadata()`:
   - Before `foreach ($indexFile->getData() as $entry)` (line 202), add:
     ```php
     /** @var list<array<string, mixed>> $indexData */
     $indexData = $indexFile->getData();
     ```
   - Change the foreach to iterate `$indexData` instead of `$indexFile->getData()`.
   - Cast `$episodeID` with a string assertion: `$episodeID = (string)($entry['episode'] ?? '');` and guard with `if ($episodeID === '')`.

2. This resolves errors at lines 200, 206, 208, 214, and 218.

### Step 5: Add PHPStan type annotation to `autoPopulateFromEpisodes()`

1. In `assets/classes/Comics/Sources/BaseComicSource.php`, inside `autoPopulateFromEpisodes()`:
   - Before the `$infoData` usage (around line 294), add:
     ```php
     /** @var array<string, mixed> $infoData */
     ```
   - This resolves the error at line 299 (`Cannot cast mixed to string`).

### Step 6: Add the stale episode ID test

1. In `tests/TestSuites/Comics/BaseComicSourceMetadataTests.php`, add:

   ```php
   /**
    * Test: mergePreservedMetadata() silently drops metadata for episode IDs
    * that no longer exist in the current indexEntries.
    */
   public function test_mergePreservedMetadata_dropsStaleEpisodeIDs(): void
   {
       $this->source->setIndexEntries([
           ['episode' => 'ep001', 'url' => 'https://example.com/ep1'],
           ['episode' => 'ep002', 'url' => 'https://example.com/ep2'],
       ]);

       $preserved = [
           'ep001' => ['chapter' => 'Chapter One'],
           'ep002' => ['title'   => 'Second Title'],
           'ep003' => ['chapter' => 'Ghost Chapter', 'title' => 'Ghost Title'],
       ];

       $this->source->callMergePreservedMetadata($preserved);

       $entries = $this->source->getIndexEntries();

       // Existing episodes get their metadata restored.
       $this->assertSame('Chapter One',   $entries[0]['chapter']);
       $this->assertSame('Second Title',  $entries[1]['title']);

       // Stale ep003 must NOT appear in any entry.
       foreach ($entries as $entry) {
           $this->assertNotSame('Ghost Chapter', $entry['chapter'] ?? null);
           $this->assertNotSame('Ghost Title',   $entry['title']   ?? null);
       }
   }
   ```

   Place this test after the existing `test_mergePreservedMetadata_doesNotOverwriteFreshlyFetchedKeys` test (after the Test 4 block), maintaining the numbered test section pattern.

### Step 7: Update `TestableBaseComicSource`

1. In `tests/TestClasses/TestableBaseComicSource.php`:
   - **Remove** the `callFilterText()` wrapper method (lines 72–77), since `MetadataFilter::filterText()` is now public and static.

### Step 8: Update filterText test to call `MetadataFilter` directly

1. In `tests/TestSuites/Comics/BaseComicSourceMetadataTests.php`:
   - **Update** `test_filterText_normalisesText()` to call `MetadataFilter::filterText($input)` instead of `$this->source->callFilterText($input)`.
   - **Add** `use WebcomicsBuilder\Comics\Sources\MetadataFilter;` to the imports.

### Step 9: Run `composer dump-autoload`

Register the new `MetadataFilter` class with the Composer classmap.

### Step 10: Run PHPStan at level 5

Verify zero regressions at the configured analysis level.

### Step 11: Run PHPStan at `--level=max` on `BaseComicSource.php`

Verify the 6 targeted errors are resolved. Document any remaining pre-existing errors (expected: 12 pre-existing errors unchanged).

### Step 12: Run PHPUnit

Full test suite must pass (current: 28 tests). After adding the stale ID test: 29 tests expected.

### Step 13: Update manifest documentation

1. **`api-surface.md`**:
   - Add `MetadataFilter` class entry in the `WebcomicsBuilder\Comics\Sources` section.
   - Remove the `private function filterText(...)` line from `BaseComicSource`'s listing.
   - Remove `filterText` mention from `PopulateIndex` if documented.

2. **`file-tree.md`**:
   - Add `MetadataFilter.php` under `assets/classes/Comics/Sources/`.

---

## Dependencies

- Steps 2 and 3 depend on Step 1 (class must exist before callers are updated).
- Steps 4 and 5 are independent of Steps 1–3 (PHPStan annotations vs. method extraction are orthogonal).
- Step 6 is independent of all other steps.
- Steps 7 and 8 depend on Step 1 (need `MetadataFilter` to exist).
- Steps 10–12 (verification) depend on all code changes being complete.
- Step 13 (docs) should be done last to reflect the final state.

**Parallelizable work:**
- Steps 1–3 (extraction) and Steps 4–5 (annotations) can be done in either order.
- Step 6 (stale test) can be done in parallel with everything else.

---

## Required Components

### New file
| File | Type | Description |
|---|---|---|
| `assets/classes/Comics/Sources/MetadataFilter.php` | PHP class | Static utility class with `filterText()` method |

### Modified files (production)
| File | Change |
|---|---|
| `assets/classes/Comics/Sources/BaseComicSource.php` | Remove private `filterText()`; add `MetadataFilter::filterText()` call; add `@var` annotations for PHPStan |
| `assets/classes/Actions/Action/PopulateIndex.php` | Remove private `filterText()`; add `MetadataFilter::filterText()` calls; add use import |

### Modified files (tests)
| File | Change |
|---|---|
| `tests/TestSuites/Comics/BaseComicSourceMetadataTests.php` | Add stale ID test; update `filterText` test to use `MetadataFilter` directly |
| `tests/TestClasses/TestableBaseComicSource.php` | Remove `callFilterText()` wrapper |

### Modified files (docs)
| File | Change |
|---|---|
| `docs/agents/project-manifest/api-surface.md` | Add `MetadataFilter` class; remove `filterText` from `BaseComicSource` private listing |
| `docs/agents/project-manifest/file-tree.md` | Add `MetadataFilter.php` entry |

---

## Assumptions

- The `JSONFile::getData()` return type (`array<int|string, mixed>`) is defined by the `mistralys/application-utils` library and will not be changed as part of this work.
- The PHPStan configuration will remain at level 5. The `--level=max` fixes are proactive quality improvements, not requirements for CI.
- No other callers of the existing private `filterText()` methods exist beyond the two identified locations.
- `composer dump-autoload` is sufficient to pick up the new `MetadataFilter` class (classmap autoload covers `assets/classes/` recursively).

---

## Constraints

- PHP 8.4+ — the new class should use `declare(strict_types=1)` and typed returns (already the pattern).
- No database, no framework — not applicable to this change but restated per protocol.
- `MetadataFilter` must be placed in `assets/classes/Comics/Sources/` (Composer classmap covers this path).
- Error code format `YYMMNN` if any new exception constants are needed (none expected for this work).

---

## Out of Scope

- Resolving the **12 pre-existing** `--level=max` errors in `BaseComicSource.php` (pre-date this feature).
- Resolving the **21 level-5 errors** across other files in the project.
- Changing the return type of `JSONFile::getData()` in the `mistralys/application-utils` library.
- The defensive `null-coalescing on $entry['episode']` in `mergePreservedMetadata()` (called out as low-priority tech debt; safe in practice since `registerEpisode()` always sets the key).
- Investigating the PHPUnit deprecation notice (unrelated framework-level issue).

---

## Acceptance Criteria

1. A new class `WebcomicsBuilder\Comics\Sources\MetadataFilter` exists with a `public static function filterText(string $text): string` method.
2. `BaseComicSource` no longer has a private `filterText()` method; it calls `MetadataFilter::filterText()`.
3. `PopulateIndex` no longer has a private `filterText()` method; it calls `MetadataFilter::filterText()`.
4. PHPStan at level 5 reports **zero new errors** (21 errors unchanged from baseline).
5. PHPStan at `--level=max` on `BaseComicSource.php` reports **6 fewer errors** than before (from 18 to 12 or fewer).
6. PHPUnit reports **29 tests passing** (28 existing + 1 new stale ID test).
7. The new stale ID test specifically asserts that an episode present in the snapshot but absent from `indexEntries` is silently dropped.
8. `TestableBaseComicSource::callFilterText()` is removed.
9. The `filterText` data-provider test calls `MetadataFilter::filterText()` directly.
10. `api-surface.md` documents the new `MetadataFilter` class.
11. `file-tree.md` lists `MetadataFilter.php` in the correct location.

---

## Testing Strategy

| Scope | Method | Expected Outcome |
|---|---|---|
| `MetadataFilter::filterText()` | Existing data-provider test (`test_filterText_normalisesText`) redirected to static call | 7 data-provider cases pass (identical behavior) |
| Stale episode ID handling | New test `test_mergePreservedMetadata_dropsStaleEpisodeIDs` | Stale `ep003` metadata not injected into any entry |
| Full regression | `vendor/bin/phpunit` | 29/29 tests pass |
| Static analysis (configured) | `vendor/bin/phpstan analyse` (level 5) | 21 errors (no change) |
| Static analysis (strict) | `vendor/bin/phpstan analyse --level=max assets/classes/Comics/Sources/BaseComicSource.php` | ≤12 errors (6 fewer than current 18) |

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **`MetadataFilter` not autoloaded** | Run `composer dump-autoload` after creating the file. Verify by running tests. |
| **Behavioral regression in `filterText()`** | The implementation is copied verbatim. The existing 7 data-provider test cases cover all edge cases. No behavior change. |
| **PHPStan annotations too narrow** | Use `array<string, mixed>` for entries (permissive enough for all index schemas). Verify with `--level=max`. |
| **Other code depends on private `filterText()`** | Both `filterText()` methods are private — no external callers possible. The two known call sites are the only ones. |
| **Test infrastructure change (`callFilterText` removal)** | Only the `filterText` data-provider test uses this wrapper. Updating to the static call is a 1-line change. |
