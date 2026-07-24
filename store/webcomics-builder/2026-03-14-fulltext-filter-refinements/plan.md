# Plan — Fulltext Filter & Pagination Refinements

## Summary

Post-implementation refinements for the fulltext filter and pagination features delivered in the `2026-03-14-fulltext-filter-and-pagination` plan. All items originate from the strategic recommendations section of that plan's [synthesis document](../2026-03-14-fulltext-filter-and-pagination/synthesis.md), which contains the full rationale, affected files, and priority assessment for each item.

## Origin

These recommendations were raised consistently across multiple review agents (Developer, QA, Reviewer) during the original plan's pipeline execution. The Planner assessed that no new plan document is needed — the synthesis provides sufficient detail (exact files, methods, changes, rationale, priority) for direct work package creation.

## Scope

All changes target three source files in the `webcomics-builder` project:

| File | Changes |
|---|---|
| `assets/classes/Comics/Filter/ComicsFilter.php` | Visibility change (`isMatch()` → `protected`); receive `FILTER_STATUS` constant from subclass |
| `assets/classes/Comics/Filter/SessionComicsFilter.php` | Move `FILTER_STATUS` constant to base class |
| `assets/classes/Page/DetailedListPage.php` | Visibility change (`renderPagination()` → `protected`); fix disabled pagination link accessibility; fix label echo bug in `displayFilters()`; remove dead code in `renderPagination()` |

Plus new test files:

| File | Purpose |
|---|---|
| `tests/TestSuites/ComicsFilterTest.php` (new) | Unit tests for search filter logic |
| `tests/TestSuites/PaginationTest.php` (new) | Unit tests for pagination rendering boundary conditions |

## Grouping

| Group | Items | Priority | Effort |
|---|---|---|---|
| Immediate (low-effort, high-value) | Visibility changes: `isMatch()` + `renderPagination()` | HIGH / MEDIUM | Trivial |
| Short-term fixes | Accessibility fixes (disabled links, label echo) + dead code removal | MEDIUM / LOW | Low |
| Unit tests | Filter and pagination test coverage | LOW | Medium |
| Housekeeping | `FILTER_STATUS` constant consolidation | LOW | Trivial |

## Constraints

- All existing constraints from `docs/agents/project-manifest/constraints.md` apply.
- No behavioral changes — visibility changes, accessibility fixes, dead code removal, and constant moves are all refactoring.
- All 60 existing tests must continue to pass after each work package.
- PHPStan level 5 with 0 errors must be maintained.
