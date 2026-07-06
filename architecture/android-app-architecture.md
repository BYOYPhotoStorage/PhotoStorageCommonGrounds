# PhotoStorage Android App — Architecture & Operational Guide

> Target audience: engineers who are not Android specialists. This document explains how the app is organized, how data moves between the device and the cloud, and how the major user workflows work.

---

## 1. What the app does

PhotoStorage is an Android app that backs up the user's local photos and videos to a Backblaze B2 bucket through B2's S3-compatible API. It can run automatically in the background, lets the user browse local and cloud-backed media, share items with expiring links, manage local storage by deleting originals that are already backed up, and restore its index when the app is reinstalled.

---

## 2. Modules and build system

The project lives in `@photoStorageApp/`. It is a Gradle + Kotlin project with two included modules:

| Module | Path | Purpose |
|--------|------|---------|
| `:app` | `photoStorageApp/app/` | The production Android app (`com.hriyaan.photostorage`). |
| `:companion` | `photoStorageApp/companion/` | A separate test-helper Android app (`com.photostorage.tester`) used by end-to-end tests to inject media into the device `MediaStore` without touching the production APK. |

Entry-point Gradle files:

- `photoStorageApp/settings.gradle.kts:24-27` includes `:app` and `:companion`.
- `photoStorageApp/build.gradle.kts:1-5` applies the Android application plugin and the Firebase / Google Services plugins at the root.
- `photoStorageApp/app/build.gradle.kts:1-103` configures the app: `compileSdk = 34`, `minSdk = 33`, `targetSdk = 34`, `viewBinding = true`, and declares dependencies through a version catalog.
- `photoStorageApp/gradle/libs.versions.toml:1-54` pins versions such as Kotlin `2.2.10`, AGP `9.1.1`, AWS S3 SDK `1.6.46`, WorkManager `2.9.0`, Media3 `1.4.1`, Coil `2.7.0`, and Security Crypto `1.1.0-alpha06`.

---

## 3. Technology stack

| Concern | Technology |
|---------|------------|
| Language | Kotlin |
| Build system | Gradle with Kotlin DSL, version catalogs |
| UI | XML layouts + ViewBinding; traditional `Activity`-centric screens; one `DialogFragment` (`ShareLinkDialog`) |
| Async / threading | Kotlin Coroutines (`Dispatchers.IO` for disk/network, `lifecycleScope` for UI) |
| Image loading | Coil (with custom fetchers for B2 thumbnails and video frames) |
| Cloud storage | AWS SDK for Kotlin `s3` talking to Backblaze B2's S3-compatible endpoint |
| HTTP engine | AWS Smithy OkHttp engine |
| Background work | WorkManager + a long-running `UploadForegroundService` |
| Video transcoding | AndroidX Media3 Transformer (H.264 / AAC) |
| Local database | Raw SQLite via `SQLiteOpenHelper` (no Room) |
| Encrypted settings | AndroidX Security Crypto `EncryptedSharedPreferences` |
| Crash reporting | Firebase Crashlytics |
| Testing | Maestro YAML flows, a `:companion` injector app, and a shell/Python `e2e-runner` |

---

## 4. Architectural style

The app is **not** using MVVM with `ViewModel`, MVP, or a formal clean-architecture layering. Instead it uses a **traditional Activity-centric + service-locator** style:

- Each screen is an `Activity`.
- Business logic lives in plain Kotlin classes: `Worker`s, a `Service`, `Repository`, `Dao`, and helper classes.
- The `Application` class (`PhotoBackupApp`) manually constructs and owns the long-lived objects that Activities and Workers reach into.
- Coroutines tie async work together; UI scopes use `lifecycleScope` and `repeatOnLifecycle`.
- `GalleryRepository` exposes a Kotlin `Flow<List<GalleryItem>>` so the gallery UI can observe changes.

This keeps the dependency surface small (no Hilt/Dagger/Koin/Room) but means the `Application` class is the central wiring point.

---

## 5. Application bootstrap and dependency wiring

`PhotoBackupApp.kt:25-113` is the heart of the app.

```kotlin
class PhotoBackupApp : Application() {
    lateinit var prefsStore: PrefsStore
    lateinit var uploadDatabase: UploadDatabase
    lateinit var s3Uploader: S3Uploader
    lateinit var galleryRepository: GalleryRepository
    lateinit var deletionEngine: DeletionEngine
    lateinit var thumbnailCacheFactory: ThumbnailCacheFactory
    lateinit var indexRecoveryService: IndexRecoveryService
    lateinit var shareLinkService: ShareLinkService
    lateinit var shareGalleryService: ShareGalleryService
    ...
}
```

