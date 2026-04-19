# Runbook: Task Tracker Issues

## Issue 1: 500 Errors on Task Creation

### Diagnosis

**What to look for:** Intermittent 500 errors on `POST` requests to the task creation endpoint.

**Log artifacts:** Server logs will show a `DateTime.Parse` failure when `X-Client-Timestamp` is missing or empty. Example log line:

```
System.FormatException: String '' was not recognized as a valid DateTime.
```

**Client-side check:** Open the browser's Network tab and inspect the request headers on failed `POST` calls. Successful requests will have a populated `X-Client-Timestamp` header; failed requests will have it empty or absent.

**Root cause location:**
- Client-side: Root cause found in `main.js`, wherein the `addTask` function contained a `Math.random()` conditional that randomly omitted the `X-Client-Timestamp` header.
- Server-side: Secondary cause found in `TaskEndpoints.cs`, wherein `DateTime.Parse` was called on the raw header value without null/empty checking, causing an unhandled exception on empty strings.

### How to Verify the Fix

1. Rebuild and run the application.
2. Open the browser Network tab.
3. Create multiple tasks (20+) and confirm all `POST` requests return 200. Also confirm all tasks were created and can be seen in the GUI.
4. Confirm all requests include a populated `X-Client-Timestamp` header.
5. Run `dotnet test` and confirm the two new test cases pass:
   - Missing `X-Client-Timestamp` should return HTTP status code 400 (not 500)
   - Invalid `X-Client-Timestamp` value should return HTTP status code 400 (not 500)

### Mitigation / Rollback

- **Client-side rollback:** Revert `main.js` to previous version. Note: this restores the random 500 errors, so this should only be done if the fix introduces a worse regression.
- **Server-side rollback:** Revert `TaskEndpoints.cs`. Note: without the server-side validation, missing timestamps will again cause unhandled 500 errors rather than a controlled 400 response.

---

## Issue 2: Slow Task List Loading

### Diagnosis

**What to look for:** Task list load times that scale disproportionately with task count. Example from `sample_slow_list_log.txt`:

| Task Count | Observed Load Time |
|---|---|
| 50 | ~250 ms |
| 200 | ~1500 ms (6x increase) |

**Query check:** Monitor the console while the application is running and click the refresh button. Before the fix, the query logged will be:

```sql
SELECT "t"."Id", "t"."CreatedAt", "t"."Status", "t"."Title", "t"."UpdatedAt", "t"."UserId"
FROM "Tasks" AS "t"
```

This fetches every record in the database without filtering, sorting, or limiting, then applies those operations in application memory.

**Root cause location:** In `TaskEndpoints.cs`, the EF Core query called `.ToListAsync()` before applying `WHERE`, `ORDER BY`, and `LIMIT` clauses, requiring every single task to be loaded into memory before performing filtering, sorting, and limiting.

### How to Verify the Fix

1. Rebuild and run the application.
2. Click the refresh button and check the console output. The logged query should now include `WHERE`, `ORDER BY`, and `LIMIT` clauses since this is now handled by the database query instead of the application logic:

```sql
SELECT "t"."Id", "t"."CreatedAt", "t"."Status", "t"."Title", "t"."UpdatedAt", "t"."UserId"
FROM "Tasks" AS "t"
WHERE "t"."UserId" = @__userId_0
ORDER BY "t"."CreatedAt" DESC
LIMIT @__p_1
```

3. Confirm load time is reduced (observed: ~20 ms vs. ~250 ms previously for the same dataset).

### Mitigation / Rollback

- Revert `TaskEndpoints.cs` to the previous version. The application will still function but will exhibit degraded performance at higher task counts.

---

## Issue 3: Duplicated / Out-of-Order Tasks After Refresh

### Diagnosis

**What to look for:** Tasks appearing duplicated in the UI after clicking the refresh button. Tasks may also appear in unexpected order.

**Root cause location:** In `main.js`, the refresh function appended newly fetched tasks to `state.tasks` via `.concat()` without clearing the existing array first, resulting in duplicates on every refresh. The out-of-order appearance was a side effect of duplicate entries introducing duplicate timestamps, which caused inconsistent sort behavior.

### How to Verify the Fix

1. Rebuild and run the application.
2. Add several tasks.
3. Click the refresh button multiple times.
4. Confirm no duplicate tasks appear in the list.
5. Confirm tasks remain in consistent order (most recent first) across refreshes.

### Mitigation / Rollback

- Revert `main.js` to the previous version. The application will still function but tasks will duplicate on every refresh.
