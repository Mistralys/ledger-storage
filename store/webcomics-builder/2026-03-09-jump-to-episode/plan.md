# Plan — Jump To Episode Feature

## Summary

Add a "Jump To" feature to the comic reader toolbar that opens a Bootstrap 5 modal where users can enter a numeric page number (1-based index) to navigate directly to that episode. This is necessary because episode IDs are often non-numeric alphanumerical slugs (e.g., `kill-six-billion-demons-chapter-1`), making it impossible to jump to a specific position by episode ID alone. The feature maps a simple 1-based numeric index onto the ordered list of fetched episodes that the reader already uses for navigation.

## Architectural Context

### Relevant Modules

- **`assets/classes/Page/ReaderPage.php`** — The reader page; its `renderView()` method builds the toolbar and renders episode images. Already loads `viewer.js` and `viewer.css`. Already has a commented-out `addToolbarHTML()` call that was the old jump-to feature (lines 224–238). Contains AJAX handlers (`ajax_*` methods) dispatched from `BasePage::handleAJAX()`.
- **`js/viewer.js`** — Client-side `Viewer` class; already has a remnant `JumpTo()` method (lines 112–120) that reads from `#field-jumper` and navigates via `baseURL + '&episode='+episodeNr`. Needs to be reworked.
- **`css/viewer.css`** — Viewer-specific styles; already has a `.jumper` class (line 17).
- **`assets/classes/UI/BasePage.php`** — Base page class; provides `addToolbarLink()`, `addToolbarButton()`, `addToolbarSeparator()`, and JS head/onload helpers. No `addToolbarHTML()` method exists (it was removed).
- **`assets/classes/UI/Toolbar/Items/ToolbarButton.php`** — Renders a toolbar anchor with an `onclick` JS statement. Ideal for the "Jump To" trigger.
- **`assets/classes/UI/Scaffold/Navigation.php`** — Renders the toolbar `<ul>` from `ToolbarItemInterface[]`. The modal HTML will **not** go inside the toolbar; it should be placed in the page body.
- **`assets/classes/Fetcher.php`** — `getFetched()` returns the ordered `Episode[]` array used for reader navigation. `getAll()` returns all episodes. The `load()` method assigns a 0-based `$index` to each episode.
- **`assets/classes/Episode.php`** — `getIndex()` returns the 0-based position in the full index (all episodes, including unfetched). `getViewURL()` builds the reader URL with `episodeID`.

### Key Patterns

- Toolbar items are PHP objects added via `addToolbarLink()` / `addToolbarButton()` in the page's `renderView()`.
- AJAX calls use `BasePage::handleAJAX()` dispatching to `ajax_{name}()` methods, responding with `sendAJAXSuccess()` / `sendAJAXError()`.
- The viewer page already has a `<script>` block at the bottom of `renderImages()` that instantiates `const viewer = new Viewer(...)` and exposes `baseURL`.
- Bootstrap 5.3 is loaded globally (via `BasePage::render()`), so modals are natively supported.
- Localization uses `t()` / `pt()` for translatable strings.

### Index Concepts

There are two index concepts that must be clearly distinguished:

1. **Full index** — `Fetcher::getAll()` returns all episodes from `index.json` (fetched + unfetched). Each episode's `Episode::getIndex()` is its 0-based position in this full list.
2. **Fetched-only index** — `Fetcher::getFetched()` returns only downloaded episodes. The reader toolbar ("First", "Last", "Previous", "Next") and `Builder::getNextBuilt()`/`getPreviousBuilt()` operate on this fetched-only list.

The "Jump To" feature should jump within the **fetched-only** list, because those are the only episodes that can actually be viewed. The page number shown to the user will be **1-based** (page 1 = first fetched episode, page N = last fetched episode).

## Approach / Architecture

### Server-side (PHP)