On `onCreate` (`PhotoBackupApp.kt:57-81`):

1. Firebase Crashlytics is initialized.
2. `PrefsStore` and `UploadDatabase` are created.
3. Notification channels are ensured.
4. `GalleryRepository` is created.
5. Periodic background WorkManager jobs are scheduled (`IndexSyncWorker`, `SoftDeleteCleanupWorker`, `LocalDeleteWorker`).
6. Cloud services are **not** created yet if credentials are missing; they are built later by `initializeCloudServices()` (`PhotoBackupApp.kt:83-112`) after the user logs in.

This is a manual service locator. Classes receive their collaborators either from the `Application` cast (`applicationContext as PhotoBackupApp`) or by constructing them locally.

---

## 6. Entry points and component map

### AndroidManifest.xml

`photoStorageApp/app/src/main/AndroidManifest.xml:4-100` declares:

- Permissions: `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, `INTERNET`, `ACCESS_NETWORK_STATE`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_DATA_SYNC`, `POST_NOTIFICATIONS`, `RECEIVE_BOOT_COMPLETED`.
- Launcher `Activity`: `MainActivity`.
- Other screens: `OnboardingActivity`, `GalleryActivity`, `FirstBackupActivity`, `IndexRecoveryActivity`, `SettingsActivity`, `CostDashboardActivity`, `ActiveShareLinksActivity`, `LocalDeleteReviewActivity`, `DetailActivity`.
- Service: `UploadForegroundService` with `foregroundServiceType="dataSync"`.
- Receivers: `BootCompletedReceiver` (exported, listens for `BOOT_COMPLETED`) and `DeleteIntentReceiver` (exported false, handles notification dismissals).

### Routing

`MainActivity.kt:11-34` decides where the user lands:

```kotlin
when {
    !prefs.hasCredentials() -> OnboardingActivity
    prefs.getLastIndexSyncAt() == null && !prefs.hasCompletedRecoveryFlow() -> IndexRecoveryActivity
    !prefs.hasCompletedFirstBackupFlow() -> FirstBackupActivity
    else -> GalleryActivity
}
```

---

## 7. Data and persistence layer

### 7.1 SQLite database (`uploads.db`)

`UploadDatabase.kt:7-141` creates a single SQLite file with three tables.

#### `uploads` table

| Column | Purpose |
|--------|---------|
| `id` | Primary key |
| `local_uri` | MediaStore content URI string |
| `filename` | Display name |
| `size` | File size in bytes |
| `date_taken` | EXIF / MediaStore timestamp (ms) |
| `photo_b2_path` | S3 key for the full photo/video |
| `thumbnail_b2_path` | S3 key for the WebP thumbnail |
| `status` | `pending`, `uploading`, `uploaded`, `failed`, `permanently_failed`, `cloud_deleted` |
| `uploaded_at` | Timestamp when upload finished |
| `retry_count` / `next_retry_at` | Exponential backoff state |
| `sha256` | Content hash for duplicate detection |
| `created_at` | When the record was inserted |
| `local_present` | Whether the original file still exists on device |
| `cloud_deleted_at` | Soft-delete timestamp |
| `media_type` | `photo` or `video` |
| `original_path_b2` | Reserved for original-video path when a compressed copy was uploaded |
| `pending_local_delete` | Flag set by `LocalDeleteWorker` before user review |
| `compressed` | Whether the uploaded video was transcoded |
| `bucket_id` | Album / folder ID from MediaStore |

Indexes are created on status, filename+size, sha256, cloud_deleted_at, date_taken, media_type, and pending_local_delete (`UploadDatabase.kt:53-59`).

#### `share_links` and `share_galleries` tables

These store metadata for expiring share links. The actual shared content is just an S3 presigned URL; the tables remember the URL, the B2 path, creation time, and expiry so the app can list active shares (`ShareLinkDao.kt`, `ShareGalleryDao.kt`).

### 7.2 Data access objects

- `UploadDao.kt:28-523` is the main DAO. It supports insert, status updates, retry tracking, cloud view queries, soft delete, hard delete, batch "pending local delete", and statistics by media type.
- `ShareLinkDao.kt:14-79` and `ShareGalleryDao.kt:14-79` handle the two share tables.

