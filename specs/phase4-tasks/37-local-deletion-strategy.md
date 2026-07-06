# Task 37 — Local Deletion Strategy

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Implement the auto-deletion strategy from PRD §4: reclaim local storage on the device after photos and videos have been successfully backed up, batched into a single user-confirmed prompt per day. Strategies: `never` (default), `immediate`, `after_days`, `after_count`. All deletions go through `MediaStore.createDeleteRequest` per Android 11+ scoped storage rules.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/worker/LocalDeleteWorker.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/worker/LocalDeleteScheduler.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/ui/LocalDeleteReviewActivity.kt` (new)
- `app/src/main/res/layout/activity_local_delete_review.xml` (new)
- `app/src/main/res/values/strings.xml` (modify — add this task's keys only)

## `LocalDeleteWorker`

```kotlin
class LocalDeleteWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    companion object {
        const val NOTIFICATION_ID = 5001
    }

    override suspend fun doWork(): Result = withContext(Dispatchers.IO) {
        val app = applicationContext as PhotoBackupApp
        val prefs = app.prefsStore
        val dao = app.uploadDatabase.dao
        val now = System.currentTimeMillis()

        prefs.setLastLocalDeleteRunAt(now)

        val strategy = prefs.getLocalDeleteStrategy()
        if (strategy == "never") return@withContext Result.success()

        val suppressUntil = prefs.getLocalDeleteSuppressUntil()
        if (suppressUntil != null && suppressUntil > now) return@withContext Result.success()

        val candidates = collectCandidates(strategy, prefs, dao, now)
        if (candidates.isEmpty()) return@withContext Result.success()

        // Stage them: mark pending_local_delete = 1 so the activity can re-read consistently.
        dao.markPendingLocalDelete(candidates.map { it.id })

        // Show the daily notification (heads-up + tap → activity).
        showDailyNotification(candidates.size, candidates.sumOf { it.size })

        Result.success()
    }

    private fun collectCandidates(
        strategy: String,
        prefs: PrefsStore,
        dao: UploadDao,
        now: Long
    ): List<UploadRecord> = when (strategy) {
        "immediate" -> dao.getEligibleForLocalDelete(olderThanUploadedAt = now)
        "after_days" -> {
            val cutoff = now - prefs.getLocalDeleteDays() * 86_400_000L
            dao.getEligibleForLocalDelete(olderThanUploadedAt = cutoff)
        }
        "after_count" -> {
            val anchor = prefs.getLastLocalDeleteRunAt() ?: 0L
            val newSince = dao.countUploadsSince(anchor)
            if (newSince < prefs.getLocalDeleteCount()) emptyList()
            else dao.getOldestUploaded(limit = newSince)
        }
        else -> emptyList()
    }
}
```

### Strategy semantics

- **`immediate`**: every uploaded row with `local_present=1 AND pending_local_delete=0` is eligible. The "immediate" name refers to the strategy's *intent* — every upload becomes a deletion candidate at the next daily run. Actual deletion still requires the user's daily confirmation.
- **`after_days`**: rows whose `uploaded_at <= now - N days`. Same filter on `pending_local_delete=0` and `local_present=1`.
- **`after_count`**: count uploads since `last_local_delete_run_at`. If that count meets the configured threshold, mark the oldest `N` uploaded rows for deletion.

### Notification

Channel: `photo_backup` (existing). Importance: heads-up.

```
Title:  "Reclaim space?"
Body:   "X backed-up items ready to delete locally (≈ Y MB)."
Action: tap → LocalDeleteReviewActivity
Dismiss: counts as a dismissal (see below)
```

The notification ID is fixed (`NOTIFICATION_ID = 5001`) so re-runs replace, not stack.

### Dismissal streak

Set a `DeleteIntentReceiver` BroadcastReceiver for notification dismissal. On dismiss:

```kotlin
val streak = prefs.getLocalDeleteDismissStreak() + 1
prefs.setLocalDeleteDismissStreak(streak)
if (streak >= 3) {
    prefs.setLocalDeleteSuppressUntil(System.currentTimeMillis() + 7 * 86_400_000L)
    prefs.setLocalDeleteDismissStreak(0)
}
```

If the user accepts the prompt (the activity issues `createDeleteRequest`), reset `dismissStreak = 0` and clear `suppressUntil`.

## `LocalDeleteScheduler`

```kotlin
object LocalDeleteScheduler {
    private const val PERIODIC_NAME = "local_delete_daily"

