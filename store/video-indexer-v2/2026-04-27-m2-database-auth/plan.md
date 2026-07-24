# Plan — M2 Database & Authentication

## Summary

Stand up the database and authentication layer of Video Indexer MKII on top of the M1
shell. M2 introduces MariaDB connectivity (MySqlConnector + Dapper), a per-launch
**DatabaseConnector → Logon → (Password Setup) → Ready** flow rendered as content states
of the existing main window, a saved-connection list persisted in `appsettings.json`, a
schema-revision check pinned to revision **35**, and a SHA-256 / fixed-salt application
password stored in `spdb_config`. Done means: launching the app from a fresh clone
against a live MariaDB shows the connector overlay; selecting / entering a connection
opens the database, validates `db_revision = 35`, prompts for password creation if no
hash row exists, otherwise prompts for the existing password, and only then reveals the
empty M1 main window content. M2 ships **no** library, indexing, or movie features.

## Architectural Context

The repository already contains the M1 shell at
[src/](../../../../src/):

- **`VideoIndexer.Core`** — pure abstractions, options records, enums, events.
  Existing files: [`Abstractions/IAppPaths.cs`](../../../../src/VideoIndexer.Core/Abstractions/IAppPaths.cs),
  [`Abstractions/ISettingsService.cs`](../../../../src/VideoIndexer.Core/Abstractions/ISettingsService.cs),
  [`Abstractions/IThemeService.cs`](../../../../src/VideoIndexer.Core/Abstractions/IThemeService.cs),
  [`Options/AppOptions.cs`](../../../../src/VideoIndexer.Core/Options/AppOptions.cs)
  and four sub-option records (`Logging`, `Appearance`, `Window`, `ExternalTools`),
  [`Enums/`](../../../../src/VideoIndexer.Core/Enums/) (`ThemeMode`),
  [`Events/`](../../../../src/VideoIndexer.Core/Events/) (`ThemeModeChangedEventArgs`).
- **`VideoIndexer.Infrastructure`** — file-system, settings, theming, logging.
  Existing files: [`AppPaths.cs`](../../../../src/VideoIndexer.Infrastructure/AppPaths.cs),
  [`Settings/JsonSettingsService.cs`](../../../../src/VideoIndexer.Infrastructure/Settings/JsonSettingsService.cs),
  [`Theme/ThemeService.cs`](../../../../src/VideoIndexer.Infrastructure/Theme/ThemeService.cs),
  [`Logging/LoggingSetup.cs`](../../../../src/VideoIndexer.Infrastructure/Logging/LoggingSetup.cs).
- **`VideoIndexer.App`** — Avalonia entry point + DI bridge. Existing files:
  [`Program.cs`](../../../../src/VideoIndexer.App/Program.cs),
  [`App.axaml.cs`](../../../../src/VideoIndexer.App/App.axaml.cs),
  [`ViewLocator.cs`](../../../../src/VideoIndexer.App/ViewLocator.cs),
  [`Views/MainWindow.axaml`](../../../../src/VideoIndexer.App/Views/MainWindow.axaml),
  [`ViewModels/MainWindowViewModel.cs`](../../../../src/VideoIndexer.App/ViewModels/MainWindowViewModel.cs),
  [`Assets/appsettings.json`](../../../../src/VideoIndexer.App/Assets/appsettings.json).
- **Tests** — [`tests/VideoIndexer.Tests/`](../../../../tests/VideoIndexer.Tests/) and
  [`tests/VideoIndexer.Infrastructure.Tests/`](../../../../tests/VideoIndexer.Infrastructure.Tests/),
  41 passing tests on xUnit + FluentAssertions.

Authoritative inputs for M2:

- [docs/projects/rebuild/management-areas/system-management-specification.md](../../../projects/rebuild/management-areas/system-management-specification.md)
  §1 (Application Launch & Authentication) — flow, single-password rule, fixed-salt
  storage, schema-revision gate.
- [docs/projects/rebuild/rebuild.md](../../../projects/rebuild/rebuild.md) — confirms
  **MySqlConnector + Dapper** as the data access stack and full compatibility with the
  legacy MariaDB schema.
- [docs/projects/rebuild/milestones/roadmap.md](../../../projects/rebuild/milestones/roadmap.md)
  — M2 scope: "MySqlConnector + Dapper bootstrap, schema-revision check,
  DatabaseConnector + Logon overlays, password setup". Roadmap rule: *the app must still
  launch cleanly with all prior milestones intact*.

Legacy reference (read-only, used to confirm SQL shapes — never imported):

- [`spdb-indexer/SPDB Indexer/sql/structure.sql`](../../../../../spdb-indexer/SPDB%20Indexer/sql/structure.sql)
  — `spdb_config (config_name VARCHAR(200), config_value TEXT)`.
- [`spdb-indexer/SPDB Indexer/Classes/Program.cs`](../../../../../spdb-indexer/SPDB%20Indexer/Classes/Program.cs)
  lines 270–340 — `GetDBConfig` / `SetDBConfig` / `DBRevision` patterns.
