# Plan

## Summary

Move comic list filter settings (status, sort field, sort direction) from the ephemeral PHP session to persistent per-user JSON storage in `storage/users.json`. The search text remains session-only (ephemeral). When a user selects their profile, their last-used filter settings are restored. When they change filters, the settings are saved to their user record alongside existing per-user data (positions, bookmarks). When no user is selected, filters fall back to defaults.

## Architectural Context

### Current filter system

- **`ComicsFilter`** ([assets/classes/Comics/Filter/ComicsFilter.php](assets/classes/Comics/Filter/ComicsFilter.php)) — Base filter class holding four settings: `status`, `orderBy`, `orderDir`, `searchText`. Provides `getMatches()` to return filtered/sorted comics.
- **`SessionComicsFilter`** ([assets/classes/Comics/Filter/SessionComicsFilter.php](assets/classes/Comics/Filter/SessionComicsFilter.php)) — Extends `ComicsFilter`. On construction, reads filter values from `Session::getArray('comicsFilter')`. `save()` writes back to the session.
- **`DetailedListPage`** ([assets/classes/Page/DetailedListPage.php](assets/classes/Page/DetailedListPage.php)) — Instantiates `SessionComicsFilter` in `_init()`. When the filter form is submitted (`$_REQUEST['filter']`), it calls `importValues()` from `$_REQUEST`, then `save()`, then redirects.
- **`CardListPage`** ([assets/classes/Page/CardListPage.php](assets/classes/Page/CardListPage.php)) — Extends `DetailedListPage`; inherits filter handling entirely.

### Current user system

- **`User`** ([assets/classes/Users/User.php](assets/classes/Users/User.php)) — Stores `positions`, `bookmarks`, `imageBookmarks`, and `properties` (generic key-value bag). Has `serialize()`/`unserialize()` for JSON persistence.
- **`UserCollection`** ([assets/classes/Users/UserCollection.php](assets/classes/Users/UserCollection.php)) — Manages `storage/users.json`. Calls `User::serialize()` to persist all users.
- **`Session`** ([assets/classes/Session.php](assets/classes/Session.php)) — Wraps `$_SESSION` with namespaced keys. Stores the selected user ID and ephemeral filter data.

### Key observation

The session is the **wrong persistence layer** for per-user data that should survive session expiry. Reading positions and bookmarks are already persisted in `users.json` — filter settings should follow the same pattern.

## Approach / Architecture

### New storage key on `User`

Add a `filterSettings` top-level key to the user JSON record, parallel to `positions` and `bookmarks`:

```json
{
    "id": "652d26ba5d95e276d67e23f33289515c",
    "name": "Argh",
    "admin": true,
    "positions": { ... },
    "bookmarks": [...],
    "imageBookmarks": { ... },
    "properties": [],
    "filterSettings": {
        "status": "active",
        "orderBy": "label",
        "orderDir": "asc"
    }
}
```

### Modified `SessionComicsFilter`

The `SessionComicsFilter` class will be updated to load from and save to the `User` record instead of (or in addition to) the raw session:

1. **Constructor**: If a user is selected (`Session::getUser()`), load filter values from `User::getFilterSettings()`. Otherwise, fall back to session-only defaults.
2. **`save()`**: If a user is selected, persist to `User::setFilterSettings()` → `User::save()`. The session copy is kept for performance (avoids JSON re-read on every page load).

### No new class needed

This can be achieved by modifying `SessionComicsFilter` and `User` directly without introducing a new class. The `SessionComicsFilter` name remains accurate — it is still the session-aware variant of filters, just backed by persistent storage now.

## Rationale

- **Consistency**: Positions and bookmarks are already per-user persistent data in `users.json`. Filter preferences are conceptually the same kind of data.
- **Minimal change**: Modifying `SessionComicsFilter` and `User` avoids adding new classes. The existing `DetailedListPage` code does not need to change since it already uses `SessionComicsFilter`.
- **Session cache**: Keeping the session as a cache layer avoids reading/parsing `users.json` on every page load. The session is populated from the user record on login/selection; subsequent requests use the session. Saving writes to both.
- **Graceful degradation**: If no user is selected, the filter falls back to defaults (same as current behavior with an empty session).

## Detailed Steps

### Step 1: Add `filterSettings` to `User`

In `assets/classes/Users/User.php`:

1. Add a new constant `KEY_FILTER_SETTINGS = 'filterSettings'`.
2. Add a private property `private array $filterSettings = []`.
3. Accept `$filterSettings` in the constructor and store it.
4. Add `getFilterSettings(): ArrayDataCollection` — returns the filter settings as an `ArrayDataCollection`.
5. Add `setFilterSettings(ArrayDataCollection $settings): self` — update the filter settings and call `save()`.
6. Update `serialize()` to include `filterSettings`.
7. Update `unserialize()` to load `filterSettings`.

### Step 2: Modify `SessionComicsFilter` to persist via `User`

