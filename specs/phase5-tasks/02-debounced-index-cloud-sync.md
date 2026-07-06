# Task 52 — Debounced SQLite index cloud sync

> Read [`../../architecture/android-system-design.md`](../../architecture/android-system-design.md) (especially §10 “Use-case scenario: repeated `make e2e-baseline` runs”), [`../phase3-tasks/27-index-sync-worker.md`](../phase3-tasks/27-index-sync-worker.md), and [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Make the SQLite index backup react to photo additions and deletions in near-real time, while still batching changes so a burst of photos produces only one index upload.

Whenever the `uploads` table is mutated, schedule a **debounced** index sync:

- Wait **10 seconds** after the most recent mutation.
- If another mutation occurs inside that 10-second window, reset the timer and wait another 10 seconds.
- When the timer finally expires, run the existing `IndexSyncWorker`, which still gates on SHA-256 so unchanged DBs never upload.

Also add an optional **minimum photo threshold**: if fewer than *N* mutations have accumulated since the last successful sync, skip the debounced upload and let the next nightly sync catch up. This avoids tiny index uploads (for example, deleting a single photo) when the user has configured a larger batch size.

## Why this matters for `make e2e-baseline`

Section 10 of the system-design doc explains that the current index backup runs once per day at 03:00. That is fine for disaster recovery, but it means E2E scenarios that want to assert remote index state must manually trigger `IndexSyncWorker` and wait for it.

With debounced sync, the remote index stays current automatically after bursts. However, tests still need deterministic waits: the debounce window is 10 seconds, and the threshold may delay small changes. Scenarios that need to inspect the index immediately should continue to call `IndexSyncScheduler.runNow()` rather than relying on the debouncer.

The debouncer does **not** change deduplication behavior. The local DB remains the source of truth; the remote index is still only read during recovery.

## Scope

- Extend `IndexSyncScheduler` with a debounced one-shot enqueue.
- Add an `IndexSyncDebouncer` helper that tracks pending-change count and decides whether to enqueue.
- Add preference keys for the threshold and pending-change counter.
- Reset the pending-change counter after every successful or no-op `IndexSyncWorker` run.
- Wire the debouncer into the main code paths that mutate `uploads`.

## Files you own

New file:

- `app/src/main/java/com/hriyaan/photostorage/worker/IndexSyncDebouncer.kt`

Modified files:

- `app/src/main/java/com/hriyaan/photostorage/worker/IndexSyncScheduler.kt`
- `app/src/main/java/com/hriyaan/photostorage/worker/IndexSyncWorker.kt`
- `app/src/main/java/com/hriyaan/photostorage/data/PrefsStore.kt`
- `app/src/main/java/com/hriyaan/photostorage/service/UploadForegroundService.kt`
- `app/src/main/java/com/hriyaan/photostorage/worker/UploadWorker.kt`
- `app/src/main/java/com/hriyaan/photostorage/gallery/DeletionEngine.kt`
- `app/src/main/java/com/hriyaan/photostorage/worker/NightlyScanWorker.kt`

## Constants and preference keys

Add to `PrefsStore`:

```kotlin
private const val KEY_INDEX_SYNC_MIN_PHOTOS = "index_sync_min_photos"
private const val KEY_INDEX_SYNC_PENDING_CHANGES = "index_sync_pending_changes"

private const val DEFAULT_INDEX_SYNC_MIN_PHOTOS = 1
```

Public accessors:

```kotlin
fun getIndexSyncMinPhotos(): Int
fun setIndexSyncMinPhotos(count: Int)
fun getIndexSyncPendingChanges(): Int
fun setIndexSyncPendingChanges(count: Int)
```

In `IndexSyncScheduler`:

```kotlin
private const val DEBOUNCE_MS = 10_000L
private const val ONE_SHOT_NAME = "index_sync_now"
```

Reuse the existing `ONE_SHOT_NAME` so manual “Back up index now” and the debouncer share the same unique work slot. The debouncer uses `ExistingWorkPolicy.REPLACE`, which gives the desired reset behavior.

## Implementation

### `IndexSyncScheduler.scheduleDebounced(context: Context)`

```kotlin
fun scheduleDebounced(context: Context) {
    val request = OneTimeWorkRequestBuilder<IndexSyncWorker>()
        .setInitialDelay(DEBOUNCE_MS, TimeUnit.MILLISECONDS)
        .setConstraints(buildConstraints(context))
        .build()

    WorkManager.getInstance(context).enqueueUniqueWork(
        ONE_SHOT_NAME,
        ExistingWorkPolicy.REPLACE,
        request
    )
}
```

### `IndexSyncDebouncer`

```kotlin
object IndexSyncDebouncer {
    private const val TAG = "IndexSyncDebouncer"

    fun trigger(context: Context) {
        val prefs = (context.applicationContext as PhotoBackupApp).prefsStore
        val threshold = prefs.getIndexSyncMinPhotos().coerceAtLeast(1)
        val pending = prefs.getIndexSyncPendingChanges() + 1
        prefs.setIndexSyncPendingChanges(pending)

        if (pending >= threshold) {
            FileLogger.getInstance(context).d(TAG, "trigger | pending=$pending >= threshold=$threshold, scheduling sync")
            IndexSyncScheduler.scheduleDebounced(context)
        } else {
            FileLogger.getInstance(context).d(TAG, "trigger | pending=$pending < threshold=$threshold, deferring to nightly sync")
        }
    }
}
```

The counter is incremented on every trigger. If the threshold has been reached, a debounced sync is scheduled; the existing unique-work `REPLACE` policy resets the 10-second timer on each new trigger.

### `IndexSyncWorker` reset

At the end of `doWork`, after a successful upload **and** after the hash-unchanged early return, reset the pending-change counter to zero:

```kotlin
prefs.setIndexSyncPendingChanges(0)
```

Do this only when the worker actually evaluates the DB. If the worker fails and returns `Result.retry()`, do **not** reset the counter, because the pending changes have not been reflected in the remote index.

### Call sites

Call `IndexSyncDebouncer.trigger(context)` from every code path that materially changes the `uploads` table. Prefer calling once per higher-level operation, not inside every inner loop.

1. **`UploadForegroundService.handleMediaChange()`**
   - After the scan loop, if `enqueued > 0`.

2. **`UploadWorker.processQueue()`**
   - After the queue loop finishes, before updating notifications. The loop mutates statuses, retry counts, and uploaded paths; a single trigger at the end is enough.

3. **`DeletionEngine`**
   - In `delete()` after `galleryRepository.invalidate()` when `result.changedState`.
   - In `deleteBatch()` after the loop, if `succeeded > 0 || failed > 0` (any state change).

4. **`NightlyScanWorker.doWork()`**
   - After inserting any newly discovered pending records.

5. **`LocalDeleteWorker` / `LocalDeleteReviewActivity`** *(if they mutate `uploads`)*
   - After `markPendingLocalDelete`, `clearPendingLocalDelete`, or `setLocalPresent` calls that affect the table.

Do **not** trigger from read-only operations, UI refreshes, or `GalleryRepository.invalidate()`.

## Constraints

- The debounce window is **10 seconds** and is approximate; WorkManager may delay slightly for Doze or network constraints.
- The debouncer must respect the existing `wifi_only_uploads` constraint via `buildConstraints()`.
- The existing manual “Back up index now” (`IndexSyncScheduler.runNow()`) and the nightly periodic sync remain unchanged.
- The SHA-256 hash gate in `IndexSyncWorker` remains the final authority on whether an upload actually happens.
- The pending-change counter must be reset after every successful/no-op sync so it does not grow unbounded.
- Threshold default is `1`, meaning the debouncer runs after the first mutation and behaves as a pure 10-second debounce. Users who want batching can raise it in settings.

## Acceptance criteria

- [ ] Taking 5 photos in rapid succession results in exactly one `IndexSyncWorker` run (after the last photo), not five.
- [ ] Taking 1 photo and then no more for 10 seconds results in one index sync.
- [ ] With `index_sync_min_photos = 5`, taking 3 photos and then stopping does **not** trigger a debounced sync; the nightly sync eventually uploads the index.
- [ ] With `index_sync_min_photos = 5`, taking 6 photos in a burst triggers one sync after the burst ends.
- [ ] Manual “Back up index now” still works immediately and resets the pending-change counter.
- [ ] The nightly periodic sync still runs and resets the pending-change counter.
- [ ] When the DB hash is unchanged, no upload occurs and the pending-change counter is still reset.
- [ ] The feature respects `wifi_only_uploads`: on metered networks the debounced work waits until Wi-Fi is available.
- [ ] A failed sync (`Result.retry()`) does not reset the pending-change counter.

## Out of scope

- Differential or incremental index sync.
- A UI setting for the threshold. The spec adds the preference accessor; wire it to `SettingsActivity` only if a later task requires it.
- Conflict resolution for multi-device writes.
- Changing the deduplication source from local DB to remote index.
- Guaranteeing exact 10-second timing under Doze or app-standby.

## Notes

- Because the debouncer reuses the same `ONE_SHOT_NAME` as `runNow()`, a manual tap replaces any pending debounced work and runs immediately. This is the desired behavior.
- E2E scenarios that assert remote index state should either wait for the 10-second debounce window to elapse or continue to call `IndexSyncScheduler.runNow()` for deterministic tests. The debouncer improves real-world behavior but does not remove the need for explicit sync in tests.
- The counter is intentionally persisted in `EncryptedSharedPreferences` rather than held in memory so it survives process death during the debounce window.