1. **Add a `resolveEpisodeByNumber` AJAX handler** in `ReaderPage` that takes a 1-based page number, looks up the corresponding fetched episode, and returns its `viewURL`.
2. **Emit the total fetched-episode count & current 1-based page number** as JS variables in the `renderImages()` script block, so the modal can display "Page X of N".
3. **Add a "Jump To" toolbar button** that triggers `viewer.ShowJumpToModal()`.
4. **Render a Bootstrap 5 modal HTML** in the `renderImages()` output (within the page body, outside the toolbar).

### Client-side (JavaScript)

1. **Rework `Viewer.JumpTo()`** to make an AJAX call to the server with the requested page number, then receive the episode's `viewURL` and redirect.
2. **Add `Viewer.ShowJumpToModal()`** to open the modal and focus the input field.
3. The modal form submits on Enter and calls `viewer.JumpTo()`.

### Why AJAX instead of client-side redirect?

The client does not have the full mapping of page-number → episode-ID. While we could emit a full JSON array of `[pageNumber → viewURL]` (which the old code seems to have attempted), this would bloat the page for comics with hundreds/thousands of episodes. An AJAX call is lightweight: the server resolves the page number to an episode and returns a URL; the client redirects.

## Rationale

- **AJAX resolution** avoids embedding a potentially large page-number-to-URL mapping in the page HTML.
- **Bootstrap 5 modal** is the natural choice since Bootstrap 5.3 is already loaded globally.
- **1-based page number on fetched-only list** matches user expectations (page 1 = first readable episode) and avoids confusion with unfetched episodes that can't be viewed.
- **Toolbar button** follows the existing pattern (same as "Save position", "Bookmark", flag toggles).
- **Reusing `Viewer.JumpTo()`** preserves the original method stub and its intent.

## Detailed Steps

### Step 1: Add AJAX handler `ajax_resolvePageNumber` in `ReaderPage.php`

Add a new AJAX method after the existing `ajax_useAsCover` method:

```php
protected function ajax_resolvePageNumber(): void
{
    $pageNumber = (int)$this->request->getParam('pageNumber');
    $episodes = $this->fetcher->getFetched();
    $total = count($episodes);

    if ($pageNumber < 1 || $pageNumber > $total) {
        $this->sendAJAXError(t('Invalid page number. Must be between 1 and %1$s.', $total));
    }

    $episode = $episodes[$pageNumber - 1];

    $this->sendAJAXSuccess(array(
        'viewURL' => $episode->getViewURL(),
        'episodeID' => $episode->getID(),
        'pageNumber' => $pageNumber,
        'totalPages' => $total
    ));
}
```

### Step 2: Add the "Jump To" toolbar button in `ReaderPage::renderView()`

In the toolbar section of `renderView()`, after the "First"/"Last" links and before the commented-out old jump-to code, add:

```php
$this->addToolbarButton(
    'viewer.ShowJumpToModal()',
    t('Jump to')
)->setIcon(UI::icon()->jumpTo()); // Or a suitable existing icon
```

Remove the old commented-out `addToolbarHTML(...)` block entirely.

Check what icons are available in the `Icon` class. If there is no suitable "jump to" icon, use an existing one like a search or hash icon, or add a new one.

### Step 3: Emit JS variables in `renderImages()` script block

In the `<script>` block inside `renderImages()`, add `currentPageNumber` and `totalPages` variables:

```php
// Compute 1-based page number among fetched episodes
$fetchedEpisodes = $this->fetcher->getFetched();
$currentPageNumber = array_search($episode, $fetchedEpisodes, true);
if ($currentPageNumber !== false) {
    $currentPageNumber += 1; // 1-based
} else {
    $currentPageNumber = 0; // fallback
}
```

Then in the `<script>` block:

```javascript
const totalPages = <?php echo count($fetchedEpisodes) ?>;
const currentPageNumber = <?php echo $currentPageNumber ?>;
```

### Step 4: Render the Bootstrap 5 modal HTML in `renderImages()`

After the `<script>` block (or before the `ViewerImages` div), output a standard Bootstrap 5 modal:

```php
<div class="modal fade" id="jumpToModal" tabindex="-1" aria-labelledby="jumpToModalLabel" aria-hidden="true">
    <div class="modal-dialog modal-sm modal-dialog-centered">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title" id="jumpToModalLabel"><?php pt('Jump to page') ?></h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="<?php pt('Close') ?>"></button>
            </div>
            <div class="modal-body">
                <form id="jumpToForm" onsubmit="event.preventDefault(); viewer.JumpTo();">
                    <div class="mb-3">
                        <label for="field-jumper" class="form-label">
                            <?php pt('Page number (1–%1$s)', count($fetchedEpisodes)) ?>
                        </label>
                        <input 
                            type="number" 
                            class="form-control" 
                            id="field-jumper" 
                            min="1"
                            max="<?php echo count($fetchedEpisodes) ?>"
                            value="<?php echo $currentPageNumber ?>"
                            placeholder="<?php pt('Page number') ?>"
                            required
                        >
                    </div>
                </form>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal"><?php pt('Cancel') ?></button>
                <button type="button" class="btn btn-primary" onclick="viewer.JumpTo()"><?php pt('Go') ?></button>
            </div>
        </div>
    </div>
</div>
```

### Step 5: Rework `Viewer.JumpTo()` in `viewer.js`

Replace the existing `JumpTo()` method:

```javascript
JumpTo()
{
    const pageNumber = parseInt($('#field-jumper').val(), 10);
    if (isNaN(pageNumber) || pageNumber < 1) {
        return;
    }

    const viewer = this;
    
    // Disable the go button to prevent double-clicks
    $('#jumpToModal .btn-primary').prop('disabled', true);

    $.ajax({
        'dataType': "json",
        'url': this.ajaxBaseURL + '&ajax=resolvePageNumber&pageNumber=' + pageNumber,
        'data': null,
        'success'(data) {
            if (data.state === 'success') {
                document.location = data.data.viewURL;
            } else {
                alert(data.data.message || 'Invalid page number');
                $('#jumpToModal .btn-primary').prop('disabled', false);
            }
        },
        'error'(jqxhr, textStatus, error) {
            alert('Error: ' + textStatus + ', ' + error);
            $('#jumpToModal .btn-primary').prop('disabled', false);
        }
    });
}
```

### Step 6: Add `Viewer.ShowJumpToModal()` in `viewer.js`

Add a new method to the `Viewer` class:

```javascript
ShowJumpToModal()
{
    const modal = new bootstrap.Modal(document.getElementById('jumpToModal'));
    modal.show();

    // Focus the input field after the modal is shown
    document.getElementById('jumpToModal').addEventListener('shown.bs.modal', function () {
        const field = document.getElementById('field-jumper');
        field.focus();
        field.select();
    }, { once: true });
}
```

### Step 7: Check/add an appropriate icon for the toolbar button

Inspect the `Icon` class (`assets/classes/UI/Icon.php`) to find or add a suitable icon method. If no jump/search icon exists, add one:

```php
public function jumpTo() : self
{
    return $this->setClasses('fa-solid fa-arrow-right-to-bracket');
    // Or fa-hashtag, fa-forward, etc.
}
```

### Step 8: Remove old commented-out toolbar code

Delete the commented-out `addToolbarHTML(...)` block (lines 224–238 in `ReaderPage.php`) since the new implementation replaces it.

### Step 9: Update the "Episode X of N" display in `renderImages()`

Currently, the header in `renderImages()` shows:

```php
<p><?php pt('Episode %1$s', '#'.$episode->getID()) ?></p>
<p><?php pt('%1$s total episodes found.', $totalEpisodes); ?></p>
```

Optionally enhance this to also show the 1-based page number:

```php
<p><?php pt('Episode %1$s', '#'.$episode->getID()) ?> — <?php pt('Page %1$s of %2$s', $currentPageNumber, count($fetchedEpisodes)) ?></p>
```

This gives the user a reference for what number to enter in the "Jump To" modal.

## Dependencies

