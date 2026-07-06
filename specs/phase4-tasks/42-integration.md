# Task 42 — Integration & Wire-up

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Glue all Phase 4 components into the application: instantiate singletons in `PhotoBackupApp`, route `MainActivity` through `FirstBackupActivity` on first launch after onboarding, schedule the new periodic worker, add the AndroidX Media3 dependency, declare new activities + the video permission in the manifest, and run the end-to-end smoke test.

This task does not implement features — it wires them.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/PhotoBackupApp.kt` (modify)
- `app/src/main/java/com/hriyaan/photostorage/MainActivity.kt` (modify)
- `app/build.gradle.kts` (modify — adds AndroidX Media3)
- `app/src/main/AndroidManifest.xml` (modify — declares new activities + permission)

You should **not** modify files owned by Tasks 31–41. If a wiring problem requires a contract change, raise it back to the owning task.

## `build.gradle.kts` changes

Add to the `dependencies` block:

```kotlin
implementation("androidx.media3:media3-transformer:1.5.0")
implementation("androidx.media3:media3-effect:1.5.0")
implementation("androidx.media3:media3-common:1.5.0")
```

No other dependencies are added. WorkManager (Phase 2), Coil (Phase 3), and the existing S3 client are unchanged.

> The exact Media3 version is whatever is current and stable at Phase 4 ship time — `1.5.0` is a placeholder. Pin all three to the same version; mixing versions causes runtime failures in `Transformer`.

## `PhotoBackupApp`

Update `onCreate` to construct Phase 4 singletons and schedule the new periodic worker. Keep all Phase 2 and Phase 3 wiring intact.

```kotlin
class PhotoBackupApp : Application() {
    // Phase 2
    lateinit var prefsStore: PrefsStore
    lateinit var uploadDatabase: UploadDatabase
    lateinit var s3Uploader: S3Uploader

    // Phase 3
    lateinit var galleryRepository: GalleryRepository
    lateinit var deletionEngine: DeletionEngine
    lateinit var thumbnailCacheFactory: ThumbnailCacheFactory
    lateinit var indexRecoveryService: IndexRecoveryService

    // Phase 4
    lateinit var shareLinkDao: ShareLinkDao
    lateinit var shareLinkService: ShareLinkService

    override fun onCreate() {
        super.onCreate()

        prefsStore = PrefsStore(this)
        uploadDatabase = UploadDatabase(this)
        s3Uploader = S3Uploader(/* existing wiring */)
        UploadNotificationManager(this).ensureChannels()

        galleryRepository = GalleryRepository(this, uploadDatabase.dao, prefsStore)
        deletionEngine = DeletionEngine(this, uploadDatabase.dao, s3Uploader, galleryRepository)
        thumbnailCacheFactory = ThumbnailCacheFactory(
            context = this,
            s3Uploader = s3Uploader,
            prefsStore = prefsStore,
            egressRecorder = { bytes ->
                prefsStore.setEgressBytesMonth(prefsStore.getEgressBytesMonth() + bytes)
            }
        )
        indexRecoveryService = IndexRecoveryService(
            this, s3Uploader, uploadDatabase.dao, uploadDatabase, prefsStore
        )

        shareLinkDao = ShareLinkDao(uploadDatabase)
        shareLinkService = ShareLinkService(s3Uploader, shareLinkDao)

        IndexSyncScheduler.schedule(this)
        SoftDeleteCleanupScheduler.schedule(this)
        LocalDeleteScheduler.schedule(this) // Phase 4

        // Pre-Phase-4 upgrade path: users who completed Phase 2/3 onboarding never see FirstBackupActivity.
        if (prefsStore.hasCredentials() && prefsStore.isAutoUploadEnabled() &&
            !prefsStore.hasCompletedFirstBackupFlow()) {
            prefsStore.setFirstBackupScope("today")
            prefsStore.setVideosEnabled(false)
            prefsStore.setFirstBackupFlowCompleted(true)
        }
    }
}
```

> The `egressRecorder` hook on `ThumbnailCacheFactory` is the one Phase 3 contract extension Phase 4 introduces. Task 38 makes the one-line constructor edit; this task uses the new parameter.

## `MainActivity`

Update routing to insert `FirstBackupActivity` between recovery and the gallery:

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val app = application as PhotoBackupApp
        val prefs = app.prefsStore

        when {
            !prefs.hasCredentials() -> {
                startActivity(Intent(this, OnboardingActivity::class.java))
            }
            // Phase 3 recovery still gates first (when there is no local index yet).
            prefs.getLastIndexSyncAt() == null && !prefs.hasCompletedRecoveryFlow() -> {
                startActivity(Intent(this, IndexRecoveryActivity::class.java))
            }
            // Phase 4: ask the first-backup questions before the gallery.
            !prefs.hasCompletedFirstBackupFlow() -> {
                startActivity(Intent(this, FirstBackupActivity::class.java))
            }
            else -> {
                startActivity(Intent(this, GalleryActivity::class.java))
            }
        }
        finish()
    }
}
```