    fun schedule(context: Context) {
        val initialDelayMs = msUntilNext(localHour = 19, localMinute = 0)
        val request = PeriodicWorkRequestBuilder<LocalDeleteWorker>(
            repeatInterval = 1, repeatIntervalTimeUnit = TimeUnit.DAYS,
            flexTimeInterval = 1, flexTimeIntervalUnit = TimeUnit.HOURS
        )
            .setInitialDelay(initialDelayMs, TimeUnit.MILLISECONDS)
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
        val request = OneTimeWorkRequestBuilder<LocalDeleteWorker>().build()
        WorkManager.getInstance(context).enqueue(request)
    }

    private fun msUntilNext(localHour: Int, localMinute: Int): Long { /* same shape as IndexSyncScheduler */ }
}
```

- **19:00 local** is the user-facing notification time (PRD §4). Reuse Phase 3's `msUntilNext` shape — or extract to a shared helper if cleaner.
- No network constraint — the worker only reads local DB + sets up the notification.
- The activity is what actually issues `createDeleteRequest`, so the worker can run without UI present.

## `LocalDeleteReviewActivity`

User-facing screen that lists pending rows and dispatches `createDeleteRequest`.

### Layout

`activity_local_delete_review.xml` (sketch):

```xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:id="@+id/header"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="20sp"
        android:textStyle="bold"
        android:text="@string/local_delete_review_title" />

    <TextView
        android:id="@+id/subhead"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="14sp" />

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/list"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="end">
        <Button
            android:id="@+id/cancel"
            android:text="@string/local_delete_cancel"
            style="?android:attr/buttonBarButtonStyle" />
        <Button
            android:id="@+id/confirm"
            android:text="@string/local_delete_confirm"
            style="?android:attr/buttonBarButtonStyle" />
    </LinearLayout>