All DAOs talk to the same `SQLiteOpenHelper` instance owned by `UploadDatabase`.

### 7.3 Encrypted preferences

`PrefsStore.kt:8-427` wraps `EncryptedSharedPreferences` (AES256_GCM values, AES256_SIV keys). It stores:

- B2 credentials (`key_id`, `application_key`, `bucket_name`).
- Feature toggles: auto-upload, Wi-Fi-only, videos enabled.
- User choices: upload timing mode, video quality/resolution/threshold, local-delete strategy, selected album bucket IDs.
- Index sync state: last synced hash, last sync timestamp, recovery flow completed.
- First-backup scope and onboarding completion.
- Cost/egress counters and diagnostics state.

### 7.4 File logger

`FileLogger.kt:10-69` is a singleton that writes a plain text log to `filesDir/logs/app.log` when diagnostics are enabled. It is used throughout the app for upload/scan/delete events and supports upload to B2 at `logs/app.log` from the Settings screen.

---

## 8. Local media discovery and indexing

The app never owns the photos itself; it reads the system `MediaStore` content provider.

### 8.1 `MediaStoreScanner`

`MediaStoreScanner.kt:14-207` queries `MediaStore.Images.Media.EXTERNAL_CONTENT_URI` and, when enabled and permitted, `MediaStore.Video.Media.EXTERNAL_CONTENT_URI`. It returns `MediaItem` objects containing URI, filename, size, date taken, media type, duration, and bucket ID. It supports filtering by:

- `since` timestamp (for incremental scans).
- `bucketIds` set (for album selection).

### 8.2 `MediaStoreQuery`

`MediaStoreQuery.kt:7-124` is a simpler helper used by the UI to list folders/albums (`queryPhotoFolders`) and to load all photos for the gallery view.

### 8.3 Duplicate detection

`DuplicateDetector.kt:18-76` runs before a new item is enqueued:

1. Looks for an existing record with the same `filename`, `size`, and `dateTaken`.
2. If not found, computes SHA-256 of the file bytes and looks for a matching hash.
3. If either matches, the item is skipped.

---

## 9. Cloud / network layer

All cloud operations go through the AWS SDK for Kotlin S3 client.

### 9.1 Client creation

`S3ClientFactory.kt:9-19` builds an `S3Client`:

```kotlin
S3Client {
    region = config.region
    endpointUrl = Url.parse(config.endpoint)
    credentialsProvider = StaticCredentialsProvider {
        accessKeyId = credentials.keyId
        secretAccessKey = credentials.applicationKey
    }
    httpClient = OkHttpEngine()
}
```

`S3Config.kt:7-20` defaults to Backblaze B2's `us-west-004` region and an endpoint of `https://s3.us-west-004.backblazeb2.com`.

### 9.2 Upload / download wrapper

`S3Uploader.kt:20-114` exposes the operations the app uses:

- `validateCredentials()` — `ListBuckets` during login.
- `upload(key, contentType, contentLength, body)` — `PutObject` with `checksumAlgorithm = null` because B2 rejects AWS's default CRC32 checksums.
- `deleteObject(path)` — `DeleteObject`.
- `headObject(path)` — `HeadObject` for index existence checks.
- `downloadObject(path, dest)` — `GetObject` for index restore and thumbnail cache misses.
- `presignGetUrl(path, ttlSeconds)` — creates time-limited HTTPS URLs for sharing.

### 9.3 Object key layout

`S3KeyBuilder.kt:8-31` organizes objects by date:

- Full photos: `photos/YYYY/MM/DD/filename.jpg`
- Thumbnails: `thumbnails/YYYY/MM/DD/basename.webp`
- Videos: `videos/YYYY/MM/DD/filename.mp4`
- Compressed videos: `videos/YYYY/MM/DD/basename.compressed.mp4`
- SQLite index: `index/photo-storage-index.sqlite`
- Shared galleries: `shares/gallery-${UUID}.html`
- Diagnostic logs: `logs/app.log`

---

## 10. Upload pipeline

The upload pipeline has two triggers: a **foreground service** for reactive auto-upload, and **WorkManager workers** for scheduled/nightly/bulk work. The actual upload logic is shared in `UploadWorker` (a plain class, not a WorkManager worker).

### 10.1 `UploadForegroundService`

`UploadForegroundService.kt:39-311` is started when auto-upload is enabled.

