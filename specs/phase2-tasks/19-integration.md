# Task 19 — Integration & Wire-up

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Glue all Phase 2 components together, update `PhotoBackupApp` and `MainActivity` to wire singletons, add the WorkManager dependency, and run the end-to-end smoke test.

## Files you own (touch)

- `app/src/main/java/com/hriyaan/photostorage/PhotoBackupApp.kt` (modify — add notification channels)
- `app/src/main/java/com/hriyaan/photostorage/MainActivity.kt` (modify — auto-upload routing)
- `app/build.gradle.kts` (modify — add WorkManager dependency)

You should **not** modify files owned by Tasks 10–18. If a wiring problem requires a contract change, raise it back to the owning task.

## `build.gradle.kts` changes

Add to the `dependencies` block:

```kotlin
implementation("androidx.work:work-runtime-ktx:2.9.1")
```

## `PhotoBackupApp`

Update `onCreate` to initialize notification channels:

```kotlin
override fun onCreate() {
    super.onCreate()
    prefsStore = PrefsStore(this)
    uploadDatabase = UploadDatabase(this)
    UploadNotificationManager(this).ensureChannels()
}
```

## `MainActivity`

Update routing to check auto-upload state after credentials:

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val app = application as PhotoBackupApp

        if (!app.prefsStore.hasCredentials()) {
            startActivity(Intent(this, OnboardingActivity::class.java))
            finish()
            return
        }

        // Credentials exist — route to gallery
        startActivity(Intent(this, GalleryActivity::class.java))
        finish()
    }
}
```

> The actual auto-upload service start happens in `GalleryActivity.onCreate` (Task 18), not here. `MainActivity` remains a simple routing activity.

## Verification: end-to-end smoke test

Test against a **real B2 bucket** with a real device.

### Pre-test setup

1. Install the Phase 2 build over the MVP build (do not uninstall — verify DB migration works).
2. Verify the MVP data is intact: open the app, previously uploaded photos still show cloud icons.

### Smoke checklist

| # | Step | Expected |
|---|------|----------|
| 1 | Launch app → gallery loads | Existing uploads show cloud icons |
| 2 | Toggle auto-upload ON | Persistent notification appears |
| 3 | Take a photo with the camera app | Within 30 seconds, the photo appears in B2 (photo + thumbnail) |
| 4 | Check the gallery | The new photo shows a cloud icon |
| 5 | Kill the app, take another photo | Relaunch the app → the photo was enqueued and uploaded while the service was running |
| 6 | Toggle airplane mode, take a photo | Photo is enqueued but not uploaded. Toggle airplane mode off → upload resumes |
| 7 | Toggle Wi-Fi-only ON, switch to mobile data | Take a photo → enqueued but not uploaded. Switch to Wi-Fi → upload starts |
| 8 | Force a failed upload (bad credentials) | After 5 retries, notification shows "Backup credentials expired" |
| 9 | Reboot the device with auto-upload ON | Service restarts automatically |
| 10 | Toggle auto-upload OFF, reboot | Service does NOT start |

### DB migration check

```bash
adb shell run-as com.hriyaan.photostorage sqlite3 databases/uploads.db ".schema uploads"
```

Verify the output shows all 4 new columns (`retry_count`, `next_retry_at`, `sha256`, `created_at`).

## Constraints

- Do not introduce new dependencies beyond WorkManager.
- Do not rewrite component implementations.
- `MainActivity` must remain a lightweight routing activity — no business logic.

## Acceptance criteria

- [ ] `PhotoBackupApp.onCreate` calls `ensureChannels()` before any notification is shown.
- [ ] `./gradlew assembleDebug` succeeds with the new WorkManager dependency.
- [ ] All 10 smoke-checklist items pass on a real device.
- [ ] DB migration from v1 → v2 succeeds without data loss.
- [ ] MVP functionality (manual tap-to-upload) still works when auto-upload is disabled.
- [ ] `./gradlew assembleDebug lint` is clean.

## Out of scope

- CI configuration changes.
- ProGuard / R8 rules for WorkManager.
- Firebase App Distribution release build.
