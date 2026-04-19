# Follow-Up Ticket: Improve Task Tracker's Input Validation, Query Performance, and Test Coverage

## Title

Improve task creation input validation, fix database overfetching, and expand test coverage

## Priority / Severity

**Priority:** High

**Severity:** Sev-2

**Rationale:** The 500 errors on task creation directly impacted users' ability to create tasks in production and were reported by clients. The overfetching query poses a scaling risk that will worsen as task volume grows. Both issues had production impact and were client-reported; however, no data loss occurred and a workaround exists for task creation (retrying), so this does not rise to Sev-1.

## Description

Three bugs were identified and fixed during incident investigation (see INCIDENT.md for full details):

1. **Intermittent 500 errors on task creation:** Client-side code in `main.js` randomly omitted the `X-Client-Timestamp` header via a `Math.random()` conditional. Server-side code in `TaskEndpoints.cs` called `DateTime.Parse` on the raw value without validation, causing unhandled `FormatException` on empty strings. Fix: removed the random conditional client-side; added null/empty check and `DateTime.TryParse` server-side to return HTTP status code 400 with a user-facing error message.

2. **Duplicate/out-of-order tasks after refresh:** The refresh function in `main.js` concatenated fetched tasks onto `state.tasks` without clearing the array. Fix: added `state.tasks = []` before concatenation.

3. **Slow task list at scale:** EF Core query in `TaskEndpoints.cs` called `.ToListAsync()` before filtering/sorting/limiting, loading the entire table into memory. Fix: moved `.Where`, `.OrderByDescending`, and `.Take` before `.ToListAsync()`.

This ticket covers deploying the fixes and completing the follow-up work identified during the incident.

## Acceptance Criteria

- [ ] All three fixes (client-side header, client-side refresh, server-side query) are deployed to production.
- [ ] Post-deploy: create 20+ tasks with zero 500 errors; confirm `X-Client-Timestamp` is present on all requests via Network tab.
- [ ] Post-deploy: refresh task list multiple times with no duplicates and consistent sort order.
- [ ] Post-deploy: confirm EF Core query includes `WHERE`, `ORDER BY`, and `LIMIT` clauses (check application logs).
- [ ] Two new test cases (missing timestamp should return HTTP status code 400, invalid timestamp should return HTTP status code 400) pass via `dotnet test`.
- [ ] Load testing validates acceptable list performance at 500+ and 1000+ tasks.

## Notes / Context

**Monitoring:**
- Consider adding structured logging for request validation failures (e.g., missing headers, parse failures) to aid future debugging.

**Testing:**
- Current test cases assert on status code only (400 vs. 500). Expand to also assert on the response body/error message.
- The test project had setup issues (missing `Microsoft.NET.Test.Sdk` package, missing `using Xunit;` directive) that should be documented or fixed in the repo to avoid onboarding friction (see SETUP_ISSUES.md).

**Codebase review:**
- Audit other endpoints for unvalidated header or parameter parsing that could produce unhandled exceptions.
- Audit other EF Core queries for overfetching patterns (`.ToListAsync()` called before filtering/sorting/limiting).

**Open question:**
- Engineering was unable to reproduce the 500 error in their environment despite it being reproducible locally and in production. Investigate whether there was a version mismatch, caching behavior, or other environmental difference that prevented reproduction.
