# Plan

## Summary

Create a standalone CLI PHP script at `tools/populate-index.php` that runs the "Populate Index" operation across all registered comics in a single batch, eliminating the need to trigger it individually for each comic through the web UI. To avoid duplicating logic, a new dedicated `IndexPopulator` service class will be introduced. Both the existing `PopulateIndex` action and the `BaseComicSource::autoPopulateFromEpisodes()` method (which currently contain equivalent but divergent logic) will be refactored to delegate to this class. The CLI script will also use the same class.

---

## Architectural Context

**Three places where population logic currently lives:**

| Location | File | Mode | Notes |
|---|---|---|---|
| `PopulateIndex::_run()` | `assets/classes/Actions/Action/PopulateIndex.php` | File-based (reads+writes `index.json`) | Uses Episode **getters** (`getChapter()`, `getTitle()`, `getAltText()`); supports `$force` flag via `$this->request`. |
| `BaseComicSource::autoPopulateFromEpisodes()` | `assets/classes/Comics/Sources/BaseComicSource.php` | In-memory (works on `$this->indexEntries`) | Reads `info.json` **directly** via `$episode->getInfoFile()->parse()` to avoid getter fallback values; no force mode. |
| CLI script (new) | `tools/populate-index.php` | File-based (same as action) | Needs force flag from `$argv`, not from `$_REQUEST`. |

**Key divergence between the two existing implementations:**

- `PopulateIndex` reads via Episode getters. `getChapter()` / `getTitle()` / `getAltText()` ultimately read from `info.json` but may return fallback empty strings, and the action guards against non-existent episodes with `$comic->episodeExists()`.
- `autoPopulateFromEpisodes` reads `info.json` raw (explicitly to avoid fallbacks) and guards via `$episode->infoFileExists()`.
- Both use `MetadataFilter::filterText()` for normalisation and share the same set of metadata keys: `chapter`, `title`, `altText`.

**Canonical resolution:** the `autoPopulateFromEpisodes` approach (direct `info.json` read, `infoFileExists()` guard) is more correct and should be the basis for the unified class. `PopulateIndex` should be updated to use this approach too.

**Other relevant files:**

| File | Role |
|---|---|
| `bootstrap.php` | App initializer. Already used by PHPUnit via `tests/bootstrap.php`, validating CLI execution. |
| `assets/classes/Comics/Index/ComicsIndex.php` | Sibling class in the `WebcomicsBuilder\Comics\Index` namespace — the natural home for the new service class. |
| `assets/classes/Comics.php` | `Comics::getInstance()->getComics()` — returns the full `Comic[]` array of all registered comics. |
| `assets/classes/Comics/Sources/MetadataFilter.php` | `MetadataFilter::filterText()` — text normalisation, used by both existing impls and will be used by the new class. |

**No `tools/` directory exists yet** — it must be created as a new top-level folder.

---

## Approach / Architecture

### New class: `IndexPopulator`

Location: `assets/classes/Comics/Index/IndexPopulator.php`
Namespace: `WebcomicsBuilder\Comics\Index`

The class accepts a `Comic` and exposes two public methods:

```php
class IndexPopulator
{
    public function __construct(Comic $comic) {}

    /**
     * Populates metadata in-place into the provided index entries array.
     * Used by BaseComicSource during an index fetch (in-memory entries, no force needed).
     *
     * @param array<int,array<string,mixed>> &$entries
     * @param bool $force Overwrite existing non-empty values when true.
     * @return int Number of entries where at least one field was updated.
     */
    public function populateEntries(array &$entries, bool $force = false): int {}

    /**
     * Reads the comic's index.json, populates it, and saves it back.
     * Used by the PopulateIndex action and the CLI script.
     *
     * @param bool $force Overwrite existing non-empty values when true.
     * @return int Number of entries where at least one field was updated.
     */
    public function populateIndexFile(bool $force = false): int {}
}
```

`populateIndexFile` is a thin wrapper: it reads `index.json -> parse()`, delegates to `populateEntries`, then calls `putData()`.

Both methods use the `autoPopulateFromEpisodes` approach internally:
- Read `info.json` directly via `$episode->getInfoFile()->parse()` (avoids getter fallback values).
- Guard via `$episode->infoFileExists()` (not `$comic->episodeExists()`).
- Normalise with `MetadataFilter::filterText()`.
- The canonical metadata key list (`chapter`, `title`, `altText`) is a `private const array` on the class, making it the single authoritative place to add new index metadata fields.

### Refactoring targets

1. **`PopulateIndex::_run()`** — replace ~30 lines of iteration logic with:
   ```php
   $force = $this->request->getBool(self::REQUEST_PARAM_FORCE);
   (new IndexPopulator($this->comic))->populateIndexFile($force);
   ```