- Bootstrap 5.3 modal JS component (already loaded globally).
- jQuery (already loaded globally).
- `Fetcher::getFetched()` providing the ordered fetched-episode list.
- `Episode::getViewURL()` to build redirect URLs.

## Required Components

### Modified Files

- `assets/classes/Page/ReaderPage.php` — Add AJAX handler, toolbar button, modal HTML, JS variables, page number display.
- `js/viewer.js` — Rework `JumpTo()`, add `ShowJumpToModal()`.
- `assets/classes/UI/Icon.php` — Possibly add a `jumpTo()` icon method (verify existing icons first).

### No New Files

All changes fit within existing files. No new classes, no new JS files, no new CSS files needed.

## Assumptions

- The "page number" concept only applies to **fetched** episodes, since unfetched episodes cannot be viewed in the reader.
- Page numbers are 1-based for user-friendliness.
- The fetched-episode ordering is stable (defined by `index.json` order, filtered to fetched-only).
- A Bootstrap 5 modal is the appropriate UI pattern for this interaction (consistent with the Bootstrap 5.3 stack).

## Constraints

- No database (JSON-only persistence) — respected; this feature has no persistence needs.
- No new PHP frameworks/libraries — respected; uses existing Bootstrap 5 modal.
- PHP 8.4+ features allowed.
- Localization: all user-visible strings must use `t()` / `pt()`.

## Out of Scope

- Keyboard shortcut to open the "Jump To" modal (could be added later).
- Paginated episode listing inside the modal.
- Searching episodes by title/name in the modal.
- Persistence of the last-used page number across sessions.
- Changes to the offline viewer (`assets/offline/`).
- Updating `index.json` structure — the numeric index is derived at runtime.

## Acceptance Criteria

1. A "Jump To" toolbar button appears in the reader toolbar between "Last" and the next separator.
2. Clicking "Jump To" opens a centered Bootstrap 5 modal.
3. The modal displays a form with:
   - A label showing the valid page range (e.g., "Page number (1–350)").
   - A number input pre-filled with the current page number.
   - "Cancel" (closes modal) and "Go" (navigates) buttons.
4. Pressing Enter in the input submits the form and navigates.
5. Entering a valid page number and clicking "Go" redirects to the correct episode.
6. Entering an invalid page number (0, negative, or > total) shows an error message.
7. The episode header in the reader shows "Page X of N" alongside the episode ID.
8. All user-visible strings are wrapped in `t()` / `pt()` for localization.

## Testing Strategy

- **Manual testing:** Open the reader for a comic with many fetched episodes; open the Jump To modal; enter page 1 (→ first episode), the last page number (→ last episode), and a middle page number (→ correct episode). Verify navigation is correct.
- **Edge cases:** Enter 0, -1, a number larger than total, non-numeric input. Verify error handling.
- **Cross-comic testing:** Test with comics that have numeric episode IDs (e.g., `gunnerkrigg-court`) and non-numeric IDs (e.g., `kill-six-billion-demons`) to ensure the page-number mapping works independently of episode ID format.
- **PHPUnit (recommended):** Add a test that creates a mock set of episodes and verifies that `ajax_resolvePageNumber` returns the correct episode for given page numbers. This would require making the method testable (e.g., extractable service).

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Fetched-episode ordering differs between `Fetcher::getFetched()` calls** | The order is deterministic (derived from `index.json` order, filtered by fetch state). No risk unless the index changes between requests. |
| **Large number of fetched episodes could cause slow lookups** | The lookup is `O(1)` array indexing (`$episodes[$pageNumber - 1]`). No performance concern. |
| **Modal conflicts with existing page JS or toolbar** | Bootstrap 5 modal is self-contained. The modal div is placed in the page body (outside the toolbar nav), avoiding DOM nesting issues. |
| **Icon class may not have a suitable icon** | Fallback: use `fa-hashtag` or `fa-forward` icon. The Icon class just wraps Font Awesome classes. |
| **AJAX call fails (e.g., session expired, network issue)** | Standard jQuery error handler shows an alert and re-enables the button. |
