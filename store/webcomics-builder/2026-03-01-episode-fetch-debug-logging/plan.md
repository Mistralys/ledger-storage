# Plan

## Summary
When fetching a comic episode fails due to a CSS selector not matching anything in the downloaded HTML, the error returned to the UI contains only the bare exception message (e.g., `Selector [#comic P IMG] not found.`). This gives no insight into *what HTML was actually downloaded*, making it very hard to diagnose whether the site layout changed, the download failed silently, an intermediate redirect was served, or something else entirely. The plan introduces a debug-HTML-dump mechanism: when a `DOMHelper` selector lookup fails, the offending HTML is written to a timestamped file in a dedicated `storage/debug/` folder, and the error message sent back to the client is enriched with the path to that file.

---

## Architectural Context

### Error origin
`DOMHelper::domSelectRequireFirst()` ([assets/classes/DOMHelper.php](../../../../assets/classes/DOMHelper.php)) throws a generic `WebcomicsBuilderException` when a CSS selector returns no elements. At the time of throwing, `$this->html` holds the complete HTML string that was being searched.

### Error propagation path
```
DOMHelper::domSelectRequireFirst()      ← throws WebcomicsBuilderException
  └─ BaseComicSource::processEpisode()  ← assets/classes/Comics/Sources/BaseComicSource.php
     └─ BaseComicSource::fetchEpisode()
        └─ Fetcher::fetchEpisode()       ← assets/classes/Fetcher.php
           └─ FetchPage::ajax_fetchEpisode() ← assets/classes/Page/FetchPage.php
              └─ BasePage::handleAJAX() ← assets/classes/UI/BasePage.php
                 └─ sendAJAXError($e->getMessage())   ← becomes {"state":"error","data":{"message":"…"}}
```

### Relevant existing infrastructure
| Asset | Notes |
|-------|-------|
| [assets/classes/DOMHelper.php](../../../../assets/classes/DOMHelper.php) | Holds `$this->html`; exposes `getHTML()`; throws at `domSelectRequireFirst()` |
| [assets/classes/DOM/DOMElementEX.php](../../../../assets/classes/DOM/DOMElementEX.php) | Only existing file in the `DOM/` sub-namespace — confirms the namespace `WebcomicsBuilder\DOM` exists |
| [assets/classes/Comics/Sources/BaseComicSource.php](../../../../assets/classes/Comics/Sources/BaseComicSource.php) | Has `fetchEpisode()` and `processEpisode()`; already maintains a `protected string[] $log` array with `log()` / `getLog()` |
| [assets/classes/UI/BasePage.php](../../../../assets/classes/UI/BasePage.php) | `handleAJAX()` wraps the AJAX method in a `try/catch (Throwable $e)` and delegates to `sendAJAXError($e->getMessage())` |
| [assets/classes/WebcomicsBuilderException.php](../../../../assets/classes/WebcomicsBuilderException.php) | Base exception for the project |
| [.gitignore](../../../../.gitignore) | Already ignores `temp/` and `backup/` — `storage/debug/` must be added |

---

## Approach / Architecture

Introduce a dedicated exception subclass `DOMSelectorException` that carries the HTML that was being searched. `DOMHelper` throws this instead of the generic base exception. `BaseComicSource` catches it, writes the HTML (plus a summary of the context) to a debug file in `storage/debug/`, then re-throws a new exception whose message includes the path to the debug file. The enriched message travels up the existing propagation chain unchanged and is already displayed to the user via `sendAJAXError`.

This approach:
- Keeps the change self-contained in three files plus one new file.
- Requires no new dependencies, no environment flags, no UI changes.
- Leverages the existing exception hierarchy and AJAX error plumbing.
- Does not alter how successful fetches behave.

### New `storage/debug/` folder
Each debug dump consists of two files created atomically in a single directory per failed fetch:
```
storage/debug/{comic-alias}--{episode-id}--{YYYYmmdd-HHiiss}/
    page.html      ← the raw HTML that was being searched
    context.txt   ← selector, episode URL, comic alias, exception message & trace
```
Using a subfolder (rather than flat files) prevents filename collisions when the same episode is retried rapidly, and groups the two artefacts together.

---

## Rationale
- **Subclass instead of adding HTML to the base exception**: Keeps the base exception lean. Only DOM-related failures carry HTML; other exceptions remain unchanged.
- **Catch in `BaseComicSource`, not in `DOMHelper`**: `DOMHelper` is a general utility class and should not know about the filesystem layout. `BaseComicSource` has the right context (comic alias, episode, storage paths).
- **Two-file dump**: A plain `.html` file is immediately openable in a browser. A separate `context.txt` preserves the selector and stack trace without embedding it in the HTML.
- **`storage/debug/` location**: Consistent with `storage/` being the writable data directory. Excluded from version control like other generated data.

---

## Detailed Steps

