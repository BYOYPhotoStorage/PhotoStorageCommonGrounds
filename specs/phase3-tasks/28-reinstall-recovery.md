# Task 28 — Reinstall Recovery Flow

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

After the existing onboarding step that validates B2 credentials, check whether an index exists in B2. If yes, offer to restore it; if no (or the user declines), proceed with an empty index. After restore-or-skip, ask the user how to catch up — three options per PRD §4. The flow also wires a "Restore index from cloud" entry point reachable from settings later.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/recovery/IndexRecoveryActivity.kt`
- `app/src/main/java/com/hriyaan/photostorage/recovery/IndexRecoveryService.kt`
- `app/src/main/res/layout/activity_index_recovery.xml`
- `app/src/main/res/values/strings.xml` (modify — recovery strings)

## `IndexRecoveryService` contract

```kotlin
class IndexRecoveryService(
    private val context: Context,
    private val s3Uploader: S3Uploader,
    private val uploadDao: UploadDao,
    private val uploadDatabase: UploadDatabase,
    private val prefsStore: PrefsStore
) {
    /** Returns null when no index exists in B2, or info about the remote index when found. */
    suspend fun hasRemoteIndex(): Result<RemoteIndexInfo?>

    /** Downloads, validates, migrates if needed, and swaps in the remote index. */
    suspend fun downloadAndRestore(progress: (Float) -> Unit = {}): Result<RestoreOutcome>

    /** After restore, iterate uploaded rows and set localPresent based on the current MediaStore. */
    suspend fun reconcileLocalPresent()
}

data class RemoteIndexInfo(val lastModified: Long?) // may be null if HEAD does not surface it
data class RestoreOutcome(val photoCount: Int, val latestUploadedAt: Long?)
```

### `hasRemoteIndex()`

```kotlin
val exists = s3Uploader.headObject(IndexSyncWorker.INDEX_B2_PATH).getOrElse { return Result.failure(it) }
return if (exists) Result.success(RemoteIndexInfo(lastModified = null)) else Result.success(null)
```

(If the S3 client surfaces `Last-Modified` cheaply, pass it through; otherwise leave null.)

### `downloadAndRestore(progress)`

```kotlin
1. Download to a temp file: temp = File(cacheDir, "restore-${ts}.sqlite")
   - s3Uploader.downloadObject(INDEX_B2_PATH, temp).getOrThrow()
   - Wire progress through a wrapping FileOutputStream if the existing client doesn't already expose it
2. Validate as SQLite:
   - Open with SQLiteDatabase.openDatabase(temp.absolutePath, null, OPEN_READONLY)
   - Run: SELECT name FROM sqlite_master WHERE type='table' AND name='uploads'
   - If absent → throw IllegalStateException("Index is missing the uploads table")
3. Read user_version:
   - val version = db.version
   - If version > current DB_VERSION → return Result.failure with a "Please update the app" message
   - If version < current DB_VERSION → run the same onUpgrade migration path against the temp DB
4. Close the temp DB
5. Atomic swap:
   - Synchronize on a recovery mutex held by UploadDatabase
   - uploadDatabase.close()
   - val dest = context.getDatabasePath("uploads.db")
   - Files.move(temp.toPath(), dest.toPath(), StandardCopyOption.REPLACE_EXISTING)
     (or use kotlin.io.File.copyTo then temp.delete())
   - uploadDatabase.reopen()
6. Compute outcome: photoCount = uploadDao.getCloudView().size, latestUploadedAt = uploadDao.getLatestUploadedAt()
7. Reset sync prefs:
   - prefsStore.setLastSyncedIndexHash(null)   // forces next sync to upload (the restored DB will likely diverge as we reconcile)
