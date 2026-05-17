# Task 16 — WorkManager Nightly Scan

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Schedule and implement a nightly WorkManager job that performs a full MediaStore scan to catch any photos missed by the ContentObserver (photos added while the app was killed, restored from backup, etc.).

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/worker/NightlyScanWorker.kt`
- `app/src/main/java/com/hriyaan/photostorage/worker/NightlyScanScheduler.kt`

## Public contract

```kotlin
package com.hriyaan.photostorage.worker

import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters

class NightlyScanWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result
}

object NightlyScanScheduler {
    fun schedule(context: Context)
    fun cancel(context: Context)
}
```

## Scheduling

`NightlyScanScheduler.schedule` creates a `PeriodicWorkRequest`:

```kotlin
val constraints = Constraints.Builder()
    .setRequiresBatteryNotLow(true)
    .setRequiredNetworkType(
        if (PrefsStore(context).isWifiOnly()) NetworkType.UNMETERED
        else NetworkType.CONNECTED
    )
    .build()

val workRequest = PeriodicWorkRequestBuilder<NightlyScanWorker>(
    repeatInterval = 24, TimeUnit.HOURS,
    flexTimeInterval = 1, TimeUnit.HOURS
)
    .setConstraints(constraints)
    .setInitialDelay(calculateDelayUntil2am(), TimeUnit.MILLISECONDS)
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "nightly_scan",
    ExistingPeriodicWorkPolicy.UPDATE,
    workRequest
)
```

### Initial delay calculation

Calculate milliseconds until the next 02:00 local time:

```kotlin
private fun calculateDelayUntil2am(): Long {
    val now = Calendar.getInstance()
    val target = Calendar.getInstance().apply {
        set(Calendar.HOUR_OF_DAY, 2)
        set(Calendar.MINUTE, 0)
        set(Calendar.SECOND, 0)
        if (before(now)) add(Calendar.DAY_OF_YEAR, 1)
    }
    return target.timeInMillis - now.timeInMillis
}
```

## Worker behavior (`doWork`)

1. Read `PrefsStore.getLastScanTimestamp()`.
2. Query MediaStore for ALL images (same query as `MediaStoreQuery.queryAllPhotos()` but no `DATE_ADDED` filter — we want everything).
3. For each photo:
   - Run `DuplicateDetector.isDuplicate()`.
   - If not a duplicate, enqueue: insert into `uploads` with `status = 'pending'`.
4. Update `PrefsStore.setLastScanTimestamp(System.currentTimeMillis())`.
5. If any photos were enqueued, trigger the upload worker to start processing immediately.
6. Return `Result.success()`.

## Constraints

- The worker must be idempotent — running twice in a row should not create duplicates (the `DuplicateDetector` handles this).
- Do NOT perform uploads directly in the worker. It only scans and enqueues; the `UploadWorker` (Task 14) processes the queue.
- `WorkManager` must be added as a dependency. See Task 19 for the gradle change.
- Use `CoroutineWorker` (not `Worker`) so the scan runs in a coroutine.

## Dependencies (by interface)

- `MediaStoreQuery` (Task 04 / MVP) — for querying photos
- `DuplicateDetector` (Task 15) — for deduplication
- `UploadDao` (Task 10) — for enqueueing
- `PrefsStore` (Task 11) — for timestamp read/write
- `UploadWorker` (Task 14) — for triggering queue processing after enqueue

## Acceptance criteria

- [ ] `NightlyScanScheduler.schedule(context)` enqueues a unique periodic work that appears in `WorkManager.getWorkInfosByTag`.
- [ ] The worker runs at approximately 02:00 local time.
- [ ] The worker scans all MediaStore photos, not just recent ones.
- [ ] Duplicate photos are not enqueued (verified by `DuplicateDetector`).
- [ ] New photos are enqueued with `status = 'pending'`.
- [ ] After scanning, `lastScanTimestamp` is updated.
- [ ] `NightlyScanScheduler.cancel(context)` removes the scheduled work.
- [ ] When `wifi_only = true`, the work request uses `NetworkType.UNMETERED`.

## Out of scope

- Configurable scan time (hardcoded to 02:00).
- Scan frequency other than daily.
- Partial / incremental scan (always full scan).