The Phase 3 `hasCompletedRecoveryFlow()` flag is already in `PrefsStore`. The new `hasCompletedFirstBackupFlow()` (Task 32) is set to `true` in three places: `FirstBackupActivity` on commit (Task 40), and the upgrade-path block in `PhotoBackupApp.onCreate` for pre-Phase-4 users (above).

## `AndroidManifest.xml`

### New permission

```xml
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
```

Goes alongside the existing `READ_MEDIA_IMAGES`. The runtime request is made by `FirstBackupActivity` and `SettingsActivity` when the user toggles `videos_enabled = true`; the manifest declaration is required regardless.

### New activities

```xml
<activity
    android:name=".ui.FirstBackupActivity"
    android:exported="false"
    android:theme="@style/Theme.PhotoStorage"
    android:configChanges="orientation|screenSize|screenLayout" />

<activity
    android:name=".ui.SettingsActivity"
    android:exported="false"
    android:theme="@style/Theme.PhotoStorage" />

<activity
    android:name=".ui.CostDashboardActivity"
    android:exported="false"
    android:theme="@style/Theme.PhotoStorage" />

<activity
    android:name=".ui.ActiveShareLinksActivity"
    android:exported="false"
    android:theme="@style/Theme.PhotoStorage" />

<activity
    android:name=".ui.LocalDeleteReviewActivity"
    android:exported="false"
    android:theme="@style/Theme.PhotoStorage" />
```

All non-exported, all using the existing app theme.

## Verification: end-to-end smoke test

Run against a real B2 bucket on a real device. Use a device with Phase 3 installed and a populated `uploads.db`; the v3 → v4 migration drops `uploads` (no real users), so expect to re-enter credentials and re-onboard.

### Pre-test setup

1. Uninstall the prior build (or clear app data). The v3 → v4 migration drops and recreates `uploads`.
2. Re-enter B2 credentials. The Phase 3 recovery prompt runs against B2 first.
3. Choose "Start fresh" at the recovery prompt (or "Restore" if a Phase 3 index already exists in your bucket — your call).

### Smoke checklist