8. Return Result.success(RestoreOutcome(...))
```

### `reconcileLocalPresent()`

Walks every uploaded row in the restored DB and updates `local_present` based on whether a matching photo exists in the current MediaStore. Implements the PRD's time-based merge semantics: photos older than the restored index's `latestUploadedAt` are typically not on this device (so `local_present` defaults to false), but if any of them happen to be present locally (e.g., the user kept a backup of the gallery folder), we surface them as `Synced`.

```kotlin
suspend fun reconcileLocalPresent() = withContext(Dispatchers.IO) {
    val records = uploadDao.getCloudView()
    if (records.isEmpty()) return@withContext

    // Build a fast lookup keyed by sha256 OR (filename,size,dateTaken)
    data class Key(val filename: String, val size: Long, val dateTaken: Long)
    val recordsByKey = records.associateBy { Key(it.filename, it.size, it.dateTaken) }
    val recordsBySha = records.filter { it.sha256 != null }.associateBy { it.sha256!! }

    // Single MediaStore pass
    val projection = arrayOf(
        MediaStore.Images.Media._ID,
        MediaStore.Images.Media.DISPLAY_NAME,
        MediaStore.Images.Media.SIZE,
        MediaStore.Images.Media.DATE_TAKEN
    )
    val cursor = context.contentResolver.query(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI, projection, null, null, null
    ) ?: return@withContext

    val foundIds = mutableSetOf<Long>()
    cursor.use { c ->
        val nameIdx = c.getColumnIndexOrThrow(MediaStore.Images.Media.DISPLAY_NAME)
        val sizeIdx = c.getColumnIndexOrThrow(MediaStore.Images.Media.SIZE)
        val dateIdx = c.getColumnIndexOrThrow(MediaStore.Images.Media.DATE_TAKEN)
        while (c.moveToNext()) {
            val key = Key(c.getString(nameIdx), c.getLong(sizeIdx), c.getLong(dateIdx))
            val match = recordsByKey[key]
            if (match != null) foundIds += match.id
            // SHA-256 match path is opt-in — computing on every MediaStore row is expensive.
            // Defer SHA-based reconciliation to the Phase 2 dedup pipeline triggered on upload.
        }
    }

    val notFound = records.map { it.id }.toSet() - foundIds
    for (id in foundIds) uploadDao.setLocalPresent(id, true)
    for (id in notFound) uploadDao.setLocalPresent(id, false)
}
```

The natural-key check is cheap. SHA-256 reconciliation is intentionally NOT run here — it would require reading every local file's bytes (10K hashes is too slow at install time). The Phase 2 `DuplicateDetector` already covers SHA matching when photos go through the upload path, so any later `ContentObserver` fire will reconcile through that route.

## `IndexRecoveryActivity` flow

Started from `MainActivity` (Task 30) after credentials are validated and before the gallery is shown.

### States

```kotlin
sealed class State {
    object Checking : State()
    object NoRemote : State()                          // proceed straight to CatchUpPrompt(latestUploadedAt = null)
    data class Found(val info: RemoteIndexInfo) : State()
    data class Restoring(val progress: Float) : State()
    data class Restored(val outcome: RestoreOutcome) : State()
    data class Failed(val message: String) : State()
    data class CatchUpPrompt(val latestUploadedAt: Long?) : State()
}
```

### UI

A single full-screen scrollable column with:
- Progress indicator (visible during Checking / Restoring)
- Title + body text (changes per state)
- Up to two buttons

State → UI:

- **Checking**: Title "Checking for backup index…", spinner
- **NoRemote**: Skip to CatchUpPrompt(null) automatically (no UI shown for this state)
- **Found**: Title "Restore your previous backup index?", body "We found an index from [info.lastModified or 'a previous install']. Restoring lets you see your existing cloud photos without rescanning.", buttons "Restore" / "Start fresh"
- **Restoring**: Title "Restoring backup index…", linear progress
- **Restored**: Title "Restored", body "Restored {outcome.photoCount} photos from your previous backup.", auto-advance to CatchUpPrompt(outcome.latestUploadedAt) after 1.5 seconds
- **Failed**: Title "We could not restore the index.", body "{message} You can start fresh and the app will rescan your gallery.", button "Start fresh"
- **CatchUpPrompt**:
  - Title "What should we back up?"
  - Body, when latestUploadedAt != null: "Your index has photos up to {format(latestUploadedAt)}."
  - Body, when null: "We will scan your gallery and back up everything not already in your cloud."
  - Three buttons stacked vertically:
    - **"Catch up from this date"** (primary, default when latestUploadedAt != null)
    - **"Full rescan"**
    - **"Skip for now"**

### Catch-up dispatch

After the user picks an option:

- **Catch up**: enqueue a one-shot `NightlyScanWorker` (Phase 2 Task 16) with a parameter `since = latestUploadedAt`. If `NightlyScanWorker` does not currently accept a `since` parameter, add it as a `Data` input — the worker uses `since ?: prefsStore.getLastScanTimestamp()`. (Coordinate with Task 16 owner if a small change is needed there.)
- **Full rescan**: enqueue a one-shot `NightlyScanWorker` with no `since` (whole-gallery scan).
- **Skip**: do nothing.

In all three cases:
1. `IndexRecoveryService.reconcileLocalPresent()`
2. Finish the activity and route to `GalleryActivity`

### Manual re-entry from settings

`IndexRecoveryActivity` accepts an extra `FORCE_RESTORE_PROMPT = true`. When set, it shows the Found / NoRemote flow even if the user already had a local index. Before swapping, show an additional confirmation: "This will replace your current index. Continue?" — Restore / Cancel.

## Layout sketch (`activity_index_recovery.xml`)

```xml
<LinearLayout
    android:orientation="vertical"
    android:padding="24dp"
    android:gravity="center"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <ProgressBar
        android:id="@+id/progress"
        style="?android:attr/progressBarStyleHorizontal"
        android:visibility="gone"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="20sp"
        android:textStyle="bold"
        android:paddingTop="16dp" />

    <TextView
        android:id="@+id/body"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:paddingTop="12dp"
        android:textSize="15sp" />

    <LinearLayout
        android:id="@+id/buttons"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:paddingTop="24dp">
        <!-- buttons inflated at runtime -->
    </LinearLayout>
