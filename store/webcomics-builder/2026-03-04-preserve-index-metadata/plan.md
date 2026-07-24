# Plan

## Summary

When the episode index is re-fetched from a comic's website, the `index.json` file is rewritten with only the core `episode` and `url` keys — discarding any metadata (chapter, title, altText) that was previously populated. This forces the user to manually re-run the `PopulateIndex` action after every index refresh. The plan introduces metadata preservation during index re-fetching, and auto-populates metadata from episode `info.json` files after the index is written, eliminating the manual step entirely.

## Architectural Context

The main components involved:

- [assets/classes/Comics/Sources/BaseComicSource.php](../../assets/classes/Comics/Sources/BaseComicSource.php) — The `fetchIndex()` method orchestrates index rebuilding. It resets `$this->indexEntries` (for non-additive sources), calls `_fetchIndex()` which invokes the source driver's `parseIndex()`, and then writes `$this->indexEntries` to `index.json` via `putData()`. The `registerEpisode()` method creates entries with only `episode` and `url` keys.

- [assets/classes/Comic.php](../../assets/classes/Comic.php) — `handleIndexFetched()` is called after the index is saved, currently only updating the last-fetched timestamp. `getIndexFile()` returns the `JSONFile` for `index.json`.

- [assets/classes/Actions/Action/PopulateIndex.php](../../assets/classes/Actions/Action/PopulateIndex.php) — Manual action that iterates index entries and copies `chapter`, `title`, and `altText` from episode `info.json` files into the index. Contains a `filterText()` method that trims whitespace, strips newlines, and blanks out numeric-only values.

- [assets/classes/Episode.php](../../assets/classes/Episode.php) — Each episode can have metadata (chapter, title, altText, volume, chapterSynopsis) stored in its `info.json` file inside `storage/comics/{alias}/downloaded/{episodeID}/`.

- Source drivers like [assets/classes/Comics/Source/ComicPressSource.php](../../assets/classes/Comics/Source/ComicPressSource.php) — During `parseIndex()`, they call `registerEpisode()` and then fluently set title/chapter on the returned Episode object. These save to `info.json` but NOT to the index entries.

**Current data flow on index re-fetch (non-additive):**
1. `fetchIndex()` resets `$this->indexEntries = []`
2. `_fetchIndex()` → `parseIndex()` → `registerEpisode()` builds entries with only `{episode, url}`
3. `putData()` writes entries to `index.json` — all metadata (chapter, title, altText) is lost
4. User must manually run PopulateIndex to restore metadata

**Current data flow on index re-fetch (additive):**
1. `fetchIndex()` loads existing entries (metadata preserved for existing episodes)
2. `registerEpisode()` adds only new episodes
3. `putData()` writes — existing metadata preserved, but new episodes lack metadata until PopulateIndex is run

## Approach / Architecture

Two complementary changes in `BaseComicSource::fetchIndex()`:

### Change 1: Preserve existing index metadata across re-fetches

Before clearing `$this->indexEntries`, read the current `index.json` and build a lookup map of `episodeID → metadata` (all keys except `episode` and `url`). After `_fetchIndex()` repopulates the entries, iterate the new entries and merge back any matching preserved metadata. This applies to all sources (additive and non-additive) uniformly.

### Change 2: Auto-populate metadata from episode `info.json` files

After the metadata preservation merge and before writing the index, iterate all index entries. For any entry that has no metadata yet (or has only empty metadata), check whether the episode has an `info.json` file with metadata. If so, copy it into the entry. This uses the same logic as `PopulateIndex::filterText()` and the same metadata keys (`chapter`, `title`, `altText`).

This auto-population step replaces the manual `PopulateIndex` run for the common case. The `PopulateIndex` action is preserved for force-overwrite scenarios and backward compatibility.

### Implementation location

All changes are in `BaseComicSource::fetchIndex()` with a new private helper method for the metadata merge/populate logic. No changes to source drivers, Episode, Comic, or the PopulateIndex action.

## Rationale

- **Metadata preservation** is the primary fix. It directly solves the stated pain point: metadata survives index re-fetches without any manual intervention.
- **Auto-population** from `info.json` is a natural follow-up that eliminates the need for *initial* PopulateIndex runs too. Whenever the index is (re-)fetched, metadata is automatically populated from any already-downloaded episodes.
- Placing both changes inside `fetchIndex()` keeps the fix centralized and avoids modifying source driver contracts.
- The `PopulateIndex` action remains useful for force-overwriting metadata, or populating metadata without a full index re-fetch, but it is no longer a required step in the normal workflow.
- The metadata map approach (keyed by episode ID) is efficient and handles re-ordered or partially changed indexes gracefully.

## Detailed Steps

1. **Add a `snapshotIndexMetadata()` private method to `BaseComicSource`** that:
   - Takes a `Comic` parameter.
   - Reads the existing `index.json` (if it exists) into an array.
   - Builds and returns an associative array: `[episodeID => [key => value, ...]]` for all keys that are NOT `episode` or `url`.

2. **Add a `mergePreservedMetadata()` private method to `BaseComicSource`** that:
   - Takes the preserved metadata map (from step 1).
   - Iterates `$this->indexEntries`.
   - For each entry matching an episode ID in the map, merges the preserved metadata into the entry (existing keys in the entry take precedence, so freshly fetched data is never overwritten).

