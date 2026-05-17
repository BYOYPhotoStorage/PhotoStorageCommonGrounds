# Task 27 — SQLite Index Sync Worker

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Run a nightly `WorkManager` job that uploads the local SQLite database to B2 at `index/photo-storage-index.sqlite`, but only if the file's SHA-256 has changed since the last successful sync. Also expose a one-shot manual trigger.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/worker/IndexSyncWorker.kt`
- `app/src/main/java/com/hriyaan/photostorage/worker/IndexSyncScheduler.kt`

## Constants

```kotlin
const val INDEX_B2_PATH = "index/photo-storage-index.sqlite"
```

Define in a `Companion` of `IndexSyncWorker` so Task 28 can reference it.

## `IndexSyncWorker`

```kotlin
class IndexSyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    companion object {
        const val INDEX_B2_PATH = "index/photo-storage-index.sqlite"
    }

    override suspend fun doWork(): Result = withContext(Dispatchers.IO) {
        val app = applicationContext as PhotoBackupApp
        val prefs = app.prefsStore
        val s3 = app.s3Uploader

        // 1. Flush WAL so the DB file is self-contained
        try {
            app.uploadDatabase.writableDatabase.use { db ->
                db.execSQL("PRAGMA wal_checkpoint(TRUNCATE)")
            }
        } catch (t: Throwable) {
            return@withContext Result.retry()
        }

        // 2. Copy to a temp file
        val src = applicationContext.getDatabasePath("uploads.db")
        if (!src.exists()) return@withContext Result.success()
        val temp = File(applicationContext.cacheDir, "index-${System.currentTimeMillis()}.sqlite")
        try {
            src.copyTo(temp, overwrite = true)
        } catch (t: Throwable) {
            temp.delete()
            return@withContext Result.retry()
        }

        try {
            // 3. Compute hash
            val hash = sha256Hex(temp)

            // 4. Compare
            if (hash == prefs.getLastSyncedIndexHash()) {
                return@withContext Result.success() // no-op
            }

            // 5. Upload
            val uploadResult = s3.uploadFile(INDEX_B2_PATH, temp) // existing upload entrypoint
            if (uploadResult.isFailure) return@withContext Result.retry()

            // 6. Persist new hash + timestamp
            prefs.setLastSyncedIndexHash(hash)
            prefs.setLastIndexSyncAt(System.currentTimeMillis())
            Result.success()
        } finally {
            temp.delete()
        }
    }
}
```

`sha256Hex(File)` is a small helper that streams the file through `MessageDigest.getInstance("SHA-256")` in 64 KB chunks. Reuse Phase 2's hash helper from `DuplicateDetector` if it's accessible; otherwise inline it.

## `IndexSyncScheduler`

```kotlin
object IndexSyncScheduler {
    private const val PERIODIC_NAME = "index_sync_nightly"
    private const val ONE_SHOT_NAME = "index_sync_now"

    fun schedule(context: Context) {
        val initialDelayMs = msUntilNext(localHour = 3, localMinute = 0)
        val request = PeriodicWorkRequestBuilder<IndexSyncWorker>(
            repeatInterval = 1, repeatIntervalTimeUnit = TimeUnit.DAYS,
            flexTimeInterval = 1, flexTimeIntervalUnit = TimeUnit.HOURS
        )
            .setConstraints(buildConstraints(context))
            .setInitialDelay(initialDelayMs, TimeUnit.MILLISECONDS)
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.MINUTES)
            .build()

        WorkManager.getInstance(context).enqueueUniquePeriodicWork(
            PERIODIC_NAME,
            ExistingPeriodicWorkPolicy.UPDATE,
            request
        )
    }

    fun cancel(context: Context) {
        WorkManager.getInstance(context).cancelUniqueWork(PERIODIC_NAME)
    }

    fun runNow(context: Context) {
        val request = OneTimeWorkRequestBuilder<IndexSyncWorker>()
            .setConstraints(buildConstraints(context))
            .build()
        WorkManager.getInstance(context).enqueueUniqueWork(
            ONE_SHOT_NAME,
            ExistingWorkPolicy.REPLACE,
            request
        )
    }

    private fun buildConstraints(context: Context): Constraints {
        val prefs = (context.applicationContext as PhotoBackupApp).prefsStore
        val networkType = if (prefs.isWifiOnly()) NetworkType.UNMETERED else NetworkType.CONNECTED
        return Constraints.Builder()
            .setRequiredNetworkType(networkType)
            .build()
    }

    private fun msUntilNext(localHour: Int, localMinute: Int): Long {
        val now = Calendar.getInstance()
        val target = (now.clone() as Calendar).apply {
            set(Calendar.HOUR_OF_DAY, localHour)
            set(Calendar.MINUTE, localMinute)
            set(Calendar.SECOND, 0)
            set(Calendar.MILLISECOND, 0)
        }
        if (target.timeInMillis <= now.timeInMillis) {
            target.add(Calendar.DAY_OF_MONTH, 1)
        }
        return target.timeInMillis - now.timeInMillis
    }
}
```

