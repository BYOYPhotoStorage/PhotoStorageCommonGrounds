# Task 40 — First-Backup Flow

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Replace the Phase 2 "Enable automatic backup?" modal with a three-step onboarding tail that asks the user (a) whether to enable auto-backup, (b) whether to back up from today only or the entire gallery history, and (c) whether to include videos. Implements PRD §1 verbatim. Also ships the `InitialBackfillWorker` that walks MediaStore once when the user picks "Entire gallery history".

The activity runs immediately after Phase 3's `IndexRecoveryActivity` (or after onboarding when no remote index exists). Task 42 wires the routing.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/ui/FirstBackupActivity.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/worker/InitialBackfillWorker.kt` (new)
- `app/src/main/res/layout/activity_first_backup.xml` (new)
- `app/src/main/res/values/strings.xml` (modify — add this task's keys only)

## `FirstBackupActivity` flow

Three steps in sequence. Each step is a full-screen view inside the same activity (no nested fragments), driven by a small `State` enum.

```kotlin
enum class Step { ENABLE, SCOPE, VIDEOS, DONE }

class FirstBackupActivity : AppCompatActivity() {
    private var step: Step = Step.ENABLE
    private var autoUploadChoice: Boolean? = null
    private var scopeChoice: String = "today"
    private var videosChoice: Boolean = false
}
```

### Step A — Enable

- Title: `R.string.first_backup_enable_title` ("Back up new photos automatically?")
- Body: `R.string.first_backup_enable_body` ("We'll watch your gallery and upload new photos in the background.")
- Buttons: "Enable" (primary) / "Not now"
- On "Enable": `autoUploadChoice = true`, advance to `SCOPE`
- On "Not now": `autoUploadChoice = false`, jump to `DONE` (skip Steps B and C)

### Step B — Scope

- Title: `R.string.first_backup_scope_title` ("What should we back up?")
- Body: `R.string.first_backup_scope_body` ("Just photos taken from now on, or your entire gallery history?")
- Radio group:
  - "From today only" (default, value `today`)
  - "Entire gallery history" (value `all`)
- Button: "Continue"
- On "Continue": `scopeChoice = selected`, advance to `VIDEOS`

### Step C — Videos

- Title: `R.string.first_backup_videos_title` ("Back up videos too?")
- Body: `R.string.first_backup_videos_body` ("Videos can use much more space than photos. You can change this in settings later.")
- Toggle switch, default off
- Button: "Finish"
- On "Finish": `videosChoice = toggleState`, advance to `DONE`

If `videosChoice` becomes `true`, request `Manifest.permission.READ_MEDIA_VIDEO` via the runtime permission API. If the user denies, leave `videosChoice = false` and proceed — Task 34's scanner already returns empty if the permission is missing, so this is safe. Show a toast: "Videos disabled — permission denied."

### Step DONE — commit and exit

```kotlin
private fun commitAndFinish() {
    val app = application as PhotoBackupApp
    val prefs = app.prefsStore

    val enabled = autoUploadChoice == true

    prefs.setAutoUploadEnabled(enabled)      // existing Phase 2 setter
    prefs.setFirstBackupScope(scopeChoice)
    prefs.setVideosEnabled(videosChoice)
    prefs.setFirstBackupFlowCompleted(true)

    if (enabled && scopeChoice == "all") {
        WorkManager.getInstance(this).enqueueUniqueWork(
            "initial_backfill",
            ExistingWorkPolicy.KEEP,
            OneTimeWorkRequestBuilder<InitialBackfillWorker>()
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(
                            if (prefs.isWifiOnly()) NetworkType.UNMETERED else NetworkType.CONNECTED
                        )
                        .build()
                )
                .build()
        )
    }

    if (enabled) {
        UploadForegroundService.start(this) // existing Phase 2 entrypoint
    }

    finish()
}
```

Routing back to the gallery is owned by Task 42's `MainActivity` — `FirstBackupActivity` only `finish()`es.

## Layout

`activity_first_backup.xml` is a single LinearLayout with TextView + content area + buttons. Implementation can re-inflate or switch visibility per `Step`:

```xml
<LinearLayout android:orientation="vertical" android:padding="24dp" android:gravity="center">
  <TextView android:id="@+id/title"  android:textSize="22sp" android:textStyle="bold"/>
  <TextView android:id="@+id/body"   android:paddingTop="16dp" android:textSize="15sp"/>

  <FrameLayout android:id="@+id/content_slot"
      android:layout_width="match_parent" android:layout_height="wrap_content"
      android:paddingTop="24dp"/>

  <LinearLayout android:orientation="horizontal" android:gravity="end" android:paddingTop="24dp">
    <Button android:id="@+id/btn_secondary" style="?android:attr/buttonBarButtonStyle"/>
    <Button android:id="@+id/btn_primary"   style="?android:attr/buttonBarButtonStyle"/>
  </LinearLayout>
</LinearLayout>
```

`content_slot` is empty on Step A, hosts a `RadioGroup` on Step B, and a `Switch` on Step C. Inflate the children programmatically; do not create three layouts.

## `InitialBackfillWorker`

One-shot worker, runs once and self-deletes via the `OneTimeWorkRequest` lifecycle.

```kotlin
class InitialBackfillWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result = withContext(Dispatchers.IO) {
        val app = applicationContext as PhotoBackupApp
        val prefs = app.prefsStore
        val dao = app.uploadDatabase.dao
        val scanner = MediaStoreScanner(applicationContext)

        // Photos: always
        val photos = scanner.scanImages(since = null)
        photos.forEach { enqueueIfNew(it, dao) }

        // Videos: only if explicitly enabled — and only forward (PRD constraint #11).
        // The full backfill does NOT include videos even when videos_enabled=true at backfill time.
        // Rationale: a full historical video upload is a much bigger commitment than the user implicitly
        // agreed to with a single yes/no in onboarding. Videos start backing up forward from "now".

        prefs.setLastScanTimestamp(System.currentTimeMillis())
        Result.success()
    }

    private fun enqueueIfNew(item: MediaItem, dao: UploadDao) {
        // Reuse Phase 2 dedup helper.
        // ...filename+size+dateTaken check, insert if new.
    }
}
```

### Cancellation

If the user toggles auto-upload off mid-backfill (via Settings), the worker continues to completion — the rows just sit in the queue waiting for `UploadModeGate` to allow upload. This is acceptable for a one-shot worker and avoids fragile mid-run state.

### Progress

The worker does not surface progress to the UI. The existing foreground notification already shows queue depth; the queue depth jumping to thousands after backfill is the user's confirmation that the worker ran. A future iteration could add a dedicated progress notification.

### Constraints

Wi-Fi-only is honored at the WorkManager constraint level (the activity sets `NetworkType.UNMETERED` when `isWifiOnly()` is true). The worker itself does not re-check.

## Strings

```xml
<string name="first_backup_enable_title">Back up new photos automatically?</string>
<string name="first_backup_enable_body">We\'ll watch your gallery and upload new photos in the background.</string>
<string name="first_backup_enable_primary">Enable</string>
<string name="first_backup_enable_secondary">Not now</string>

<string name="first_backup_scope_title">What should we back up?</string>
<string name="first_backup_scope_body">Just photos taken from now on, or your entire gallery history?</string>
<string name="first_backup_scope_today">From today only</string>
<string name="first_backup_scope_all">Entire gallery history</string>
<string name="first_backup_scope_continue">Continue</string>

<string name="first_backup_videos_title">Back up videos too?</string>
<string name="first_backup_videos_body">Videos can use much more space than photos. You can change this in settings later.</string>
<string name="first_backup_videos_toggle">Include videos</string>
<string name="first_backup_videos_finish">Finish</string>
<string name="first_backup_videos_permission_denied">Videos disabled — permission denied.</string>
```

## Implementation notes

- `setAutoUploadEnabled` is the existing Phase 2 PrefsStore method. If the actual signature differs (`setAutoUpload(...)` etc.), use the existing one — Task 40 must not modify `PrefsStore` (that is Task 32's file).
- The runtime permission request for `READ_MEDIA_VIDEO` uses `ActivityResultContracts.RequestPermission`. The manifest declaration is added by Task 42.
- Step transitions should not animate (no `Fragment` transactions). Simple `View.GONE` / `View.VISIBLE` swapping inside `content_slot` keeps the activity lightweight and avoids back-stack quirks.
- Pressing the system back button at Step A finishes without committing (caller decides what to do next — Task 42's MainActivity treats "not completed" as "ask again on next launch"). At Step B/C, back returns to the previous step in-activity, not the previous activity.
- The `InitialBackfillWorker` uses `ExistingWorkPolicy.KEEP` — if for any reason the user goes through the flow twice (e.g., wipes app data and reinstalls), a still-running backfill is not interrupted.
- Idempotent: enqueueing photos that are already in `uploads` is a no-op thanks to the Phase 2 dedup helper.

## Constraints

- Pre-Phase-4 users (Phase 2/3 testers) must not see this screen. Task 32's `hasCompletedFirstBackupFlow()` defaults to `false`, but Task 42's MainActivity treats "credentials exist AND auto_upload_enabled" as "already onboarded; mark flow completed and skip". See Task 42 for the upgrade-path logic.
- `first_backup_scope = all` only backfills photos, never videos (PRD constraint #11 / open question #8). Document this in code with a one-line comment in `InitialBackfillWorker`.
- The activity must not start the upload foreground service before `commitAndFinish` — preferences must be persisted first.

## Dependencies (by interface)

- `PrefsStore` (Task 32) — `setAutoUploadEnabled` (Phase 2), `setFirstBackupScope`, `setVideosEnabled`, `setFirstBackupFlowCompleted`, `isWifiOnly`
- `MediaStoreScanner` (Task 34) — `scanImages` (used by `InitialBackfillWorker`)
- `UploadDao` (Task 31) — `insert` (with `media_type`)
- `UploadForegroundService` (Phase 2) — `start(context)` helper
- WorkManager (`androidx.work:work-runtime-ktx`)

## Acceptance criteria

- [ ] On first run after credentials are validated (and no remote index exists), `FirstBackupActivity` opens at Step A.
- [ ] Choosing "Not now" at Step A finishes the activity, sets `auto_upload_enabled = false`, `first_backup_scope = today` (untouched default), `videos_enabled = false`, and marks `first_backup_flow_completed = true`.
- [ ] Choosing "Enable" advances to Step B.
- [ ] Choosing "From today only" + "Continue" advances to Step C without enqueueing any photos.
- [ ] Choosing "Entire gallery history" + "Continue" advances to Step C and, on Finish, the `InitialBackfillWorker` runs and enqueues every MediaStore image.
- [ ] Toggling videos on at Step C requests `READ_MEDIA_VIDEO`; denying it leaves `videos_enabled = false`.
- [ ] On Finish, the upload foreground service starts (if enabled) and the gallery opens.
- [ ] `InitialBackfillWorker` enqueues photos but NOT videos, even when `videos_enabled = true`.
- [ ] Pressing back at Step B/C navigates within the activity (no premature finish).

## Out of scope

- Allowing the user to re-run this flow from settings (a future iteration could expose "Re-enter first-backup choices" but Phase 4 does not).
- A progress indicator for the backfill worker.
- Choosing a starting date other than "today" or "all" (no custom date picker in Phase 4).
- Backfilling videos historically when toggled on later.
- Skipping the flow with a "Use defaults" shortcut.