What it does:

1. Calls `startForeground()` with a persistent notification so Android keeps it alive (`UploadNotificationManager.buildForegroundNotification`).
2. Registers `ContentObserver`s on the images URI (and the videos URI if videos are enabled).
3. When the MediaStore changes, it debounces for 3 seconds (`DEBOUNCE_MS = 3000L`) and runs `handleMediaChange()` (`UploadForegroundService.kt:191-251`).
4. `handleMediaChange()` scans MediaStore since the last scan, deduplicates, inserts pending records, updates `lastScanTimestamp`, invalidates the gallery cache, and triggers `UploadWorker.processQueue()`.
5. Uploads run under a `Mutex` so only one queue run happens at a time.

### 10.2 `UploadWorker`

`UploadWorker.kt:31-404` is the actual upload engine.

`processQueue()`:

1. Reads `uploadMode` and `wifiOnly` from `PrefsStore`.
2. Asks `UploadModeGate` whether uploads should run now or be deferred.
3. Loads pending records plus retryable failed records.
4. Filters by selected album bucket IDs if the user restricted folders.
5. For each record:
   - Sets status to `uploading`.
   - Shows a progress notification.
   - Calls `processPhotoRecord()` or `processVideoRecord()`.
   - On success, updates the record to `uploaded`.
   - On failure, either retries with exponential backoff or marks `permanently_failed`.

`processPhotoRecord()` (`UploadWorker.kt:142-208`):

1. Opens the content URI and reads bytes.
2. Uploads the original bytes to the `photos/...` key.
3. Generates a 200x200 WebP thumbnail.
4. Uploads the thumbnail to the `thumbnails/...` key.

`processVideoRecord()` (`UploadWorker.kt:210-303`):

1. Decides whether to compress based on the user's video quality setting.
2. If compression is needed, `Transcoder.kt:32-165` uses Media3 Transformer to produce an H.264/AAC MP4 at 720p or 1080p.
3. Uploads the original or compressed file.
4. Generates a frame thumbnail and uploads it.
5. Records whether compression happened.

Thumbnail generation is in `ThumbnailGen.kt:12-100`. It prefers `ContentResolver.loadThumbnail()` and falls back to subsampled `BitmapFactory` decoding.

### 10.3 Retry and permanent failure

`UploadWorker.kt:375-389` (`markFailed`):

- Increments `retryCount`.
- If `retryCount < 5`, computes `nextRetryAt = now + 2^retryCount minutes` and sets status `failed`.
- If `retryCount >= 5`, sets status `permanently_failed`.
- S3 authentication errors (`InvalidAccessKeyId`, `SignatureDoesNotMatch`, `AccessDenied`) are immediately marked `permanently_failed` and an auth notification is shown.

---

## 11. Background workers and scheduling

WorkManager is used for work that does not need to be reactive.

| Worker | Scheduler | Purpose |
|--------|-----------|---------|
| `IndexSyncWorker` | `IndexSyncScheduler` | Daily at 3 AM, upload `uploads.db` to B2 if it changed. |
| `NightlyScanWorker` | `NightlyScanScheduler` | Daily at 2 AM, scan MediaStore, enqueue pending items, then run uploads. |
| `InitialBackfillWorker` | One-shot from `FirstBackupActivity` | First-time bulk scan based on the user's chosen scope. |
| `LocalDeleteWorker` | `LocalDeleteScheduler` | Daily at 7 PM, evaluate local-delete strategy and notify the user. |
| `SoftDeleteCleanupWorker` | `SoftDeleteCleanupScheduler` | Daily, hard-delete records that were soft-deleted more than 24 hours ago. |

All schedulers respect the user's Wi-Fi-only preference by setting WorkManager network constraints to `UNMETERED` or `CONNECTED`.

`BootCompletedReceiver.kt:10-20` restarts the foreground service and re-schedules the nightly scan after a reboot if auto-upload is enabled.

---

## 12. Gallery, thumbnails, and caching

### 12.1 `GalleryRepository`

`GalleryRepository.kt:18-316` builds the data set the user sees.

It supports three view modes:

- **Local** — everything currently on the device, matched against DB to show queued/uploaded state.
- **Cloud** — everything ever uploaded that is not soft-deleted; thumbnails come from local MediaStore when possible, otherwise from B2.
- **Merged** — a unified view where local+cloud matches become `Synced`, cloud-only items are `CloudOnly`, and not-yet-uploaded local items are `LocalOnly`.

