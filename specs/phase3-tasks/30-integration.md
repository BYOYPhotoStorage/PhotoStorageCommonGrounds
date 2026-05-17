# Task 30 — Integration & Wire-up

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Glue all Phase 3 components into the application: instantiate singletons in `PhotoBackupApp`, route `MainActivity` through the `IndexRecoveryActivity` on first launch, schedule the new periodic workers, add the Coil dependency, declare the new activity in the manifest, and run the end-to-end smoke test.

## Files you own (touch)

- `app/src/main/java/com/hriyaan/photostorage/PhotoBackupApp.kt` (modify)
- `app/src/main/java/com/hriyaan/photostorage/MainActivity.kt` (modify)
- `app/build.gradle.kts` (modify — Coil dependency)
- `app/src/main/AndroidManifest.xml` (modify — declare `IndexRecoveryActivity`)

You should **not** modify files owned by Tasks 20–29. If a wiring problem requires a contract change, raise it back to the owning task.

## `build.gradle.kts` changes

Add to the `dependencies` block:

```kotlin
implementation("io.coil-kt:coil:2.7.0")
```

No other dependencies are added. WorkManager (`androidx.work:work-runtime-ktx:2.9.1`) is already present from Phase 2 Task 19.

## `PhotoBackupApp`

Update `onCreate` to construct Phase 3 singletons and schedule the new periodic workers. Keep all Phase 2 wiring intact.

```kotlin
class PhotoBackupApp : Application() {
    lateinit var prefsStore: PrefsStore
    lateinit var uploadDatabase: UploadDatabase
    lateinit var s3Uploader: S3Uploader

    // Phase 3 singletons
    lateinit var galleryRepository: GalleryRepository
    lateinit var deletionEngine: DeletionEngine
    lateinit var thumbnailCacheFactory: ThumbnailCacheFactory
    lateinit var indexRecoveryService: IndexRecoveryService

    override fun onCreate() {
        super.onCreate()
        prefsStore = PrefsStore(this)
        uploadDatabase = UploadDatabase(this)
        s3Uploader = S3Uploader(/* existing wiring */)
        UploadNotificationManager(this).ensureChannels()

        galleryRepository = GalleryRepository(this, uploadDatabase.dao, prefsStore)
        deletionEngine = DeletionEngine(this, uploadDatabase.dao, s3Uploader, galleryRepository)
        thumbnailCacheFactory = ThumbnailCacheFactory(this, s3Uploader, prefsStore)
        indexRecoveryService = IndexRecoveryService(this, s3Uploader, uploadDatabase.dao, uploadDatabase, prefsStore)

        // Schedule Phase 3 workers
        IndexSyncScheduler.schedule(this)
        SoftDeleteCleanupScheduler.schedule(this)

        // Phase 2's NightlyScanScheduler is already scheduled inside GalleryActivity when auto-upload is on
    }
}
```

> Notes:
> - `IndexSyncScheduler.schedule` uses `ExistingPeriodicWorkPolicy.UPDATE` so re-scheduling on each launch is safe and picks up Wi-Fi-only changes.
> - `SoftDeleteCleanupScheduler.schedule` uses `KEEP`.
> - The Phase 2 `UploadWorker` should be updated (small change to that file is allowed here only if Task 14's owner approves) to call `galleryRepository.invalidate()` after each status change. If the worker cannot reach `galleryRepository`, expose a top-level helper `app.galleryRepository.invalidate()` and call it from the worker.
> - The Phase 2 `UploadForegroundService` `ContentObserver` should also call `invalidate()` after debouncing. Same coordination rule.

## `MainActivity`

Update routing to insert the recovery flow between credentials and gallery:

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

        if (app.prefsStore.getLastIndexSyncAt() == null && !app.prefsStore.hasCompletedRecoveryFlow()) {
            startActivity(Intent(this, IndexRecoveryActivity::class.java))
            finish()
            return
        }

        startActivity(Intent(this, GalleryActivity::class.java))
        finish()
    }
}
```

> `hasCompletedRecoveryFlow()` is a new boolean flag in `PrefsStore` — coordinate with Task 21's owner to add it (`recovery_flow_completed`, default false). `IndexRecoveryActivity` sets it true on completion. This prevents the recovery prompt from re-triggering forever in the "Skip for now" case where `last_index_sync_at` may also stay null until the first nightly sync runs.

> Alternatively, treat "Skip for now" as setting `last_index_sync_at = System.currentTimeMillis()` so the existing check naturally short-circuits. Use whichever Task 21's owner finds cleaner.

## `AndroidManifest.xml`

Declare the new activity:

```xml
<activity
    android:name=".recovery.IndexRecoveryActivity"
    android:exported="false"
    android:theme="@style/Theme.PhotoStorage"
    android:configChanges="orientation|screenSize|screenLayout" />
