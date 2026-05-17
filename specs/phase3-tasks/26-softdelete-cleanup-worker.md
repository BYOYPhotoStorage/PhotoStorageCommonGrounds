# Task 26 — Soft-Delete Cleanup Worker

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Run a daily `WorkManager` job that hard-deletes `uploads` rows with `status = 'cloud_deleted'` whose `cloud_deleted_at` is older than 24 hours. This keeps the SQLite index from accumulating tombstone rows indefinitely.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/worker/SoftDeleteCleanupWorker.kt`
- `app/src/main/java/com/hriyaan/photostorage/worker/SoftDeleteCleanupScheduler.kt`

## `SoftDeleteCleanupWorker`

```kotlin
class SoftDeleteCleanupWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result = withContext(Dispatchers.IO) {
        val app = applicationContext as PhotoBackupApp
        val dao = app.uploadDatabase.dao
        val cutoff = System.currentTimeMillis() - TimeUnit.HOURS.toMillis(24)

        val stale = dao.getSoftDeletedOlderThan(cutoff)
        for (record in stale) {
            dao.hardDelete(record.id)
        }
        Result.success()
    }
}
```

- Idempotent: a second run finds nothing to do.
- No notifications.
- Runs even if auto-upload is disabled — the soft-delete tombstones are a database-hygiene concern, not an upload concern.

## `SoftDeleteCleanupScheduler`

```kotlin
object SoftDeleteCleanupScheduler {
    private const val WORK_NAME = "soft_delete_cleanup"

    fun schedule(context: Context) {
        val request = PeriodicWorkRequestBuilder<SoftDeleteCleanupWorker>(
            repeatInterval = 1, repeatIntervalTimeUnit = TimeUnit.DAYS,
            flexTimeInterval = 6, flexTimeIntervalUnit = TimeUnit.HOURS
        )
            .setConstraints(
                Constraints.Builder()
                    .setRequiresBatteryNotLow(true)
                    .build()
            )
            .setInitialDelay(1, TimeUnit.HOURS) // small delay so first launch doesn't churn immediately
            .build()

        WorkManager.getInstance(context).enqueueUniquePeriodicWork(
            WORK_NAME,
            ExistingPeriodicWorkPolicy.KEEP,
            request
        )
    }
}
```

- Single periodic job per device (`ExistingPeriodicWorkPolicy.KEEP` ensures repeated schedule calls don't duplicate).
- 1 day with a 6-hour flex window — Android will schedule efficiently.
- `setRequiresBatteryNotLow(true)` is a courtesy constraint; the work itself is trivial.

## Implementation notes

- Reuses the singleton `UploadDao` from `PhotoBackupApp`.
- `System.currentTimeMillis()` is used directly — no need to inject a clock.
- The 24h window is hardcoded per PRD resolution. If this ever becomes user-configurable, hoist `HOURS_24` to a constant and read from a setting; out of scope for Phase 3.

## Constraints

- The 24h window is locked at 24 hours (PRD constraint #4).
- Do not delete rows with `status != 'cloud_deleted'`.
- Do not delete rows where `cloud_deleted_at IS NULL` — the DAO query already filters these out, but the worker should treat the DAO result as authoritative.

## Dependencies (by interface)

- `UploadDao` (Task 20) — `getSoftDeletedOlderThan`, `hardDelete`
- `PhotoBackupApp` (singleton accessor for the DAO)
- `androidx.work:work-runtime-ktx` (already added by Phase 2 Task 19)

## Acceptance criteria

- [ ] A row with `status = 'cloud_deleted'` and `cloud_deleted_at = now - 25h` is hard-deleted when the worker runs.
- [ ] A row with `status = 'cloud_deleted'` and `cloud_deleted_at = now - 23h` is preserved.
- [ ] A row with `status = 'uploaded'` is never touched.
- [ ] `SoftDeleteCleanupScheduler.schedule()` enqueues exactly one periodic job (`enqueueUniquePeriodicWork` semantics).
- [ ] The worker exits with `Result.success` even when there are no rows to clean up.
- [ ] No notifications are surfaced by this worker.

## Out of scope

- Configurable retention window.
- Trash / "recently deleted" UI.
- Telling the user how many tombstones were cleaned.