1. **Create `assets/classes/DOM/DOMSelectorException.php`** *(new file)*
   - Namespace: `WebcomicsBuilder\DOM`
   - Extends: `WebcomicsBuilder\WebcomicsBuilderException`
   - Constructor: accepts `string $selector`, `string $html`, `int $code`, forwards message to parent
   - Public getters: `getSelector() : string`, `getHTML() : string`

2. **Modify `assets/classes/DOMHelper.php`**
   - Add `use WebcomicsBuilder\DOM\DOMSelectorException;`
   - In `domSelectRequireFirst()`: replace the thrown `WebcomicsBuilderException` with `DOMSelectorException`, passing `$selector` and `$this->html`

3. **Modify `assets/classes/Comics/Sources/BaseComicSource.php`**
   - Add `use WebcomicsBuilder\DOM\DOMSelectorException;`
   - In `fetchEpisode()`, wrap the call to `$this->processEpisode(...)` in a try/catch block:
     - Catch `DOMSelectorException $e`
     - Determine debug directory: `APP_ROOT . '/storage/debug/' . $comic->getAlias() . '--' . $episode->getID() . '--' . date('Ymd-His')`  
       *(`APP_ROOT` is defined in [bootstrap.php](../../../../bootstrap.php) as `dirname(__FILE__)`; no separate storage constant exists)*
     - Create the directory with `FileHelper::createFolder()`
     - Write `page.html` with `file_put_contents()`
     - Write `context.txt` containing: selector, episode URL, comic alias, exception message, full stack trace
     - Re-throw a plain `WebcomicsBuilderException` whose message reads:  
       `Selector [{selector}] not found. Debug files saved to: storage/debug/{dir-name}/`

4. **Add `storage/debug/` to `.gitignore`**
   - Append `/storage/debug/` to [.gitignore](../../../../.gitignore)

---

## Dependencies
- Existing `WebcomicsBuilderException` base class
- Existing `FileHelper` (already used in `BaseComicSource` for image path operations)
- `APP_ROOT` constant (defined in [bootstrap.php](../../../../bootstrap.php))

---

## Required Components
- **New** `assets/classes/DOM/DOMSelectorException.php`
- **Modified** `assets/classes/DOMHelper.php`
- **Modified** `assets/classes/Comics/Sources/BaseComicSource.php`
- **Modified** `.gitignore`
- **New directory** `storage/debug/` (created at runtime by the code; a `.gitkeep` is not needed since the directory is gitignored)

---

## Assumptions
- The storage root path is `APP_ROOT . '/storage'`. `APP_ROOT` is the global constant defined in [bootstrap.php](../../../../bootstrap.php) line 11 and is available everywhere in the application.
- `FileHelper::createFolder()` creates intermediate directories (it already does this in other locations in the codebase).
- The same `DOMSelectorException` mechanism does not need to be applied to `domSelectRequireFirstEX()` separately — it delegates to `domSelectRequireFirst()` and naturally benefits from the change.

---

## Constraints
- Must not change the AJAX JSON response structure — the enriched message already fits within the existing `{"state":"error","data":{"message":"…"}}` envelope.
- `DOMHelper` must remain unaware of filesystem paths.
- No new Composer dependencies.

---

## Out of Scope
- A UI panel to browse debug dumps from within the application.
- Automatic cleanup / rotation of old debug files.
- Logging selector failures that occur outside the episode-fetch flow (e.g., index parsing).
- Changing the error handling when `debug=1` is present in the request (that path already re-throws and shows a full PHP error page).

---

## Acceptance Criteria
- When episode fetching fails due to a missing selector, `storage/debug/` contains a subfolder with `page.html` and `context.txt`.
- `page.html` contains the actual HTML that was downloaded and searched.
- `context.txt` contains the failing selector, the episode URL, the comic alias, and the full PHP exception trace.
- The AJAX error message includes the path to the debug subfolder (e.g., `storage/debug/grrl-power--42--20260301-143022/`).
- Successful episode fetches are unaffected.
- The `storage/debug/` directory is excluded from version control.

---

## Testing Strategy
- **Unit test** (optional): modify or create a comic source fixture that calls `domSelectRequireFirst()` with a selector guaranteed to fail, then assert that the debug folder is created and both files are written.
- **Manual test**: trigger a fetch on a comic whose selector is intentionally broken; verify the AJAX error message contains the debug path and that the files exist and contain expected content.
- **Regression test**: trigger a successful episode fetch and confirm no debug files are written.

---

## Risks & Mitigations
| Risk | Mitigation |
|------|------------|
| **Large HTML files filling disk** | Noted as out of scope for now; can add a simple count-based or age-based cleanup as a follow-up |
| **Race condition on directory name if same episode fetched simultaneously** | The timestamp granularity (seconds) is sufficient for a single-user local tool; not a concern |
| **`DOMSelectorException` not caught if a comic source overrides `processEpisode()`** | The catch is placed in `fetchEpisode()` which all sources use; custom `processEpisode()` overrides still throw upward through `fetchEpisode()` |