```

No new permissions.

## Verification: end-to-end smoke test

Test against a real B2 bucket with a real device. Use a device that has Phase 2 installed and a populated `uploads.db`.

### Pre-test setup

1. Uninstall any prior build (or wipe app data). The v2 → v3 migration drops and recreates the `uploads` table, so existing rows are not preserved — start from a clean slate.
2. Re-enter B2 credentials. The gallery starts empty; the catch-up flow (or the foreground service, once enabled) will populate it.

### Smoke checklist

| # | Step | Expected |
|---|------|----------|
| 1 | Launch app, complete onboarding, take a few photos so the upload pipeline populates `uploads` | Photos upload; gallery loads in Merged view by default |
| 2 | Switch to Local view | Only local photos visible, no B2 fetches |
| 3 | Switch to Cloud view | All `status='uploaded'` photos visible; tiles use B2 thumbnails for cloud-only items |
| 4 | Long-press a `Synced` photo → enter selection mode → delete | Confirmation says "Delete from phone and cloud?" with destructive button; both copies removed |
| 5 | In Cloud view, delete a photo | Confirmation says "Delete from cloud?"; B2 objects gone, local file unchanged |
| 6 | In Local view, delete a `Synced` photo | Confirmation says "Delete from this device?"; local file gone, row visible in Cloud view |
| 7 | Tap overflow → "Back up index now" | Index uploads to B2; status line updates to "Index last backed up just now" |
| 8 | Tap "Back up index now" again immediately | No upload fires (hash gate); status timestamp does not change |
| 9 | Take a photo with the camera app | Upload pipeline triggers; index status updates on next scheduled sync |
| 10 | Uninstall + reinstall + re-enter B2 credentials | "Restore your previous backup index?" prompt appears |
| 11 | Tap "Restore" | Index downloads, swaps in, "Restored N photos" shown |
| 12 | Catch-up prompt appears with three options | Choose "Catch up from this date" — one-shot scan enqueued |
| 13 | After the scan completes, Cloud view matches the previous device's state | Photos appear without a full rescan |
| 14 | Trigger a soft delete on a cloud photo, then force-run the cleanup worker after 24h | Row is hard-deleted from `uploads` |
| 15 | Scroll through Cloud view twice | Second pass serves thumbnails from disk cache (verify via logs / no B2 GETs) |
| 16 | Toggle Wi-Fi-only ON, switch to mobile data, scroll Cloud view | Prefetch is suppressed; tiles still load on visible items |

### DB schema check

```bash
adb shell run-as com.hriyaan.photostorage sqlite3 databases/uploads.db ".schema uploads"
```

Verify the output includes `local_present INTEGER NOT NULL DEFAULT 1` and `cloud_deleted_at INTEGER`.

### B2 index verification

After Step 7 succeeds:

```bash
aws s3 ls s3://<bucket>/index/ --endpoint-url <b2-endpoint>
```

Expect `photo-storage-index.sqlite` present, sized within a few MB.

## Constraints

- Do not introduce new dependencies beyond Coil.
- Do not rewrite component implementations.
- `MainActivity` must remain a lightweight routing activity — no business logic.
- Smoke test must run on a real device against a real B2 bucket. Mock-only verification is not acceptable for Phase 3 sign-off.

## Acceptance criteria

- [ ] `./gradlew assembleDebug` succeeds with the new Coil dependency.
- [ ] `PhotoBackupApp.onCreate` constructs all six Phase 3 singletons and schedules both new periodic workers.
- [ ] `MainActivity` routes to `IndexRecoveryActivity` only on the first post-credential launch; subsequent launches go straight to `GalleryActivity`.
- [ ] All 16 smoke-checklist items pass on a real device.
- [ ] Manual "Back up index now" works and the hash gate suppresses redundant uploads.
- [ ] Reinstall recovery successfully restores the index and the catch-up scan enqueues the right photos.
- [ ] `./gradlew assembleDebug lint` is clean.

## Out of scope

- CI configuration changes.
- ProGuard / R8 rules for Coil (default rules in the library are sufficient).
- Firebase App Distribution release build artifact.
- Onboarding copy / marketing screens.
