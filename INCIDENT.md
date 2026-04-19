# Incident Report: Task Tracker 500 Errors, Performance Degradation, and UI Duplication

## Impact

- **Task creation failures:** Users experienced intermittent 500 errors when creating tasks. The failure was random (driven by a `Math.random()` call in client-side code), meaning any user could be affected on any request.
- **Slow task list loading:** Users with larger task volumes experienced degraded load times (~6x slower at 200 tasks vs. 50 tasks) due to overfetching from the database.
- **Duplicate/out-of-order tasks:** All users saw tasks duplicated in the UI after refreshing, with inconsistent sort order as a side effect.

## Detection

- Issues #1, #2, and #3 were reported by clients.
- Issue #4 (500 errors in production) was reported by clients with an attached log snippet; engineering was unable to reproduce in their environment.
- Issue #3 (duplication/ordering) was also independently observable from the GUI during investigation.

## Timeline

1. Client reports received for: intermittent 500 errors on task creation, slow task list for some users, and duplicated/out-of-order tasks after refresh.
2. Application set up locally; ran `dotnet restore`, `dotnet run`, and `dotnet test`. Test project required fixes before tests could execute (see SETUP_ISSUES.md).
3. Reproduced the 500 error locally. Inspected failing requests via browser Network tab and identified that the `X-Client-Timestamp` header was empty on all failed requests.
4. Traced the issue to `main.js`, wherein the `addTask` function used `Math.random()` to conditionally populate the `X-Client-Timestamp` header.
5. Reviewed the production log snippet from issue #4 and confirmed the same root cause: empty `X-Client-Timestamp` causing a `DateTime.Parse` failure server-side.
6. Fixed the client-side code to always send the header. Added server-side validation in `TaskEndpoints.cs` (null/empty check + `DateTime.TryParse`) to return 400 instead of 500 on bad input. Added two test cases for the new validation.
7. Investigated the duplication issue. Found that the refresh function in `main.js` appended fetched tasks to `state.tasks` without clearing the array first. Added a reset before concatenation. The out-of-order issue was no longer reproducible after fixing duplication.
8. Investigated the slow list issue. Confirmed locally that load time was ~250 ms. Reviewed `sample_slow_list_log.txt` and observed disproportionate scaling at 200 tasks. Inspected the EF Core query and found it was fetching all records before filtering/sorting/limiting in memory. Moved filtering, sorting, and limiting before `.ToListAsync()` so the database handles them. Load time dropped to ~20 ms.

## Root Cause

Three separate bugs across client and server code:

1. **500 errors (Issues #1 and #4):** `main.js` used `Math.random()` to randomly omit the `X-Client-Timestamp` request header. Server-side code in `TaskEndpoints.cs` called `DateTime.Parse` on the raw header value without validation, causing an unhandled `FormatException` on empty strings.
2. **Duplicate/out-of-order tasks (Issue #3):** The refresh function in `main.js` concatenated newly fetched tasks onto the existing `state.tasks` array without clearing it first, causing duplicates on every refresh. Duplicate timestamps from the duplicated entries caused inconsistent sort behavior.
3. **Slow task list (Issue #2):** The EF Core query in `TaskEndpoints.cs` called `.ToListAsync()` before applying `WHERE`, `ORDER BY`, and `LIMIT`, causing the entire `Tasks` table to be loaded into memory before filtering.

## Mitigation / Resolution

- **Client-side (`main.js`):**
  - Removed the `Math.random()` conditional; `X-Client-Timestamp` is now always included in request headers.
  - Added `state.tasks = []` before concatenation in the refresh function.
- **Server-side (`TaskEndpoints.cs`):**
  - Added a null/empty check on `X-Client-Timestamp` that returns a 400 with a user-facing message directing them to contact application support.
  - Changed `DateTime.Parse` to `DateTime.TryParse` to catch invalid timestamp formats and return 400 instead of 500.
  - Moved EF Core filtering (`.Where`), sorting (`.OrderByDescending`), and limiting (`.Take`) before `.ToListAsync()` so the database handles them.
- **Tests (`TaskApiTests.cs`):**
  - Added two test cases: missing timestamp should return HTTP status code 400, invalid timestamp should return HTTP status code 400.
  - All tests pass.

## Verification

- Created 20+ tasks with no 500 errors; all requests include `X-Client-Timestamp` in the Network tab.
- Refreshed multiple times with no duplicate tasks and consistent sort order.
- Task list loads in ~20 ms (down from ~250 ms); EF Core query now includes `WHERE`, `ORDER BY`, and `LIMIT` clauses.
- `dotnet test` passes all test cases including the two new validation tests.

## Follow-Up Actions

- Investigate why engineering was unable to reproduce the 500 error locally (issue #4); determine if there was a different client version deployed, caching behavior, or other environmental differences.
- Add load testing to validate list performance at higher task counts (500+, 1000+).
- Review codebase for other instances of in-memory filtering on database queries.
- Review codebase for other instances of unvalidated header parsing.
- Consider expanding test coverage to assert on specific error messages, not just status codes.
- Consider front-end test framework e.g. Selenium to avoid regressions for client-side issues.