</LinearLayout>
```

### List items

Each row shows: square thumbnail (reuse Phase 3 Coil + B2 fetcher for cloud thumbnails since the local file is *about to be deleted*; fall back to the local URI thumbnail while the file still exists), filename, size, uploaded-at relative time, and a checkbox (default checked). The user can uncheck items to keep them.

### Flow

```kotlin
class LocalDeleteReviewActivity : AppCompatActivity() {
    private val launcher = registerForActivityResult(
        ActivityResultContracts.StartIntentSenderForResult()
    ) { result ->
        if (result.resultCode == RESULT_OK) onUserApproved()
        else onUserCancelled()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_local_delete_review)
        val app = application as PhotoBackupApp
        val dao = app.uploadDatabase.dao
        val pending = lifecycleScope.async(Dispatchers.IO) { dao.getPendingLocalDelete() }
        lifecycleScope.launch {
            val rows = pending.await()
            renderList(rows)
        }
        findViewById<Button>(R.id.confirm).setOnClickListener { onConfirm() }
        findViewById<Button>(R.id.cancel).setOnClickListener { finish() }
    }

    private fun onConfirm() {
        val selected = getSelectedRows()
        if (selected.isEmpty()) { finish(); return }
        val uris = selected.map { Uri.parse(it.localUri) }
        val intentSender = MediaStore.createDeleteRequest(contentResolver, uris).intentSender
        launcher.launch(IntentSenderRequest.Builder(intentSender).build())
        pendingSelected = selected
    }

    private fun onUserApproved() {
        val app = application as PhotoBackupApp
        val dao = app.uploadDatabase.dao
        lifecycleScope.launch(Dispatchers.IO) {
            pendingSelected.forEach { row ->
                dao.setLocalPresent(row.id, false)
                dao.clearPendingLocalDelete(row.id)
            }
            app.prefsStore.setLocalDeleteDismissStreak(0)
            app.prefsStore.setLocalDeleteSuppressUntil(null)
            app.galleryRepository.invalidate()
        }
        finish()
    }

    private fun onUserCancelled() {
        // Leave pending_local_delete = 1 — the next worker run will re-prompt.
        finish()
    }
}
```

### Items the user unchecked

For each unchecked row before confirmation: clear `pending_local_delete` so the worker does not re-stage it the next day. The user has explicitly opted out for this batch.

### Items missing from MediaStore at confirm time

If the system delete request silently drops items (e.g., the file was already removed manually), the activity does not have to do anything — `createDeleteRequest` accepts the absence. The DB rows still get `local_present = false` since the goal state (file gone) is achieved.

## Strings

```xml
<string name="local_delete_review_title">Free up space</string>
<string name="local_delete_review_subhead">%1$d items, %2$s</string>
<string name="local_delete_confirm">Delete selected</string>
<string name="local_delete_cancel">Cancel</string>
<string name="local_delete_notif_title">Reclaim space?</string>
<string name="local_delete_notif_body">%1$d backed-up items ready to delete locally (≈ %2$s).</string>
```

## Implementation notes

- The worker runs even when `strategy = never` (it still calls `setLastLocalDeleteRunAt`, which is harmless), but exits early. Skipping the entire enqueue is a future optimization.
- The dismissal streak only counts notification dismissals (swipe-away), not "Cancel" clicks inside the review activity. Cancel inside the activity is treated as "show me again tomorrow" without streak increment.
- After the user confirms, the deletion sets `local_present = false` and the row stays in the DB so Cloud view continues to show it. The Phase 3 `cloud_deleted_at` semantics are unrelated and untouched.
- When the strategy changes via Settings (Task 41), no immediate re-stage is required — the next 19:00 worker run picks up the new strategy. Manual `runNow` is exposed for users who want immediate effect.

## Constraints

- Android scoped storage (min SDK 33 — well past Android 11) requires user confirmation per delete batch. `createDeleteRequest` is the only legal path.
- Never delete files outside `pending_local_delete = 1 AND status = 'uploaded'`. A file that failed to upload must never be auto-deleted.
- Never delete the upload row from `uploads` here — only flip `local_present`.
- Notification importance: heads-up but not full-screen. Use the existing `photo_backup` channel.

## Dependencies (by interface)

- `PrefsStore` (Task 32) — every local-delete preference + the dismiss streak
- `UploadDao` (Task 31) — `markPendingLocalDelete`, `getPendingLocalDelete`, `getEligibleForLocalDelete`, `getOldestUploaded`, `clearPendingLocalDelete`, `setLocalPresent`, `countUploadsSince`
- `GalleryRepository` (Phase 3 Task 23) — `invalidate` after a successful confirmation
- WorkManager (`androidx.work:work-runtime-ktx`)
- `MediaStore.createDeleteRequest` (platform API)

## Acceptance criteria

- [ ] With `strategy = never`, the worker is a no-op (no notification, no DB writes beyond `lastLocalDeleteRunAt`).
- [ ] With `strategy = immediate`, the worker stages every eligible uploaded row and posts the notification when count > 0.
- [ ] With `strategy = after_days` and `local_delete_days = 30`, only rows with `uploaded_at <= now - 30d` are staged.
- [ ] With `strategy = after_count` and `local_delete_count = 100`, the worker does nothing until at least 100 uploads have happened since the last run, then stages the oldest 100.
- [ ] Tapping the notification opens `LocalDeleteReviewActivity` showing the staged rows.
- [ ] Confirming with all rows checked triggers a single system delete prompt for all URIs.
- [ ] After user approval, `local_present = 0` for the deleted rows and they appear in Cloud view but not Local view (Phase 3 semantics).
- [ ] Dismissing the notification three days in a row sets `suppress_until = now + 7 days` and resets the streak.
- [ ] Confirming a deletion clears both the streak and the suppression window.
- [ ] Uncheck-then-confirm clears `pending_local_delete` on the unchecked rows (no re-stage tomorrow).

## Out of scope

- Configurable prompt time (PRD Open Question #6 — defer).
- Bulk un-stage UI ("undo all").
- Permanent ignore-this-file marking.
- Auto-deletion on a different device (cross-device sync is out of scope).
- Deletion of cloud copies — that is Phase 3 Cloud-view delete, untouched.