- **03:00 local** is 1 hour after the Phase 2 nightly scan (02:00) so the scan completes first.
- `runNow` uses `ExistingWorkPolicy.REPLACE` so back-to-back manual triggers debounce naturally.
- Constraints respect the Phase 2 `wifi_only_uploads` setting.

## Implementation notes

- Always upload from a temp file, never from `uploads.db` directly — SQLite may write to the live file mid-upload otherwise.
- Use the existing `S3Uploader.uploadFile(path, file)` entrypoint (whichever name MVP/Phase 2 settled on for file uploads). If the existing entrypoint only accepts a `Uri` or `ByteArray`, wrap the temp file into the supported argument.
- If `getDatabasePath("uploads.db")` does not exist (extreme corner case on a freshly opened install), exit with `Result.success` — there is nothing to sync.
- Keep the file `<close>`d as quickly as possible. `writableDatabase.use { }` calls `close()` after the WAL checkpoint.
- The hash storage in `PrefsStore` is plain hex — no base64, no truncation.

## Constraints

- The B2 path is fixed at `index/photo-storage-index.sqlite` — do not version it (PRD says we always overwrite).
- The hash gate is the ONLY de-dup mechanism. Do not also store a timestamp comparison.
- The worker must NOT write to the DB itself. Reading via `PRAGMA wal_checkpoint` is fine.
- No encryption applied to the index file before upload (PRD open-question #1 → no need).
- Do not surface failure notifications. Logging at `Log.w` is sufficient.

## Dependencies (by interface)

- `PrefsStore` (Task 21) — `getLastSyncedIndexHash`, `setLastSyncedIndexHash`, `setLastIndexSyncAt`, `isWifiOnly`
- `S3Uploader` (existing — `uploadFile`)
- `UploadDatabase` (singleton) — for the WAL checkpoint
- `androidx.work:work-runtime-ktx`

## Acceptance criteria

- [ ] First-ever run uploads the index and stores `last_synced_index_hash` and `last_index_sync_at`.
- [ ] Second consecutive run (no DB changes) exits as `Result.success` without an upload (hash gate).
- [ ] A run after any DB write (insert/update/delete) detects a hash change and re-uploads.
- [ ] On B2 failure (5xx / network), the worker returns `Result.retry()` and the prefs are NOT updated.
- [ ] Manual `runNow` succeeds and the gallery status line updates after completion.
- [ ] With `wifi_only_uploads = true`, the worker only runs on unmetered networks.
- [ ] The local DB file is never the upload source — always the temp copy.

## Out of scope

- Differential / incremental sync.
- Multi-version retention (B2 lifecycle / versioning is the user's bucket-level concern, not the app's).
- Conflict resolution if two devices upload simultaneously — last-write-wins is acceptable for Phase 3.
- Compression of the index before upload.