| # | Step | Expected |
|---|------|----------|
| 1 | After onboarding (+ recovery), `FirstBackupActivity` opens at Step A | Title "Back up new photos automatically?" |
| 2 | Tap "Not now" | Gallery opens; no foreground service running |
| 3 | Reset by uninstalling + reinstalling; this time tap "Enable" → Step B | Title "What should we back up?" |
| 4 | Choose "From today only" → "Continue" → toggle videos OFF → "Finish" | Gallery opens; `videos_enabled = false` |
| 5 | Re-set state and choose "Entire gallery history" → Finish | `InitialBackfillWorker` enqueues every MediaStore image; no videos enqueued even if videos_enabled later toggles on |
| 6 | Open Settings → Backup timing → "Hybrid"; turn off Wi-Fi, take a photo | Photo enqueued but does not upload until Wi-Fi |
| 7 | Switch to Wi-Fi | Queue drains |
| 8 | Set Backup timing = Scheduled, take a photo at 14:00 | Photo enqueued; status notification reflects "uploads run nightly" |
| 9 | Force-run the worker via `adb shell cmd jobscheduler ...` or wait until 02:00 | Queue drains |
| 10 | Settings → Videos → enable. Grant permission. Record a 30s video | Video uploads at original quality to `videos/YYYY/MM/DD/...mp4` |
| 11 | Set quality mode = duration_based, threshold = 1 min. Record a 90s video | Video is transcoded; uploads to `*.compressed.mp4` |
| 12 | Set quality mode = original. Record another video | Original-quality upload |
| 13 | Settings → Storage management → Auto-delete strategy = Immediate. Tap "Review pending deletions now" | Notification appears (and/or activity launches); confirm |
| 14 | Approve the system delete dialog | Local files removed; rows show `local_present = 0`; Cloud view still shows the photos |
| 15 | Dismiss the notification three days in a row (simulate by setting `local_delete_dismiss_streak = 3`) | `suppress_until` is set; no new notification for 7 days |
| 16 | Settings → Storage & cost | Cost dashboard opens, shows totals and an estimated monthly cost |
| 17 | Long-press a Synced tile → Share link → 1 hour → Create | URL copied to clipboard; share sheet opens |
| 18 | Open the URL in a browser within 1 hour | Photo displays |
| 19 | Open the URL after 1 hour | 403 from B2 |
| 20 | Settings → Active share links | The created link appears with correct expiry |
| 21 | Wait 24h+ past expiry | Link disappears from the active list |
| 22 | Settings → Sign out → confirm | Routes back to onboarding; SQLite index is preserved on disk |
| 23 | Sign back in with same credentials | Recovery prompt restores index; gallery returns to its previous state |

### DB schema check

```bash
adb shell run-as com.hriyaan.photostorage sqlite3 databases/uploads.db ".schema uploads"
```

Verify the output includes `media_type TEXT NOT NULL DEFAULT 'photo'`, `original_path_b2 TEXT`, `pending_local_delete INTEGER NOT NULL DEFAULT 0`, `compressed INTEGER NOT NULL DEFAULT 0`.

```bash
adb shell run-as com.hriyaan.photostorage sqlite3 databases/uploads.db ".schema share_links"
```

Verify the `share_links` table exists with the documented columns.

### B2 path verification

```bash
aws s3 ls s3://<bucket>/videos/ --endpoint-url <b2-endpoint> --recursive
```

Expect a tree like:

```
videos/2026/05/17/IMG_001.mp4
videos/2026/05/17/IMG_002.compressed.mp4
```

## Constraints

- Do not introduce new dependencies beyond AndroidX Media3.
- Do not rewrite component implementations.
- `MainActivity` must remain a lightweight routing activity — no business logic.
- Smoke test must run on a real device against a real B2 bucket. Mock-only verification is not acceptable for Phase 4 sign-off.

## Acceptance criteria

- [ ] `./gradlew assembleDebug` succeeds with the new AndroidX Media3 dependencies.
- [ ] `PhotoBackupApp.onCreate` constructs every Phase 4 singleton and schedules `LocalDeleteScheduler`.
- [ ] Pre-Phase-4 users (existing `auto_upload_enabled = true`) silently flip `first_backup_flow_completed = true` on first launch after the Phase 4 update and never see `FirstBackupActivity`.
- [ ] `MainActivity` routing inserts `FirstBackupActivity` only on fresh installs (or after data wipe) where `first_backup_flow_completed = false`.
- [ ] All 23 smoke-checklist items pass on a real device.
- [ ] `./gradlew assembleDebug lint` is clean.
- [ ] AndroidManifest declares `READ_MEDIA_VIDEO` and all five new activities.

## Out of scope

- CI configuration changes (Firebase App Distribution still triggered manually).
- ProGuard / R8 rules for AndroidX Media3 (the library's consumer rules are sufficient at version 1.5+).
- Onboarding copy / marketing screens.
- App-icon refresh.
- Localization beyond English.