The repository exposes `observe(mode): Flow<List<GalleryItem>>` and invalidates its cache when upload/delete state changes.

### 12.2 Thumbnail loading

`ThumbnailCacheFactory.kt:16-66` builds a Coil `ImageLoader` with:

- A 200 MB disk cache in `cacheDir/thumbnails`.
- A memory cache capped at 20% of available memory.
- Two custom fetchers registered:
  - `B2ThumbnailFetcher` — downloads a B2 thumbnail on cache miss and records egress bytes for the cost dashboard.
  - `VideoFrameFetcher` — extracts a frame from a local video URI.

Prefetching only runs on unmetered networks when Wi-Fi-only is enabled.

---

## 13. Deletion and storage management

### 13.1 Manual deletion

`DeletionEngine.kt:29-245` handles deletion from the gallery.

Behavior depends on the current view mode:

- **Local mode** — deletes the local MediaStore file. If the item was already uploaded, the DB record is kept with `localPresent = false`.
- **Cloud mode** — deletes the B2 objects and soft-deletes the DB record.
- **Merged mode** — deletes both. If one side fails, it updates `localPresent` to reflect the partial state.

For local deletes, the engine batches URIs into a single system `MediaStore.createDeleteRequest()` IntentSender.

### 13.2 Automatic local storage cleanup

`LocalDeleteWorker.kt:25-127` runs daily and evaluates the user's strategy:

- `never` — do nothing.
- `immediate` — mark all uploaded local files for deletion.
- `after_days` — mark uploaded files older than N days.
- `after_count` — if N new uploads have happened since the last run, mark the oldest N.

Marked records get `pendingLocalDelete = 1`. A notification opens `LocalDeleteReviewActivity`, where the user reviews the list, unchecks items, and confirms. Confirming launches the system delete request; on approval the app sets `localPresent = false`.

Dismissing the notification 3 times suppresses prompts for 7 days (`DeleteIntentReceiver.kt:8-20`).

### 13.3 Soft-delete cleanup

`SoftDeleteCleanupWorker.kt:11-27` permanently removes DB records that were soft-deleted more than 24 hours ago. This is a safety window: cloud objects are deleted immediately, but the local record lingers briefly.

---

## 14. Sharing

### 14.1 Single-item share links

`ShareLinkService.kt:8-47` creates a presigned `GetObject` URL for a cloud-backed photo or video and stores it in `share_links` with a TTL of 1 hour, 1 day, or 1 week (`ShareLinkTtl.kt:5-11`).

### 14.2 Gallery share links

`ShareGalleryService.kt:13-99` creates a shareable HTML page:

1. Generates presigned URLs for each selected item's full file and thumbnail.
2. `ShareGalleryHtmlGenerator.kt:7-41` reads `assets/share_gallery_template.html` and injects a JSON array of items.
3. Uploads the HTML to B2 under `shares/gallery-${UUID}.html`.
4. Creates a presigned URL for the HTML page and stores metadata in `share_galleries`.

### 14.3 Active links screen

`ActiveShareLinksActivity.kt:28-124` lists non-expired share links and lets the user copy or open them.

---

## 15. Index backup and recovery

Because the local SQLite database is the source of truth for what has been uploaded, the app can back it up to B2 and restore it on a new install.

### 15.1 Backup

`IndexSyncWorker.kt:17-100` runs daily:

1. Copies `uploads.db` to the cache directory.
2. Computes SHA-256 of the copy.
3. If the hash differs from the last synced hash, uploads the DB to `index/photo-storage-index.sqlite`.
4. Updates `lastSyncedIndexHash` and `lastIndexSyncAt`.

### 15.2 Restore

`IndexRecoveryService.kt:18-137`:

1. `hasRemoteIndex()` does a `HeadObject` on the index path.
2. `downloadAndRestore()` downloads the index, validates that it contains an `uploads` table and is not newer than the app supports, closes the local DB, overwrites it, and reconciles `localPresent` flags by scanning MediaStore.

`IndexRecoveryActivity.kt:26-247` is the UI flow that offers **Restore**, **Start fresh**, **Catch up from date**, **Full rescan**, or **Skip**.

---

## 16. Notifications and permissions

### 16.1 Notification channels

`NotificationChannels.kt:8-33` creates two channels:

- `upload_service` — low importance, persistent foreground notification.
- `upload_events` — progress, completion, auth failure, permanent failure, and local-delete-review notifications.

`UploadNotificationManager.kt:15-107` builds all notifications.

### 16.2 Permissions

Runtime permissions requested:

- `READ_MEDIA_IMAGES` — required to read photos (`GalleryActivity.kt:90-99`, `PhotoPermission.kt:8-15`).
- `READ_MEDIA_VIDEO` — requested when the user enables video backup (`FirstBackupActivity.kt:61-69`, `SettingsActivity.kt:46-55`).
- `POST_NOTIFICATIONS` — requested on Android 13+ (`GalleryActivity.kt:101-103`).

`FOREGROUND_SERVICE_DATA_SYNC` and `RECEIVE_BOOT_COMPLETED` are declared in the manifest.

---

## 17. Cost dashboard and diagnostics

### 17.1 Cost dashboard

`CostDashboardService.kt:9-63` reads the SQLite DB and `PrefsStore` to compute:

- Uploaded photo/video counts and bytes.
- Pending uploads.
- Permanently failed uploads.
- Estimated monthly storage cost at `$0.006/GB`.
- Estimated monthly egress cost at `$0.01/GB` (when egress bytes > 0).
- Breakdown by year (last 5 years).

Prices are constants in `B2Pricing.kt:3-17`. `CostDashboardActivity.kt:20-143` renders the numbers.

### 17.2 Diagnostics

`FileLogger.kt` is described in section 7.4. The Settings screen can start/stop logging, upload the log to B2, and clear it.

---

## 18. User workflows and scenarios

### Scenario A: First install and login

1. `MainActivity` sees no credentials and opens `OnboardingActivity`.
2. The user enters Key ID, Application Key, and Bucket Name (or taps a test shortcut).
3. `OnboardingActivity.kt:56-93` validates the fields, builds an `S3Client`, and calls `S3Uploader.validateCredentials()` (`ListBuckets`).
4. On success, `PrefsStore.saveCredentials()` encrypts the credentials and `PhotoBackupApp.initializeCloudServices()` creates the global `S3Uploader`, `DeletionEngine`, share services, etc.
5. The user is taken to `FirstBackupActivity` because `firstBackupFlowCompleted` is still false.

### Scenario B: First backup setup

1. `FirstBackupActivity.kt:42-343` shows a wizard: enable auto-upload, choose scope, pick folders, enable videos.
2. The chosen scope is translated into a `firstBackupSince` timestamp and saved.
3. If auto-upload is enabled, `InitialBackfillWorker` is enqueued with that timestamp.
4. `InitialBackfillWorker.kt:17-97` scans MediaStore, deduplicates, and inserts pending `UploadRecord`s.
5. `UploadForegroundService.start()` is called so future photos are picked up reactively.

### Scenario C: Take a new photo and auto-upload it

1. The camera app writes a new image to MediaStore.
2. `UploadForegroundService`'s `ContentObserver` fires.
3. After a 3-second debounce, `handleMediaChange()` scans MediaStore since the last scan.
4. `DuplicateDetector` confirms the photo is new.
5. A pending record is inserted with S3 keys.
6. `UploadWorker.processQueue()` checks `UploadModeGate`:
   - Immediate mode + not Wi-Fi-only → upload now.
   - Hybrid mode → upload only on unmetered network.
   - Scheduled mode → defer to the nightly window unless it is already 2-3 AM.
7. `processPhotoRecord()` reads the bytes, uploads the original, generates a WebP thumbnail, uploads the thumbnail, and marks the record `uploaded`.
8. `GalleryRepository` cache is invalidated; the gallery shows the item as `Synced`.

### Scenario D: Manual batch upload from the gallery

1. In `GalleryActivity`, the user long-presses a `LocalOnly` item, selects more items, and taps **Upload**.
2. `GalleryActivity.onUploadSelected()` (`GalleryActivity.kt:466-532`) inserts pending records or resets failed records.
3. `UploadForegroundService.processQueueNow()` starts the service with the `PROCESS_QUEUE` action.
4. The same `UploadWorker` pipeline runs as in Scenario C.

### Scenario E: Upload a long video with compression

1. The user enables videos and sets quality to `duration_based` with a 2-minute threshold.
2. A 5-minute video is enqueued.
3. `UploadWorker.processVideoRecord()` sees the duration exceeds the threshold.
4. `Transcoder.transcode()` produces a compressed H.264/AAC MP4 at the chosen resolution.
5. The compressed file is uploaded to `videos/YYYY/MM/DD/basename.compressed.mp4`.
6. `UploadDao.setUploadedVideoPaths()` records `compressed = true`.