2. **`BaseComicSource::autoPopulateFromEpisodes()`** — replace ~40 lines of iteration logic with:
   ```php
   (new IndexPopulator($comic))->populateEntries($this->indexEntries);
   ```
   The private method is retained for readability; only its body changes. The call site in `fetchIndex()` is unchanged.

### CLI script: `tools/populate-index.php`

- Bootstraps via `require_once __DIR__.'/../bootstrap.php'`.
- Parses `$argv` for `--force` / `-f` and `--comic=<alias>`.
- Validates `--comic` alias against `Comics::getInstance()->aliasExists()` if provided; exits with code `1` and an error message if unknown.
- Iterates `Comics::getInstance()->getComics()` (filtered by alias if supplied).
- For each comic: skip if `!$comic->hasIndex()`, otherwise call `(new IndexPopulator($comic))->populateIndexFile($force)`.
- Prints one line per comic and a final summary (processed / skipped / episodes updated).
- Exits with code `0` on success, `1` on fatal error.

---

## Rationale

- **Single source of truth** — the metadata key list and population algorithm live in one class. Adding a new index metadata field requires a change in exactly one place.
- **Clean separation of concerns** — `IndexPopulator` is a pure data-transformation service with no HTTP, no session, no request coupling. It can be called from web actions, CLI scripts, or future automated processes without modification.
- **Direct `info.json` read** is chosen over Episode getters because `autoPopulateFromEpisodes` already established and commented this as the correct approach ("avoid fallback values from getters"). Aligning `PopulateIndex` to this behaviour is a correctness improvement, not just a refactor.
- **`populateEntries` with a reference parameter** is chosen over returning a new array to match the in-memory mutation pattern already used by `BaseComicSource` (`$this->indexEntries` is modified by reference), keeping the `fetchIndex` call site unchanged.
- **CLI force flag from `$argv`** — `IndexPopulator` accepts a plain `bool`, so neither `$_REQUEST` nor `Request::getInstance()` is needed in the CLI script. The web context just passes `$this->request->getBool(...)`.

---

## Detailed Steps

1. **Create `tools/` directory** at the project root.

2. **Create `assets/classes/Comics/Index/IndexPopulator.php`**:
   - Namespace `WebcomicsBuilder\Comics\Index`.
   - Constructor: `public function __construct(private readonly Comic $comic)`.
   - `private const array METADATA_KEYS = ['chapter', 'title', 'altText'];`
   - Implement `populateEntries(array &$entries, bool $force = false): int` — the unified algorithm from `autoPopulateFromEpisodes`, extended to support `$force`.
   - Implement `populateIndexFile(bool $force = false): int` — reads `index.json -> parse()`, calls `populateEntries`, then `putData()`.

3. **Run `composer dump-autoload`** — required because a new class file was added to `assets/classes/`.

4. **Refactor `PopulateIndex::_run()`** — replace the loop body with a call to `IndexPopulator::populateIndexFile()`.

5. **Refactor `BaseComicSource::autoPopulateFromEpisodes()`** — replace the loop body with a call to `IndexPopulator::populateEntries()`. The private method itself is retained; only its body changes.

6. **Create `tools/populate-index.php`** — CLI script as described in the Approach section.

7. **Update `docs/agents/project-manifest/file-tree.md`** — add the `tools/` entry and the `IndexPopulator.php` entry under `Comics/Index/`.

8. **Update `docs/agents/project-manifest/api-surface.md`** — add `IndexPopulator` to the `WebcomicsBuilder\Comics\Index` section.

9. **Update `docs/agents/project-manifest/data-flows.md`** — add a "CLI Batch Tools" section.

---

## Dependencies

- `bootstrap.php` — already present; no changes needed.
- `Comics`, `Comic`, `Episode`, `MetadataFilter` — already autoloaded.
- `composer dump-autoload` — required after adding `IndexPopulator.php` (classmap autoloading).
- No new Composer packages.

---

## Required Components

| Component | Status | Path |
|---|---|---|
| `tools/` directory | **New** | `tools/` |
| `IndexPopulator` class | **New** | `assets/classes/Comics/Index/IndexPopulator.php` |
| `tools/populate-index.php` | **New** | `tools/populate-index.php` |
| `PopulateIndex::_run()` | **Refactor** | `assets/classes/Actions/Action/PopulateIndex.php` |
| `BaseComicSource::autoPopulateFromEpisodes()` | **Refactor** | `assets/classes/Comics/Sources/BaseComicSource.php` |
| `docs/agents/project-manifest/file-tree.md` | **Update** | add `tools/` and `IndexPopulator.php` entries |
| `docs/agents/project-manifest/api-surface.md` | **Update** | add `IndexPopulator` class signature |
| `docs/agents/project-manifest/data-flows.md` | **Update** | add CLI batch flow section |

---

## Assumptions