- The legacy `Logon.cs` uses a hardcoded crypto key (`Sport007`) rather than a stored
  hash; the spec supersedes this and M2 implements the spec, not the legacy control.

### Conventions inherited from M1 (MUST be preserved)

- Sealed records with `init`-only properties for option/DTO types.
- Full nullable annotation, `TreatWarningsAsErrors=true`, `<WarningLevel>9999</WarningLevel>`.
- Atomic file writes via `.tmp` + `File.Move(overwrite:true)`.
- `#if DEBUG` for any debug-only UI scaffolding; binary verification before shipping.
- `xUnit` + `FluentAssertions`; naming `Subject_Scenario_ExpectedBehavior`; in-memory
  fakes; `IDisposable` temp-directory fixtures.
- Dependency direction: Core → (nothing); Infrastructure → Core; App → Core +
  Infrastructure; Tests → Core + Infrastructure.

## Approach / Architecture

### New solution shape (additions only)

```
src/
├── VideoIndexer.Core/
│   ├── Abstractions/
│   │   ├── IDbConnectionFactory.cs            (NEW)
│   │   ├── ISpdbConfigRepository.cs           (NEW)
│   │   ├── IDatabaseBootstrapper.cs           (NEW)
│   │   ├── IPasswordService.cs                (NEW)
│   │   ├── IPasswordHasher.cs                 (NEW)
│   │   └── IDatabaseConnectionStore.cs        (NEW)
│   ├── Enums/
│   │   ├── DatabaseBootstrapStatus.cs         (NEW: Ok | RevisionTooOld | RevisionTooNew | ConfigTableMissing)
│   │   └── ShellState.cs                      (NEW: Connecting | LoggingOn | SettingPassword | Ready | Error)
│   ├── Events/
│   │   └── ShellStateChangedEventArgs.cs      (NEW)
│   └── Options/
│       ├── DatabaseOptions.cs                 (NEW: contains saved connections list + active id)
│       ├── DatabaseConnectionOptions.cs       (NEW: per-connection record)
│       └── AppOptions.cs                      (MODIFIED: add Database property)
├── VideoIndexer.Infrastructure/
│   ├── Database/
│   │   ├── MySqlConnectionFactory.cs          (NEW)
│   │   ├── DatabaseBootstrapper.cs            (NEW: revision check)
│   │   └── SpdbConfigRepository.cs            (NEW: Dapper-backed)
│   ├── Auth/
│   │   ├── Sha256FixedSaltPasswordHasher.cs   (NEW)
│   │   └── PasswordService.cs                 (NEW: orchestrates hash / verify / store)
│   └── Settings/
│       └── JsonDatabaseConnectionStore.cs     (NEW: persistence wrapper around ISettingsService)
└── VideoIndexer.App/
    ├── ViewModels/
    │   ├── ShellViewModel.cs                  (NEW: state machine)
    │   ├── DatabaseConnectorViewModel.cs      (NEW)
    │   ├── LogonViewModel.cs                  (NEW)
    │   ├── PasswordSetupViewModel.cs          (NEW)
    │   ├── MainContentViewModel.cs            (NEW: empty placeholder, occupies "Ready")
    │   └── MainWindowViewModel.cs             (MODIFIED: hosts ShellViewModel)
    ├── Views/
    │   ├── DatabaseConnectorView.axaml(.cs)   (NEW)
    │   ├── LogonView.axaml(.cs)               (NEW)
    │   ├── PasswordSetupView.axaml(.cs)       (NEW)
    │   ├── MainContentView.axaml(.cs)         (NEW: empty placeholder)
    │   └── MainWindow.axaml(.cs)              (MODIFIED: ContentControl bound to ShellViewModel.Current)
    └── Assets/appsettings.json                (MODIFIED: add Database section)

tests/
├── VideoIndexer.Tests/
│   ├── DatabaseOptionsTests.cs                (NEW)
│   ├── ShellViewModelTests.cs                 (NEW)
│   └── PasswordHasherTests.cs                 (NEW)
└── VideoIndexer.Infrastructure.Tests/
    ├── Database/
    │   ├── DatabaseBootstrapperTests.cs       (NEW: against an in-memory fake IDbConnectionFactory)
    │   └── SpdbConfigRepositoryTests.cs       (NEW: live integration, see Testing Strategy)
    ├── Auth/
    │   └── PasswordServiceTests.cs            (NEW)
    └── Settings/
        └── JsonDatabaseConnectionStoreTests.cs (NEW)
```

### Composition root changes

`Program.cs` gains the following DI registrations (all singletons except where noted):

- `IDbConnectionFactory` → `MySqlConnectionFactory` (resolves the **active** connection
  from `IDatabaseConnectionStore`; throws `InvalidOperationException` if none active).