### Scenario F: Go offline, take photos, come back online

1. MediaStore changes are detected and pending records are inserted.
2. `UploadWorker` runs but `UploadModeGate` sees no network (or WorkManager constraints are not met) and defers.
3. Records stay `pending` or `failed` with `nextRetryAt`.
4. When connectivity returns, the foreground service or the next WorkManager run picks them up and retries.

### Scenario G: Delete a photo from the device but keep the cloud copy

1. In merged view, the user selects a `Synced` item and taps **Delete**.
2. `DeletionEngine.deleteBatch()` (`DeletionEngine.kt:43-74`) launches the system MediaStore delete request for the local URIs.
3. `deleteMergedMode()` (`DeletionEngine.kt:155-205`) deletes the B2 objects, soft-deletes the DB record, deletes the local file, then hard-deletes the record because both sides succeeded.

> Wait — this hard-deletes the record, so the cloud copy is gone too in merged mode. To keep only the cloud copy, use **Cloud view** and delete there, or the local delete strategy flow.

Correct flow for "free local storage but keep cloud":

1. `LocalDeleteWorker` marks uploaded local files as `pendingLocalDelete`.
2. The user opens `LocalDeleteReviewActivity`, reviews, and confirms.
3. Only the local MediaStore files are deleted; the DB record stays with `localPresent = false`.
4. The item now appears as `CloudOnly` in merged view.

### Scenario H: Reinstall the app and recover the index

1. On first launch after reinstall, `MainActivity` sees credentials but no `lastIndexSyncAt` and recovery not completed.
2. `IndexRecoveryActivity` checks `IndexRecoveryService.hasRemoteIndex()`.
3. If an index exists, the user chooses **Restore**.
4. `downloadAndRestore()` replaces the local `uploads.db` with the remote copy and reconciles `localPresent`.
5. The user then chooses catch-up scope, `NightlyScanWorker` scans for new local photos, and the app proceeds to the gallery.

### Scenario I: Share a cloud-backed photo

1. In `GalleryActivity`, the user selects a `Synced` or `CloudOnly` item and taps **Share link**.
2. `ShareLinkDialog` asks for TTL (1 hour / 1 day / 1 week).
3. `ShareLinkService.createLink()` calls `S3Uploader.presignGetUrl()`.
4. The URL is saved to `share_links` and copied to the clipboard.
5. `ActiveShareLinksActivity` shows the link until it expires.

### Scenario J: Change settings

1. The user opens `SettingsActivity`.
2. Toggling auto-upload starts/stops `UploadForegroundService` and schedules/cancels `NightlyScanWorker`.
3. Toggling Wi-Fi-only changes the network constraints used by `UploadModeGate`, `IndexSyncScheduler`, and `NightlyScanScheduler`.
4. Changing backup folders clears the pending queue (`SettingsActivity.kt:165-225`) because the scope changed.
5. Changing the local-delete strategy changes what `LocalDeleteWorker` will mark the next day.

---

## 19. Test support

The production app is tested with a multi-tool setup documented separately in `mobile-end-to-end-testing-architecture.md`. The pieces that live in this repo are:

- **Maestro flows** in `photoStorageApp/maestroTests/` — black-box UI tests (login, wait-for-gallery, trigger index sync).
- **`:companion` app** in `photoStorageApp/companion/` — a separate APK that injects photos/videos into `MediaStore` via explicit intents.
- **`e2e-runner/`** — shell/Python orchestration that resets state, injects media, runs Maestro, verifies the B2 bucket, and cleans up.

These are not part of the production architecture but they exercise the same MediaStore, SQLite, S3, and WorkManager paths described above.

---

## 20. Notable source file index