In `assets/classes/Comics/Filter/SessionComicsFilter.php`:

1. **Constructor**: Check `Session::getUser()`. If a user is available, load structural filters (status, orderBy, orderDir) from `$user->getFilterSettings()`. Search text continues to be loaded from the session only. If the user has no saved filters (empty array), fall back to defaults. Also write to session for caching.
2. **`save()`**: Write all values to the session (as before). Additionally, if a user is selected, write only the structural filters (excluding search text) to `$user->setFilterSettings()`.

### Step 3: Populate session on user selection

In `Session::selectUser()` or in the `DetailedListPage::_init()` flow — when a user is selected, their filter settings should be loaded into the session. Since `SessionComicsFilter`'s constructor already does this, no extra work is needed: the first page load after user selection will construct a `SessionComicsFilter`, which reads from user storage and populates the session.

### Step 4: Handle user switching

When a different user is selected, the old session filter data must be replaced. Since `Session::selectUser()` sets a new user ID, the next `SessionComicsFilter` construction will detect the new user and load their settings, overwriting the session cache. No special code is needed beyond the constructor logic.

### Step 5: Update manifest documents

- Update `storage.md` — Document the new `filterSettings` key in the `users.json` schema.
- Update `api-surface.md` — Document `User::getFilterSettings()`, `User::setFilterSettings()`, and the new constant.

## Dependencies

- No new packages or libraries required.
- Relies on existing `ArrayDataCollection` from `mistralys/application-utils`.
- Relies on existing `SessionComicsFilter` / `User` / `Session` classes.

## Required Components

- **Modified**: `assets/classes/Users/User.php` — New `filterSettings` property, getter, setter, serialization.
- **Modified**: `assets/classes/Comics/Filter/SessionComicsFilter.php` — Load from user, save to user.
- **Modified**: `docs/agents/project-manifest/storage.md` — Document new JSON key.
- **Modified**: `docs/agents/project-manifest/api-surface.md` — Document new User methods.

## Assumptions

- The `searchText` filter is **excluded** from persistence. It remains session-only. Users do not expect a stale search term to be remembered across sessions; structural filters (status, sort) are the ones that represent a lasting preference.
- `User::save()` (which calls `UserCollection::save()`) is acceptably fast. The user JSON file is small and writes are infrequent (only on filter change).
- No migration step is needed. Users without a `filterSettings` key will simply get default filter values (the `unserialize` method treats the missing key as an empty array).

## Constraints

- **JSON-only persistence** — no database (per `constraints.md`).
- **PHP 8.4+ features** are available (typed properties, typed class constants).
- **Backward compatibility** — Existing `users.json` files without `filterSettings` must work without modification.

## Out of Scope

- Per-list-type filter settings (separate filters for CardList vs DetailedList). Both lists share the same filter set today and will continue to do so.
- Filter presets / saved filter profiles.
- Clearing/resetting filter settings to defaults from the UI (can be a follow-up).
- Migrating any existing session filter data to user records on upgrade. The filters will start fresh from defaults on the user's first visit after deployment.

## Acceptance Criteria

- When user A sets filters (e.g., status = "active", sort = "label desc") and navigates away, the structural filters are persisted.
- When user A returns (even after session expiry), the same structural filter settings are restored. The search text field is empty (session-only, not persisted).
- When user B selects their profile, they see their own filter settings, not user A's.
- A new user with no saved filters sees the default filter settings (no status filter, label ascending, no search text).
- Existing `users.json` files without the `filterSettings` key continue to work without errors.
- The `filterSettings` key appears in the user's JSON record after they change a filter.

## Testing Strategy

- **Unit test**: Create a test for `User::getFilterSettings()` / `User::setFilterSettings()` / `serialize()` round-trip to confirm the filter settings are correctly persisted and loaded.
- **Unit test**: Create a test for `SessionComicsFilter` that verifies filter values are loaded from a mock user's `filterSettings`.
- **Manual test**: Verify through the UI that filter settings survive session expiry (clear cookies, revisit, select same user).
- **Backward compatibility test**: Load a `users.json` fixture without `filterSettings` and verify no errors occur.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Disk I/O on every filter save** | `UserCollection::save()` writes the full `users.json` file. This is the same codepath used for bookmark saves today and has proven acceptable for a local tool with a small user list. |
| **Stale session cache** | If `users.json` is edited manually while a session is active, the session copy may diverge. This is an existing limitation (also affects positions/bookmarks) and is acceptable for local-only use. |
| **Search text not persisted** | Users who expect search text to survive sessions will find it cleared. This is intentional — a stale search term is more confusing than helpful. The structural filters (status, sort) represent lasting preferences; search is transient by nature. |
| **Missing `filterSettings` key in old data** | `ArrayDataCollection::create()` on an empty array returns an object that returns empty strings for all `getString()` calls, which maps to defaults. No migration needed. |