- `ISpdbConfigRepository` → `SpdbConfigRepository`.
- `IDatabaseBootstrapper` → `DatabaseBootstrapper`.
- `IPasswordHasher` → `Sha256FixedSaltPasswordHasher`.
- `IPasswordService` → `PasswordService`.
- `IDatabaseConnectionStore` → `JsonDatabaseConnectionStore`.
- `ShellViewModel`, `DatabaseConnectorViewModel`, `LogonViewModel`,
  `PasswordSetupViewModel`, `MainContentViewModel` as transient.

The Generic Host is unchanged. M1's `App.axaml.cs` already wires `MainWindow` from DI;
M2 only changes `MainWindow`'s content binding.

### Shell state machine

`ShellViewModel` owns a single `ShellState Current` observable property and a
`CurrentViewModel` projection. Transitions:

```
Connecting ──► (connection ok, config table missing) ──► Error("schema not initialised")
Connecting ──► (connection ok, db_revision != 35)    ──► Error("revision mismatch: …")
Connecting ──► (connection ok, no password row)      ──► SettingPassword
Connecting ──► (connection ok, password row exists)  ──► LoggingOn
LoggingOn  ──► (password verified)                   ──► Ready
SettingPassword ──► (password stored)                ──► Ready
*          ──► (Cancel / Disconnect)                 ──► Connecting
```

`MainWindow.axaml` hosts a single `<ContentControl Content="{Binding Current}" />`; the
`ViewLocator` already knows how to resolve a view from a view-model type. No modal
windows; no Avalonia overlay layer.

### Saved-connections model

```jsonc
"Database": {
  "ActiveConnectionId": null,                 // GUID of the entry to auto-select on launch, or null = show picker
  "Connections": [
    {
      "Id": "5b6a…",                          // GUID
      "DisplayName": "Local MariaDB",         // user-supplied label
      "Host": "127.0.0.1",
      "Port": 3306,
      "Database": "spdb",
      "Username": "spdb",
      "Password": "…",                        // plaintext, see security note below
      "UseSsl": false
    }
  ]
}
```