- PHP is available on the `PATH` (`php tools/populate-index.php`).
- `config-local.php` is present (required by `bootstrap.php`; user already has a working install).
- `bootstrap.php` is located using `__DIR__` so it is path-independent.
- `session_start()` inside `bootstrap.php` does not cause fatal errors in CLI — already validated by PHPUnit using the same bootstrap.
- `getIndexFile()->parse()` returns an empty array for a missing/empty index file; `$comic->hasIndex()` in the CLI script provides an earlier guard.
- `Episode::getInfoFile()->parse()` returns an empty array or partial array if `info.json` is absent or malformed — the code guards with `infoFileExists()` already.

---

## Constraints

- **PHP 8.4+** — `declare(strict_types=1)`, typed properties, `readonly` promoted constructor parameters.
- **No database** — all reads/writes are `index.json` via `JSONFile`. Satisfied.
- **No new PHP framework** — `IndexPopulator` is a plain service class; CLI script is plain PHP.
- **No new Composer packages** — satisfied.
- `MetadataFilter::filterText()` must be used (not raw trimming) to maintain normalisation parity.
- `composer dump-autoload` must be run after adding `IndexPopulator.php` (classmap autoloading).

---

## Out of Scope

- Fetching (downloading) missing episode images — this script only populates existing index metadata.
- Re-fetching the remote archive index (`Fetcher::fetchIndex()`).
- Scheduled / cron execution setup — left to the operator.
- Interactive prompts or progress bars — plain line-by-line STDOUT output is sufficient.
- A `--dry-run` flag (could be a follow-up).
- Converting the `Request` dependency in `BaseAction` to be CLI-aware.

---

## Acceptance Criteria

- `php tools/populate-index.php` runs without errors against all comics in `storage/comics.json`.
- For each comic with an existing index, chapter/title/altText values are populated if absent (matching the default non-forced UI behavior).
- `php tools/populate-index.php --force` overwrites existing index values (matching the "forced" UI button behavior).
- `php tools/populate-index.php --comic=grrl-power` limits execution to that comic; all others are skipped.
- Comics with no `index.json` are skipped with a printed notice rather than causing an error.
- A summary is printed at the end: total processed / skipped / episodes updated.
- The script exits with code `0` on success and `1` if a fatal error occurs (e.g., unknown alias to `--comic`).
- The "Populate now" and "Populate now (forced)" buttons in the web UI behave identically to before.
- Index fetching via the web UI continues to auto-populate metadata as before.
- `vendor/bin/phpstan analyse` passes at level 5 with no new errors.
- The existing PHPUnit test suite passes without modification.
- The metadata key list (`chapter`, `title`, `altText`) exists in exactly **one** place in the codebase: `IndexPopulator::METADATA_KEYS`.

---

## Testing Strategy

- **Manual smoke test (CLI):** Run against a comic with a populated index; verify STDOUT output and that no data is corrupted.
- **Force flag test:** Clear a chapter field from an index entry manually; run with `--force` and verify it is restored. Run without `--force` and verify an already-populated field is not overwritten.
- **Alias filter test:** Run with `--comic=grrl-power`; confirm only that comic's `index.json` changes.
- **No-index test:** Temporarily rename a comic's `index.json`; verify the script skips with a notice rather than crashing.
- **Web UI regression:** Trigger "Populate now" and "Populate now (forced)" in the UI; verify results are identical to pre-refactor behavior.
- **Index fetch regression:** Re-fetch a comic's index in the UI; verify metadata is still auto-populated into the index file.
- **PHPStan:** `vendor/bin/phpstan analyse tools/ assets/classes/Comics/Index/IndexPopulator.php` — no errors at level 5.
- A dedicated PHPUnit test for `IndexPopulator` is **recommended** but not strictly required for this ticket. At minimum, the existing test suite must continue to pass.

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **Behavioural difference between old `PopulateIndex` (getters) and new `IndexPopulator` (direct info.json read)** | For fetched episodes the values are identical. The only divergence is episodes where a folder exists but `info.json` is absent — the old action's `episodeExists()` guard could return `true` while `infoFileExists()` returns `false`. The new code will skip those entries, which is the safer behaviour. |
| **`session_start()` warning in CLI** | Already mitigated by PHPUnit using the same bootstrap. If noise appears, suppress with `error_reporting()` at the top of the CLI script only — not in `bootstrap.php`. |
| **`$this->indexEntries` reference mutation in `BaseComicSource`** | `populateEntries()` takes `array &$entries` by reference — mutation semantics are preserved. |
| **Logic divergence if either caller is updated independently in future** | Eliminated by the refactor: both callers delegate to `IndexPopulator`. |
| **`composer dump-autoload` forgotten** | The plan explicitly lists it as a mandatory step (step 3). The engineer must verify the class is autoloaded before testing. |