3. **Add an `autoPopulateFromEpisodes()` private method to `BaseComicSource`** that:
   - Iterates `$this->indexEntries`.
   - For each entry, checks whether the episode has a downloaded `info.json` file.
   - For metadata keys (`chapter`, `title`, `altText`), if the key is missing or empty in the entry, reads it from the episode `info.json` and applies `filterText()` (same logic as PopulateIndex).

4. **Extract `filterText()` from `PopulateIndex` into a shared location** — either as a static method on a utility class, or duplicated as a private method on `BaseComicSource` (pragmatic approach to avoid coupling).

5. **Modify `BaseComicSource::fetchIndex()`** to:
   - Call `snapshotIndexMetadata()` BEFORE resetting `$this->indexEntries`.
   - After `_fetchIndex()` completes, call `mergePreservedMetadata()`.
   - After merging, call `autoPopulateFromEpisodes()`.
   - Then proceed with `putData()` as before.

6. **Update `PopulateIndex`** description/UI text to clarify it's now primarily for force-overwriting, since auto-population handles the common case.

7. **Update manifest documents** as needed.

## Dependencies

- `AppUtils\FileHelper\JSONFile` — already used for reading/writing JSON files.
- `Episode` class — for reading `info.json` metadata.
- `Comic` class — for resolving episode existence and storage paths.

## Required Components

- **Modified:** [assets/classes/Comics/Sources/BaseComicSource.php](../../assets/classes/Comics/Sources/BaseComicSource.php) — Core fix location.
- **Modified (optional):** [assets/classes/Actions/Action/PopulateIndex.php](../../assets/classes/Actions/Action/PopulateIndex.php) — Updated description text.
- **Updated docs:** `api-surface.md`, `data-flows.md` — Reflect the new auto-populate behavior.

## Assumptions

- Episode IDs are stable across index re-fetches (same comic page = same episode ID). This is already implicitly assumed by the additive index logic.
- The metadata keys to auto-populate from `info.json` are: `chapter`, `title`, `altText`. These match what `PopulateIndex` currently handles.
- The `filterText()` logic (trim, strip newlines, blank out numeric-only strings) should be preserved for auto-populated values.

## Constraints

- No database usage; all data is in JSON files per project constraints.
- Must not alter the `ComicSourceInterface` contract or require changes to individual source driver implementations.
- Must not change episode `info.json` write behavior.
- Must be backward-compatible: indexes without metadata continue to work; `PopulateIndex` action is preserved.

## Out of Scope

- Modifying how source drivers set metadata during `parseIndex()` (e.g., making `registerEpisode` accept metadata parameters).
- Auto-populating metadata during `fetchEpisode()` / `handleEpisodeFetched()` (per-episode index writes would be expensive with large indexes).
- Removing the `PopulateIndex` action (still useful for force-overwrite).
- Changing the index file structure (still a flat array of objects).

## Acceptance Criteria

- Re-fetching a non-additive index preserves all previously populated `chapter`, `title`, and `altText` values in the index entries.
- Re-fetching an additive index continues to work as before (existing entries with metadata are unchanged).
- After an index re-fetch, any episodes that have been downloaded and have metadata in their `info.json` files automatically have that metadata populated in the index — without running `PopulateIndex`.
- New episodes added during a re-fetch (not yet downloaded) have no metadata in the index (expected).
- Running `PopulateIndex` with force mode still overwrites metadata as before.
- Existing unit tests continue to pass.

## Testing Strategy

- **Manual test (primary):** For a comic with populated metadata in its index (e.g., galaxion), re-fetch the index and verify that all chapter/title/altText values are preserved in `index.json`.
- **Manual test (auto-populate):** For a comic with fetched episodes but unpopulated index metadata, fetch the index and verify metadata is automatically populated.
- **Unit test (recommended):** Create a test that:
  1. Sets up a mock index with metadata entries.
  2. Simulates a non-additive re-fetch that rebuilds the index.
  3. Asserts that metadata from the original index is preserved in the rebuilt index.
- **Unit test (recommended):** Create a test that verifies `filterText()` behavior (numeric-only blanking, whitespace trimming).
- **Regression test:** Verify additive indexes still work correctly (existing entries preserved, new entries added).

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Episode ID changes across re-fetches** (e.g., website restructures URLs) | Metadata is matched by episode ID; changed IDs naturally lose their metadata. This is acceptable and consistent with how the rest of the system works. The data is still in `info.json` and can be restored via PopulateIndex. |
| **Performance overhead** of reading existing index before rebuild | Index files are already read for additive sources. The additional read for non-additive sources is a single JSON parse of a file already on disk — negligible compared to HTTP fetching. |
| **Stale metadata persisted** (e.g., website changes a chapter title) | Preserved metadata does NOT overwrite freshly fetched data. Auto-populated data from `info.json` also does not overwrite existing values. The `PopulateIndex` force mode can be used if a metadata refresh is needed. |
| **`filterText()` logic duplication** between `PopulateIndex` and `BaseComicSource` | Keep the duplication minimal (3 lines). If more metadata processing is added later, extract to a shared utility. |