- **Persistence model.** Connections are first-class data on `AppOptions.Database`
  (extension of M1's settings tree), persisted via the existing `ISettingsService`.
  A new `IDatabaseConnectionStore` wraps `ISettingsService` to expose CRUD that is
  semantically focused on connections (`Add`, `Update`, `Remove`, `SetActive`, `List`).
- **Connection string.** Built by `MySqlConnectionFactory` from a
  `DatabaseConnectionOptions` record using `MySqlConnectionStringBuilder`; never
  stored in the settings file as a single concatenated string.
- **Trust boundary.** Passwords are persisted in plaintext (matching the legacy app and
  the rebuild's intent to keep a single-user desktop trust model). The README and
  `m2-database-auth.md` will document this explicitly. Process-launch arguments must
  never contain the password (currently true; included as a non-regression check).

### Schema-revision check

`SpdbConfigRepository.GetValueAsync(string name, CancellationToken)` reads a row from
`spdb_config`. `DatabaseBootstrapper.CheckAsync` does:

1. Open a connection. Surface `MySqlException` as `DatabaseBootstrapStatus`-tagged
   results — never a thrown exception across the abstraction boundary.
2. Query `INFORMATION_SCHEMA.TABLES` for `spdb_config`. If absent →
   `ConfigTableMissing`.
3. `repo.GetValueAsync("db_revision")`. Parse as `int`.
   - `< 35` → `RevisionTooOld` (carries actual + expected for the error UI).
   - `> 35` → `RevisionTooNew`.
   - `== 35` → `Ok`.
4. The expected revision is a `public const int ExpectedRevision = 35;` on
   `DatabaseBootstrapper`. Future milestones add new revisions by bumping this constant
   and shipping a migration handler — out of scope for M2.

### Password authentication

- **Hash format.** SHA-256 over UTF-8 `password + "spdb_42"`, hex-encoded lowercase
  (64 chars). Stored as the `config_value` of the `spdb_config` row with
  `config_name = 'app_password_hash'`.
- **`IPasswordHasher.Hash(string password) → string`** — pure, no I/O, fully unit-testable.
- **`IPasswordService.HasPasswordAsync(ct)`** — true iff the row exists.
  `VerifyAsync(string)` — hashes input and compares with constant-time
  `CryptographicOperations.FixedTimeEquals` over the byte arrays of the two hashes.
  `SetPasswordAsync(string)` — writes the hash row via the repository.
- **Confirmation field.** `PasswordSetupViewModel` enforces a non-empty password and a
  matching confirmation field before enabling the *Set Password* button.

### Localisation note

The spec lists English / German / French. M1 deferred localisation runtime; M2 keeps
all UI strings as **literals in XAML** for now. A `Strings.resx` rollout is its own
future WP and is **out of scope** for M2.

## Rationale

- **Saved-connections list** matches the user's choice and the legacy `DBConnections`
  affordance. It costs one extra view-model and minor JSON shape work versus a single
  hard-coded set.
- **State-machine shell over modal windows** keeps a single Avalonia `Window`
  lifetime, avoids modal-dialog plumbing (unparented dialogs are awkward to centre on
  the main window before it shows), is trivially unit-testable, and matches Avalonia
  + MVVM idioms.
- **Dapper micro-repositories** match `rebuild.md`'s tech stack rationale (plain SQL,
  no schema warping). One repository per logical table avoids the legacy `DBHelper`
  god-class. Only `SpdbConfigRepository` ships in M2; M3+ add their own.
- **SHA-256 with the spec's fixed salt** uses a modern primitive without inventing a
  format outside the spec. The fixed salt is acknowledged as access-gating, not
  password storage at scale; this is documented in the security audit output and the
  README.
- **Schema revision pinned to 35** per user direction. The constant is a single source
  of truth; future revisions require bumping it (and adding migration code in the WP
  that introduces them).
- **Plaintext DB password persistence** matches the legacy and the user's choice. The
  trust boundary is single-user filesystem ACLs on `%APPDATA%`. Cross-platform
  encryption is deferred to avoid forking behaviour between Windows DPAPI and a
  bespoke Linux/macOS path before M10 ships.
- **Empty `MainContentView`** is a deliberate non-feature: it proves the shell hands
  off cleanly to the post-auth surface that M3 will replace.

## Detailed Steps

1. **Extend the options tree.**
   - Add `DatabaseConnectionOptions` (`Id`, `DisplayName`, `Host`, `Port`, `Database`,
     `Username`, `Password`, `UseSsl`) and `DatabaseOptions`
     (`ActiveConnectionId`, `Connections` list) records to `VideoIndexer.Core.Options`.
   - Add `Database` property to `AppOptions`. Update bundled defaults
     (`Assets/appsettings.json`) with an empty `Database` section.

2. **Add core abstractions.**
   - `IDbConnectionFactory.OpenAsync(CancellationToken)` returning `IDbConnection`
     (avoids leaking `MySqlConnection` through Core).
   - `ISpdbConfigRepository` with `GetValueAsync(name)`, `SetValueAsync(name, value)`,
     `ExistsAsync(name)`, `IsSchemaPresentAsync()`.
   - `IDatabaseBootstrapper` returning a result record
     `(DatabaseBootstrapStatus Status, int? ActualRevision)`.
   - `IPasswordHasher` (pure), `IPasswordService` (orchestrates hashing + storage),
     `IDatabaseConnectionStore` (CRUD on the saved list).
   - `ShellState` enum and `ShellStateChangedEventArgs`.

3. **Add infrastructure implementations.**
   - `MySqlConnectionFactory` reads the active connection via
     `IDatabaseConnectionStore`, builds the connection string with
     `MySqlConnectionStringBuilder`, opens asynchronously.
   - `SpdbConfigRepository` uses Dapper (`ExecuteAsync`, `QuerySingleOrDefaultAsync`).
     Idempotent `SetValueAsync` deletes-then-inserts inside a single transaction
     (mirrors the legacy idempotent semantics).
   - `DatabaseBootstrapper.ExpectedRevision = 35`. Returns the result record above.
   - `Sha256FixedSaltPasswordHasher` — single static SHA-256 + constant
     `"spdb_42"`. No I/O.
   - `PasswordService` composes hasher + repository.
   - `JsonDatabaseConnectionStore` mutates `AppOptions.Database` and calls
     `ISettingsService.SaveAsync`.

4. **Register all new services in `Program.cs`.** No host-builder structural changes.

5. **Build the shell view-models.**
   - `ShellViewModel` exposes `Current` (`ShellState`), `CurrentViewModel` (projection),
     `TransitionTo(ShellState)` and the high-level orchestration methods
     `ConnectAsync(connectionId)`, `LogonAsync(password)`, `SetPasswordAsync(new, confirm)`,
     `Disconnect()`.
   - `DatabaseConnectorViewModel` — backed by `IDatabaseConnectionStore`. Exposes
     observable `Connections` collection, `Selected`, `Add`/`Edit`/`Remove`/`Connect`
     `[RelayCommand]`s. Calls into `ShellViewModel.ConnectAsync`.
   - `LogonViewModel` — single `Password` field + `Submit` command; on submit calls
     `ShellViewModel.LogonAsync`.
   - `PasswordSetupViewModel` — `Password` + `Confirm` + `Submit` command; validation
     blocks the command unless both fields are non-empty and equal.
   - `MainContentViewModel` — empty.
   - `MainWindowViewModel` — gains a `ShellViewModel Shell` injected through the
     constructor; the existing Ctrl+T debug theme cycle is preserved by routing via
     `Shell` only when in `Ready`.

6. **Build the views.**
   - `DatabaseConnectorView`, `LogonView`, `PasswordSetupView`, `MainContentView` —
     plain Avalonia XAML; bindings only, no code-behind logic except DI ctors.
   - `MainWindow.axaml` becomes a single `<ContentControl Content="{Binding Shell.CurrentViewModel}" />`
     wrapped in a top bar (placeholder app title only). Window chrome unchanged.
   - `ViewLocator` already resolves these by naming convention.

7. **Wire startup.**
   - When the host starts and the App opens `MainWindow`, `ShellViewModel` initialises
     to `Connecting`. If `DatabaseOptions.ActiveConnectionId` is non-null and resolves
     to a saved entry, `ConnectAsync(activeId)` is called automatically; otherwise the
     connector view shows the picker.
   - All exception-to-error mapping lives in `ShellViewModel`; views only display
     `Shell.LastError` when present.

8. **Tests.**
   - `DatabaseOptionsTests` — round-trip through `JsonSettingsService` round-trips the
     new `Database` section.
   - `JsonDatabaseConnectionStoreTests` — `Add` assigns a Guid; `SetActive` updates
     `ActiveConnectionId`; `Remove` clears `ActiveConnectionId` if it was the active
     entry.
   - `PasswordHasherTests` — known-vector test: `hash("hunter2")` produces the
     SHA-256 hex of `"hunter2spdb_42"`; identical inputs produce identical hashes;
     different passwords differ.
   - `PasswordServiceTests` — using a fake `ISpdbConfigRepository`, verifies
     `HasPasswordAsync`, `VerifyAsync` (true / false), `SetPasswordAsync` writes the
     correct row name + value.
   - `DatabaseBootstrapperTests` — using a fake `ISpdbConfigRepository` +
     `IDbConnectionFactory`, verifies all four `DatabaseBootstrapStatus` paths.
   - `ShellViewModelTests` — drives the state machine with all fakes:
     no-password → SettingPassword; existing-password + correct → Ready; wrong →
     stays in LoggingOn with `LastError` set; revision mismatch → Error.
   - `SpdbConfigRepositoryTests` — see Testing Strategy below; gated on a real
     MariaDB available via test config, otherwise `Skip`-ped.

9. **Documentation.**
   - Add [docs/projects/rebuild/milestones/m2-database-auth.md](../../../projects/rebuild/milestones/m2-database-auth.md)
     per the roadmap's milestone-doc template. Include the schema-loading
     prerequisite step (load `structure.sql` into `spdb_tests`, seed
     `db_revision = 35`).
   - Update `README.md` "Build & Run" section:
     - Prerequisite is a reachable MariaDB with the legacy schema at
       `db_revision = 35`.
     - Document the plaintext-password trust boundary.
     - Add a "Running the tests" subsection: copy `tests/test-config.dist.json` to
       `tests/test-config.json` and fill in credentials, **or** set
       `VIDEOINDEXER_TEST_DB`. Note that the integration class self-skips when
       neither is configured, so the rest of the suite still runs cleanly.

## Dependencies

- **Prior milestone:** M1 Foundation (complete; 41/41 tests passing).
- **External tooling:** a developer-accessible MariaDB instance is **required** for
  the integration test class and for manual smoke tests. The test schema name is
  `spdb_tests` and is seeded with a **copy of production data**, so tests must treat
  it as live data: every write is wrapped in a `MySqlTransaction` that is always
  rolled back, and no test issues `CREATE`/`DROP`/`ALTER` (the schema is loaded from
  [`spdb-indexer/SPDB Indexer/sql/structure.sql`](../../../../../spdb-indexer/SPDB%20Indexer/sql/structure.sql)
  outside the test runner). Unit tests for the rest of M2 use fakes and do not
  require a DB.
- **Test configuration files:**
  - [`tests/test-config.dist.json`](../../../../tests/test-config.dist.json) — committed template.
  - `tests/test-config.json` — gitignored, contains the developer's actual
    credentials. Already seeded for this workspace with
    `host=localhost`, `database=spdb_tests`, `username=root`, `password=AnyPassword`.
- **NuGet packages** (versions resolved at implementation time, all stable):
  - `MySqlConnector` (Infrastructure)
  - `Dapper` (Infrastructure)
  - No new App-side packages.
  - Tests: existing xUnit + FluentAssertions stack, plus `Xunit.SkippableFact` for
    the integration test class.
- **Specifications that must remain stable for M2:** the §1 launch flow in
  [system-management-specification.md](../../../projects/rebuild/management-areas/system-management-specification.md)
  and the data-stack section of
  [rebuild.md](../../../projects/rebuild/rebuild.md).

## Required Components

All paths below are workspace-relative. **(NEW)** items do not exist; **(MODIFIED)**
items already exist and gain content.

- `src/VideoIndexer.Core/Options/DatabaseOptions.cs` (NEW)
- `src/VideoIndexer.Core/Options/DatabaseConnectionOptions.cs` (NEW)
- [`src/VideoIndexer.Core/Options/AppOptions.cs`](../../../../src/VideoIndexer.Core/Options/AppOptions.cs) (MODIFIED)
- `src/VideoIndexer.Core/Abstractions/IDbConnectionFactory.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/ISpdbConfigRepository.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IDatabaseBootstrapper.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IPasswordHasher.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IPasswordService.cs` (NEW)
- `src/VideoIndexer.Core/Abstractions/IDatabaseConnectionStore.cs` (NEW)
- `src/VideoIndexer.Core/Enums/DatabaseBootstrapStatus.cs` (NEW)
- `src/VideoIndexer.Core/Enums/ShellState.cs` (NEW)
- `src/VideoIndexer.Core/Events/ShellStateChangedEventArgs.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Database/MySqlConnectionFactory.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Database/SpdbConfigRepository.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Database/DatabaseBootstrapper.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Auth/Sha256FixedSaltPasswordHasher.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Auth/PasswordService.cs` (NEW)
- `src/VideoIndexer.Infrastructure/Settings/JsonDatabaseConnectionStore.cs` (NEW)
- [`src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj`](../../../../src/VideoIndexer.Infrastructure/VideoIndexer.Infrastructure.csproj) (MODIFIED — package refs)
- `src/VideoIndexer.App/ViewModels/ShellViewModel.cs` (NEW)
- `src/VideoIndexer.App/ViewModels/DatabaseConnectorViewModel.cs` (NEW)
- `src/VideoIndexer.App/ViewModels/LogonViewModel.cs` (NEW)
- `src/VideoIndexer.App/ViewModels/PasswordSetupViewModel.cs` (NEW)
- `src/VideoIndexer.App/ViewModels/MainContentViewModel.cs` (NEW)
- [`src/VideoIndexer.App/ViewModels/MainWindowViewModel.cs`](../../../../src/VideoIndexer.App/ViewModels/MainWindowViewModel.cs) (MODIFIED)
- `src/VideoIndexer.App/Views/DatabaseConnectorView.axaml(.cs)` (NEW)
- `src/VideoIndexer.App/Views/LogonView.axaml(.cs)` (NEW)
- `src/VideoIndexer.App/Views/PasswordSetupView.axaml(.cs)` (NEW)
- `src/VideoIndexer.App/Views/MainContentView.axaml(.cs)` (NEW)
- [`src/VideoIndexer.App/Views/MainWindow.axaml`](../../../../src/VideoIndexer.App/Views/MainWindow.axaml) (MODIFIED)
- [`src/VideoIndexer.App/Program.cs`](../../../../src/VideoIndexer.App/Program.cs) (MODIFIED — DI registrations)
- [`src/VideoIndexer.App/Assets/appsettings.json`](../../../../src/VideoIndexer.App/Assets/appsettings.json) (MODIFIED)
- `tests/VideoIndexer.Tests/DatabaseOptionsTests.cs` (NEW)
- `tests/VideoIndexer.Tests/ShellViewModelTests.cs` (NEW)
- `tests/VideoIndexer.Tests/PasswordHasherTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/Database/DatabaseBootstrapperTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/Database/SpdbConfigRepositoryTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/Auth/PasswordServiceTests.cs` (NEW)
- `tests/VideoIndexer.Infrastructure.Tests/Settings/JsonDatabaseConnectionStoreTests.cs` (NEW)
- `docs/projects/rebuild/milestones/m2-database-auth.md` (NEW)
- [`README.md`](../../../../README.md) (MODIFIED — prereq + trust boundary section)
- [`tests/test-config.dist.json`](../../../../tests/test-config.dist.json) (already created — template for the gitignored `tests/test-config.json`)
- [`.gitignore`](../../../../.gitignore) (already updated — adds `tests/test-config.json`)

No external services, no infrastructure provisioning, no CI changes.

## Assumptions

- A reachable MariaDB instance is available to the developer (`localhost`, schema
  `spdb_tests`, user `root` / `AnyPassword` per the seeded `tests/test-config.json`)
  for both the integration test class and manual smoke tests. The schema is seeded
  with a copy of production data; tests must therefore be strictly read-only or
  rollback-only. CI does not require a DB; the integration class self-skips when no
  config is found.
- The legacy `spdb_config` schema (`config_name VARCHAR(200), config_value TEXT`) is
  unchanged; the rebuild does not alter it.
- `db_revision = 35` is the current production revision and acceptable as the M2
  baseline. Future revisions ship in the WP that introduces them.
- The single-user desktop trust model holds: filesystem ACLs on `%APPDATA%` are the
  defense for the plaintext DB password.
- `MySqlConnector` 2.x supports `MySqlConnectionStringBuilder` and standard async
  `OpenAsync`; this is the maintained successor to `MySql.Data` and works on .NET 8.
- The M1 `IThemeService.SetAsync` (post-rework rename) and the `Ctrl+T` debug command
  do not need to change in M2 — only their reachability via the new `Ready`-state
  routing in `MainWindowViewModel`.
- Localisation (English / German / French) remains deferred per M1's stance; all M2
  UI text is plain XAML literals.

## Constraints

- Must target **.NET 8** (LTS); test-project framework targets unchanged from M1.
- Must build cleanly under `TreatWarningsAsErrors=true` and `<WarningLevel>9999</WarningLevel>`.
- Must not break any M1 acceptance criterion: empty themed window, log file written,
  settings file at `%APPDATA%\VideoIndexer\appsettings.json`, theme cycle works.
- Must not introduce a dependency on Windows-only APIs (cross-platform constraint
  from `rebuild.md`). DPAPI is therefore explicitly out.
- Must use Dapper for SQL access; no Entity Framework.
- Must use `MySqlConnector`, not the older `MySql.Data` package.
- Connection passwords must never be passed via process command-line arguments.

## Out of Scope

- **Library Management** UI, folder registration, indexing, hashing — M3.
- **Startup Selection** screen (open library / add folders / open last) — deferred to
  M3 because all three options target M3-only surfaces.
- **Preferences dialog UI** — M8. M2 surfaces no UI for editing the Database section
  beyond the connector picker (the picker can add/edit/remove saved connections
  inline, but the broader Preferences dialog is M8).
- **Database backup** (`mysqldump`) — M8.
- **Schema migrations** — M2 only checks revision. Migration handlers ship in the WP
  that bumps the constant past 35.
- **Localisation runtime** (resx, language switch) — future WP.
- **DPAPI / Keychain / libsecret** encryption of the persisted DB password — future
  WP if/when the trust model changes.
- **Multi-user accounts** — explicitly out per spec ("single password system").
- **Connection-test button** in the connector form — deferred; the picker tests the
  connection by attempting to connect on confirm.
- **Saved-library snapshots** referenced in §1.4 — M3.

## Acceptance Criteria

- [ ] `dotnet restore && dotnet build -c Release` from a fresh clone produces zero
      warnings and zero errors across all projects.
- [ ] All M1 unit tests still pass (41/41 baseline).
- [ ] All new M2 unit tests pass under `dotnet test`. Live-MariaDB integration tests
      (`SpdbConfigRepositoryTests`) are `Skip`-ped when no test connection string is
      configured and pass when one is.
- [ ] On launch with **no saved connections**, the `DatabaseConnectorView` is shown,
      the M1 main content area is **not** visible, and the user can add a connection,
      have it persisted to `appsettings.json`, and connect.
- [ ] On launch with one saved connection set as active, the connector view is
      bypassed and the next state (`LoggingOn` or `SettingPassword`) is shown.
- [ ] If `spdb_config` is missing, the user sees an error explaining the schema must
      be initialised manually; the app does not bootstrap the database.
- [ ] If `db_revision != 35`, the user sees an error stating the actual vs expected
      revision, with separate copy for too-old vs too-new; the shell stays in `Error`.
- [ ] If no `app_password_hash` row exists, `PasswordSetupView` is shown; on submit,
      a row is written with the SHA-256 hex of `password + "spdb_42"`, and the shell
      transitions to `Ready`.
- [ ] If the row exists, `LogonView` is shown; correct password transitions to
      `Ready`; wrong password keeps the user on `LogonView` with an error message and
      no DB writes.
- [ ] `Ready` state shows the (empty) `MainContentView`. `Ctrl+T` (debug-only) still
      cycles the theme.
- [ ] `appsettings.json` written by M2 is backward-readable by M1 logic (the new
      `Database` section is purely additive).
- [ ] DB password is **not** passed on any process command line, written to logs at
      any verbosity, or echoed in error messages.
- [ ] [docs/projects/rebuild/milestones/m2-database-auth.md](../../../projects/rebuild/milestones/m2-database-auth.md)
      exists and conforms to the milestone-doc template.

## Testing Strategy

- **Unit tests (xUnit + FluentAssertions)** — the bulk of M2 coverage:
  - `PasswordHasherTests` — known-vector + determinism.
  - `PasswordServiceTests`, `DatabaseBootstrapperTests`,
    `JsonDatabaseConnectionStoreTests`, `DatabaseOptionsTests`,
    `ShellViewModelTests` — drive the production code through fakes
    (`InMemorySpdbConfigRepository`, `FakeDbConnectionFactory`,
    `InMemorySettingsService`).
  - Test fixtures (`TempDirectory`, `FakeAppPaths`, `InMemorySettingsService`) are
    extracted to `tests/VideoIndexer.Tests/Fixtures/` (carry-over from M1's
    recommendation #4) and reused.
- **Integration test (recommended; runs locally by default)** —
  `SpdbConfigRepositoryTests` connects to a real MariaDB to verify Dapper SQL is
  correct against the legacy schema. Configuration source order:
  1. Environment variable `VIDEOINDEXER_TEST_DB` (full connection string), then
  2. [`tests/test-config.json`](../../../../tests/test-config.json) (gitignored;
     template at [`tests/test-config.dist.json`](../../../../tests/test-config.dist.json)).

  When neither source resolves, the entire test class is `Skip`-ped via
  `[SkippableFact]` from the `Xunit.SkippableFact` NuGet package (added to the
  Infrastructure.Tests project only). Test discipline (the `spdb_tests` schema is
  seeded with a copy of production data — these rules are mandatory, not advisory):

  - Every test that writes wraps its work in a `MySqlTransaction` that is
    **always rolled back** in `Dispose` (success, failure, or cancellation).
    Read-only tests do not need a transaction but must not depend on row counts.
  - Tests touch `spdb_config` only.
  - Tests **never** issue `CREATE`, `DROP`, `ALTER`, or `TRUNCATE` statements.
  - Tests **never** call `MySqlTransaction.Commit()`.
  - Test row keys (`config_name`) used for write-then-rollback assertions are
    namespaced with a `vi_test_` prefix so a forgotten rollback would leave an
    obviously-test row rather than overwriting `db_revision` or
    `app_password_hash`.
  - Tests that assert on production rows (e.g. read-back of `db_revision`) treat
    those rows as read-only and never delete or update them.
  - On connection-open failure during fixture setup, the class self-skips with a
    clear message naming both the env var and the json file path; never a hard
    failure.
  - The fixture verifies `spdb_config` exists at startup and skips with a
    "schema not loaded — run `structure.sql` against `spdb_tests`" message if
    not, so a fresh developer machine gets an actionable error rather than a
    misleading test failure.
- **Manual smoke test** documented in `m2-database-auth.md`:
  0. **Prerequisite:** the `spdb_tests` schema exists on the local MariaDB and is
     seeded with a copy of production data (already done for this workspace).
     Verify `SELECT config_value FROM spdb_config WHERE config_name='db_revision'`
     returns `35`. If the smoke test needs to mutate `db_revision` or
     `app_password_hash`, **note the original value first and restore it after**
     — do not leave the test DB in a degraded state.
  1. Drop the row `app_password_hash` from `spdb_config` if present.
  2. Launch — confirm Connector → Password Setup → Ready flow.
  3. Restart — confirm Connector (auto-skipped if active connection set) → Logon →
     Ready.
  4. Set `db_revision` to 34 then 36 and confirm both error variants appear.
  5. Drop the `spdb_config` table and confirm the "schema not initialised" error.
- **No Avalonia Headless tests** in M2 — the M1 recommendation to add them is
  carried forward but is a separate WP.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Plaintext DB password in `appsettings.json` is a credential-disclosure risk if `%APPDATA%` is shared.** | Document the trust boundary explicitly in README and `m2-database-auth.md`. Add a security-audit note that flags the deferred DPAPI/libsecret WP. Verify the password never appears in logs at any verbosity (covered by an explicit unit test on `LoggingSetup` log enrichment if needed). |
| **`tests/test-config.json` is accidentally committed.** | The file is added to `.gitignore` as part of M2; the template `test-config.dist.json` ships in its place. Reviewer to verify `git ls-files` does not list `test-config.json` before merge. |
| **`spdb_tests` is seeded with a copy of production data; a forgotten `Commit()` or a destructive query would corrupt that data.** | Mandatory rules in Testing Strategy: no `Commit()` ever, all writes in always-rollback transactions, namespaced `vi_test_` row keys for any write-back assertion, no DDL of any kind. A `MySqlConnectionStringBuilder.AllowUserVariables=false` setting is enforced on the test connection to make accidental multi-statement injection harder. Reviewer checklist item to grep new tests for `\.Commit\(`. |
| **A developer points the test config at a different DB than `spdb_tests` and the rollback rule still leaks a stray test row to whatever they pointed at.** | The fixture asserts the resolved schema name equals `spdb_tests` (case-insensitive) and skips the entire class with an explanatory message otherwise. Override requires editing the fixture, not just the config. |
| **`SpdbConfigRepositoryTests` becomes flaky in CI without a DB.** | Tests are `Skip`-ped when no test connection string is configured. CI continues to be M1-style fakes only until a DB-in-CI WP is planned. |
| **Saved-connection list grows unbounded on disk.** | Picker exposes Remove + a hard cap is unnecessary at this scale; documented as a non-issue. |
| **State-machine deadlocks if a transition handler throws.** | All async transition methods (`ConnectAsync`, `LogonAsync`, `SetPasswordAsync`) wrap their work in `try/catch` that always lands the shell in a recoverable state (`Connecting` or `Error`). Covered by `ShellViewModelTests`. |
| **Schema-revision drift across installs.** | The `ExpectedRevision = 35` constant is the single source of truth; the user-facing error explicitly names actual vs expected so a mismatched DB is diagnosable from the dialog alone. |
| **Hash format change is a one-way door.** | Documented as a known limitation in `m2-database-auth.md`; future format changes require a migration WP. SHA-256 + fixed salt is the spec-compliant minimum. |
| **`MySqlConnector.OpenAsync` hangs on unreachable host with no timeout.** | `DatabaseConnectionOptions` carries an explicit `ConnectionTimeout` (default 5 s) passed to `MySqlConnectionStringBuilder`. The connector view surfaces the timeout error in the `Error` state. |
| **Adding `Database` section breaks M1's settings round-trip.** | The new section has full default values; existing `appsettings.json` files without it deserialise to the default `DatabaseOptions()`. Covered by `DatabaseOptionsTests`. |
| **CommunityToolkit.Mvvm + Dapper analyzers may emit new warnings under `WarningLevel=9999`.** | First build catches them; suppress per-line with `#pragma` *only* with a justification comment, otherwise fix at the source. No project-wide suppressions. |

AGENT: Planning
STATUS: READY_FOR_PM