| File | Responsibility |
|------|----------------|
| `PhotoBackupApp.kt` | Application class, manual service locator, schedules periodic workers |
| `MainActivity.kt` | Launcher activity, routes to onboarding/recovery/first-backup/gallery |
| `ui/OnboardingActivity.kt` | B2 credential entry and validation |
| `ui/FirstBackupActivity.kt` | First-time wizard: scope, folders, videos, auto-upload |
| `ui/GalleryActivity.kt` | Main gallery screen, view modes, selection, upload/delete/share actions |
| `ui/SettingsActivity.kt` | All user preferences and toggles |
| `ui/IndexRecoveryActivity.kt` | Restore or catch up the SQLite index from B2 |
| `ui/LocalDeleteReviewActivity.kt` | Review local files marked for deletion |
| `ui/ActiveShareLinksActivity.kt` | List active share links |
| `ui/CostDashboardActivity.kt` | Render cost estimates |
| `data/PrefsStore.kt` | EncryptedSharedPreferences wrapper |
| `data/UploadDatabase.kt` / `UploadDao.kt` | SQLite schema and uploads CRUD |
| `data/ShareLinkDao.kt` / `ShareGalleryDao.kt` | Share metadata tables |
| `data/FileLogger.kt` | On-device diagnostic logging |
| `media/MediaStoreScanner.kt` / `MediaStoreQuery.kt` | Read the system MediaStore |
| `media/Transcoder.kt` | Video compression with Media3 Transformer |
| `dedup/DuplicateDetector.kt` | Filename/size/date and SHA-256 duplicate detection |
| `gallery/GalleryRepository.kt` | Build local/cloud/merged gallery views |
| `gallery/GalleryItem.kt` | Sealed model: LocalOnly / CloudOnly / Synced |
| `gallery/DeletionEngine.kt` | Local/cloud/merged deletion logic |
| `b2/S3ClientFactory.kt` / `b2/S3Config.kt` | Build the AWS S3 client for B2 |
| `b2/S3Uploader.kt` | Upload, download, delete, head, presign URLs |
| `b2/S3KeyBuilder.kt` | Date-based S3 key generation |
| `service/UploadForegroundService.kt` | Foreground service with MediaStore observers |
| `worker/UploadWorker.kt` | Core upload engine invoked by service and workers |
| `worker/UploadModeGate.kt` | Decide whether to upload now based on mode + network |
| `worker/IndexSyncWorker.kt` / `IndexSyncScheduler.kt` | Backup SQLite DB to B2 |
| `worker/NightlyScanWorker.kt` / `NightlyScanScheduler.kt` | Scheduled nightly scan + upload |
| `worker/InitialBackfillWorker.kt` | First-time bulk scan |
| `worker/LocalDeleteWorker.kt` / `LocalDeleteScheduler.kt` | Automatic local-storage cleanup suggestions |
| `worker/SoftDeleteCleanupWorker.kt` | Hard-delete old soft-deleted records |
| `thumbnail/ThumbnailGen.kt` | WebP thumbnail generation |
| `thumbnail/ThumbnailCacheFactory.kt` | Coil image loader with disk/memory cache |
| `thumbnail/B2ThumbnailFetcher.kt` | Download cloud thumbnails through Coil |
| `thumbnail/VideoFrameFetcher.kt` | Extract video frames for thumbnails |
| `share/ShareLinkService.kt` / `ShareGalleryService.kt` | Create expiring share links/galleries |
| `recovery/IndexRecoveryService.kt` | Download and restore the remote index |
| `notification/UploadNotificationManager.kt` | All notifications |
| `receiver/BootCompletedReceiver.kt` | Restart service/scheduler after reboot |
| `receiver/DeleteIntentReceiver.kt` | Handle notification dismiss streak |
| `cost/CostDashboardService.kt` / `B2Pricing.kt` | Cost estimation |

---

## 21. Glossary

| Term | Meaning |
|------|---------|
| **MediaStore** | Android's system content provider that indexes photos, videos, and audio on external storage. |
| **B2** | Backblaze B2 cloud storage, used here through its S3-compatible API. |
| **S3 key** | The path-like identifier of an object in an S3/B2 bucket, e.g. `photos/2026/07/04/IMG.jpg`. |
| **Presigned URL** | A time-limited HTTPS URL generated by the app that lets anyone download a private B2 object. |
| **ContentObserver** | Android mechanism that notifies the app when the MediaStore content changes. |
| **WorkManager** | Android library for deferrable background work that survives reboots and respects constraints. |
| **Foreground service** | A service that shows a persistent notification and is less likely to be killed by the OS. |
| **SQLiteOpenHelper** | Android helper class that manages creating/upgrading a raw SQLite database. |
| **ViewBinding** | Compile-time generated bindings between XML layouts and Kotlin code. |
| **Coil** | Kotlin image-loading library used for thumbnails. |

---

*Last updated: 2026-07-04*