</LinearLayout>
```

## New strings

```xml
<string name="recovery_checking">Checking for backup index…</string>
<string name="recovery_found_title">Restore your previous backup index?</string>
<string name="recovery_found_body">We found an index from %1$s. Restoring lets you see your existing cloud photos without rescanning.</string>
<string name="recovery_restoring">Restoring backup index…</string>
<string name="recovery_restored_title">Restored</string>
<string name="recovery_restored_body">Restored %1$d photos from your previous backup.</string>
<string name="recovery_failed_title">We could not restore the index</string>
<string name="recovery_failed_body">%1$s You can start fresh and the app will rescan your gallery.</string>
<string name="recovery_restore_button">Restore</string>
<string name="recovery_start_fresh_button">Start fresh</string>
<string name="catchup_title">What should we back up?</string>
<string name="catchup_body_with_date">Your index has photos up to %1$s.</string>
<string name="catchup_body_no_date">We will scan your gallery and back up everything not already in your cloud.</string>
<string name="catchup_from_date">Catch up from this date</string>
<string name="catchup_full">Full rescan</string>
<string name="catchup_skip">Skip for now</string>
<string name="recovery_replace_confirm">This will replace your current index. Continue?</string>
```

## Constraints

- The DB swap must be atomic from the perspective of the rest of the app. Other components must not hold an open DB handle during the swap.
- After restore, `prefsStore.setLastSyncedIndexHash(null)` so the first nightly sync will re-upload (the restored DB might be modified by `reconcileLocalPresent` immediately after).
- The catch-up prompt is shown **every time** the recovery flow runs — do not persist the user's choice across reinstalls (PRD constraint #7).
- `reconcileLocalPresent` uses natural-key matching only; do not compute SHA-256 for every MediaStore row at install time.
- If the downloaded index has a newer schema version than the app supports, refuse with a "please update the app" message rather than attempting destructive fallback.

## Dependencies (by interface)

- `S3Uploader` (Task 22) — `headObject`, `downloadObject`
- `UploadDao` and `UploadDatabase` (Task 20) — read access for outcome computation, write access for `replaceAll` (via swap) and `setLocalPresent`
- `PrefsStore` (Task 21) — clear `last_synced_index_hash`, read `last_scan_timestamp`
- `NightlyScanWorker` (Phase 2 Task 16) — one-shot dispatch with `since` parameter
- Existing onboarding flow — `MainActivity` routes here when credentials are present but `last_index_sync_at` is null AND the activity hasn't been completed yet (Task 30 wires the routing)

## Acceptance criteria

- [ ] Fresh install + no remote index → activity finishes immediately into CatchUpPrompt(null), three buttons visible.
- [ ] Fresh install + remote index exists → "Restore your previous backup index?" prompt shown; "Restore" downloads and swaps the local DB.
- [ ] After restore, `getCloudView().size` matches the restored row count.
- [ ] `reconcileLocalPresent` correctly sets `local_present = true` for rows whose (filename, size, dateTaken) exists in MediaStore.
- [ ] Choosing "Catch up from this date" enqueues a one-shot scan with the correct `since` timestamp.
- [ ] Choosing "Full rescan" enqueues a one-shot scan with no `since`.
- [ ] Choosing "Skip for now" finishes without enqueueing scans.
- [ ] Manual re-entry (settings → "Restore index from cloud") shows the replace-confirmation step.
- [ ] If the remote index has a newer schema than the app, the activity surfaces a non-destructive error.

## Out of scope

- Multi-source recovery (different B2 bucket, manual file picker).
- Snapshot history of the SQLite index in B2 (last-write-wins is acceptable).
- Background restore — Phase 3 restore is foreground/visible.
- Restore that preserves the local DB's `uploads` rows that aren't in the remote index (the swap is wholesale).
