# PhotoStorage Android App — High-Level System Design

This document describes the PhotoStorage Android app at a system-design level: what systems it talks to, what major containers and components exist inside it, and how data flows through the most important operations. It is intended for engineers who are not Android specialists.

---

## 1. Scope

The system under design is the **PhotoStorage Android app** (`com.hriyaan.photostorage`). It is a client application that:

1. Reads the user's local photos and videos from the Android `MediaStore`.
2. Backs up originals and thumbnails to a user-owned **Backblaze B2** bucket using B2's S3-compatible API.
3. Maintains a local SQLite index of what has been backed up.
4. Provides gallery browsing, sharing with expiring links, local-storage cleanup, and index recovery.

The companion test app, Maestro flows, and shell/Python e2e harness are not part of the production system, but they are shown in the context diagram because they are important test surfaces.

---

## 2. System context (C4 Level 1)

```
                           +-------------------+
                           |   Android User    |
                           +---------+---------+
                                     |
                                     | taps, selects, configures
                                     v
+------------------------------------+------------------------------------+
|                          PhotoStorage Android App                       |
|  (reads MediaStore, backs up to B2, shows gallery, manages storage)    |
+------------------------------------+------------------------------------+
                 |                                |
                 | reads/observes                 | S3-compatible API
                 v                                v
+----------------+------------------+      +---------------------+
| Android MediaStore                 |      | Backblaze B2        |
| (system content provider           |      | (user's own bucket) |
|  for photos/videos)                |      |                     |
+------------------------------------+      +---------------------+

External test/QA surfaces (not production):
+----------------------+      +-------------------+      +------------------+
| :companion app       |      | Maestro UI flows  |      | e2e-runner       |
| (injects test media  |      | (black-box UI     |      | (shell/Python    |
|  into MediaStore)    |      |  automation)      |      |  orchestration)  |
+----------------------+      +-------------------+      +------------------+
```

### External systems

| System | Role | Protocol / API |
|--------|------|----------------|
| **Android MediaStore** | System content provider that indexes all photos and videos on external storage. The app queries it and registers observers for changes. | Android `ContentResolver` + `ContentObserver` |
| **Backblaze B2** | Object storage where originals, thumbnails, shared HTML pages, diagnostic logs, and the SQLite index backup are stored. | HTTPS — S3-compatible API via AWS SDK for Kotlin |
| **Firebase Crashlytics** | Crash reporting only; no analytics events are logged. | Firebase SDK |

---

## 3. Container diagram (C4 Level 2)

Inside the Android device, the production system consists of the following containers:

```
+-------------------------------------------------------------------------+
|                           Android Device                                |
|                                                                         |
|  +--------------------+        +-------------------------------+       |
|  |  PhotoStorage App  |        |  Android MediaStore           |       |
|  |  (:app module)     |<------>|  (system content provider)    |       |
|  +---------+----------+        +-------------------------------+       |
|            |                                                            |
|            | HTTPS / S3-compatible API                                  |
|            v                                                            |
|  +--------------------+                                                  |
|  |  Backblaze B2      |                                                  |
|  |  (user bucket)     |                                                  |
|  +--------------------+                                                  |
|                                                                         |
|  Test containers (not production):                                      |
|  +--------------------+  +--------------------+  +------------------+   |
|  | :companion app     |  | Maestro test flows |  | e2e-runner       |   |
|  +--------------------+  +--------------------+  +------------------+   |
|                                                                         |
+-------------------------------------------------------------------------+
```

### Production container responsibilities

| Container | Technology | Responsibility |
|-----------|------------|----------------|
| **PhotoStorage App** | Kotlin/Android APK | The entire production application: UI, background upload, local indexing, sharing, recovery, settings. |
| **Android MediaStore** | Android OS | Source of truth for local media metadata and URIs. |
| **Backblaze B2** | Cloud object storage | Destination for uploaded media, thumbnails, shared gallery HTML, diagnostic logs, and the SQLite index backup. |

---

## 4. Component diagram (C4 Level 3)

A high-level view of the major components inside the `:app` container.

```
+--------------------------------------------------------------------------+
|                        PhotoStorage App (:app)                           |
|                                                                          |
|  +----------------+  +----------------+  +-------------------------+    |
|  | UI Layer       |  | Settings /     |  | Application Bootstrap   |    |
|  | (Activities,   |  | PrefsStore     |  | (PhotoBackupApp)        |    |
|  |  Adapters)     |  |                |  |                         |    |
|  +-------+--------+  +-------+--------+  +------------+------------+    |
|          |                   |                        |                 |
|          | observes          | reads/writes           | owns            |
|          v                   v                        v                 |
|  +----------------+  +----------------+  +-------------------------+    |
|  | Gallery        |  | Encrypted      |  | Long-lived Services     |    |
|  | Repository     |  | SharedPrefs    |  | (manual service locator)|    |
|  +-------+--------+  +----------------+  +------------+------------+    |
|          |                                            |                 |
|          | uses                                       | uses            |
|          v                                            v                 |
|  +----------------+                          +---------------------+    |
|  | SQLite         |                          | UploadForeground    |    |
|  | (uploads.db)   |                          | Service             |    |
|  +-------+--------+                          +----------+----------+    |
|          |                                              |               |
|          | reads/writes                                 | triggers      |
|          v                                              v               |
|  +----------------+  +----------------+  +-------------------------+   |
|  | DAOs           |  | WorkManager    |  | UploadWorker            |   |
|  | (UploadDao,    |  | Workers        |  | (upload engine)         |   |
|  |  ShareLinkDao, |  | (IndexSync,    |  |                         |   |
|  |  ShareGallery) |  |  NightlyScan,  |  |                         |   |
|  +----------------+  |  LocalDelete,  |  +------------+------------+   |
|                      |  SoftDelete)   |               |                |
|                      +-------+--------+               | uses           |
|                              |                        v                |
|                              |               +---------------------+   |
|                              |               | Network Layer       |   |
|                              |               | (S3ClientFactory,   |   |
|                              |               |  S3Uploader)        |   |
|                              |               +------------+--------+   |
|                              |                            |            |
|                              |                            | HTTPS      |
|                              |                            v            |
|                              |               +---------------------+   |
|                              |               | Backblaze B2        |   |
|                              |               +---------------------+   |
|                              |                                         |
|                              v                                         |
|               +-------------------------+                              |
|               | Local Media Scanner     |                              |
|               | (MediaStoreScanner,     |                              |
|               |  MediaStoreQuery)       |                              |
|               +-------------------------+                              |
|                                                                          |
+--------------------------------------------------------------------------+
```

### Component descriptions

| Component                     | Key files                                                                                                                              | Responsibility                                                                                                                                                                            |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Application Bootstrap**     | `PhotoBackupApp.kt`                                                                                                                    | Manual service locator. Creates `PrefsStore`, `UploadDatabase`, `GalleryRepository`, and lazily creates cloud-scoped objects after login. Schedules periodic WorkManager jobs on startup. |
| **UI Layer**                  | `MainActivity.kt`, `ui/*.kt`                                                                                                           | Activities render screens, request permissions, and launch coroutines. No `ViewModel`s.                                                                                                   |
| **Settings / Preferences**    | `PrefsStore.kt`                                                                                                                        | Encrypted key/value store for credentials, feature toggles, upload mode, video settings, local-delete strategy, and diagnostics state.                                                    |
| **Gallery Repository**        | `gallery/GalleryRepository.kt`                                                                                                         | Builds local/cloud/merged views by joining SQLite records with MediaStore data. Exposes Kotlin `Flow` for UI observation.                                                                 |
| **SQLite Database**           | `data/UploadDatabase.kt`, `data/*Dao.kt`                                                                                               | Raw SQLite database (`uploads.db`) storing upload records, share links, and share galleries.                                                                                              |
| **Local Media Scanner**       | `media/MediaStoreScanner.kt`, `data/MediaStoreQuery.kt`                                                                                | Queries the Android `MediaStore` for photos and videos.                                                                                                                                   |
| **Foreground Upload Service** | `service/UploadForegroundService.kt`                                                                                                   | Long-running service that observes MediaStore changes, debounces them, enqueues new items, and triggers uploads.                                                                          |
| **WorkManager Workers**       | `worker/*.kt`                                                                                                                          | Scheduled/one-shot background jobs: nightly scan, index backup, initial backfill, local-delete review, and soft-delete cleanup.                                                           |
| **Upload Engine**             | `worker/UploadWorker.kt`                                                                                                               | Plain Kotlin class (not a WorkManager worker) that processes the pending queue, generates thumbnails, optionally transcodes video, uploads to B2, and handles retries.                    |
| **Network Layer**             | `b2/S3ClientFactory.kt`, `b2/S3Uploader.kt`, `b2/S3Config.kt`, `b2/S3KeyBuilder.kt`                                                    | Builds the AWS S3 client for B2, performs S3 operations, and generates date-based object keys.                                                                                            |
| **Thumbnail Pipeline**        | `thumbnail/ThumbnailGen.kt`, `thumbnail/ThumbnailCacheFactory.kt`, `thumbnail/B2ThumbnailFetcher.kt`, `thumbnail/VideoFrameFetcher.kt` | Generates WebP thumbnails, caches them with Coil, and downloads cloud thumbnails on demand.                                                                                               |
| **Video Pipeline**            | `media/Transcoder.kt`                                                                                                                  | Compresses long videos using AndroidX Media3 Transformer.                                                                                                                                 |
| **Sharing**                   | `share/ShareLinkService.kt`, `share/ShareGalleryService.kt`, `share/ShareGalleryHtmlGenerator.kt`                                      | Creates presigned single-item links and multi-item gallery HTML pages.                                                                                                                    |
| **Deletion**                  | `gallery/DeletionEngine.kt`, `worker/LocalDeleteWorker.kt`, `worker/SoftDeleteCleanupWorker.kt`                                        | Handles manual delete-by-mode, automatic local-storage cleanup suggestions, and final cleanup of soft-deleted records.                                                                    |
| **Index Recovery**            | `recovery/IndexRecoveryService.kt`, `worker/IndexSyncWorker.kt`                                                                        | Backs up the SQLite DB to B2 and restores it on reinstall.                                                                                                                                |
| **Notifications**             | `notification/UploadNotificationManager.kt`, `notification/NotificationChannels.kt`                                                    | Foreground, progress, completion, auth failure, permanent failure, and local-delete-review notifications.                                                                                 |
| **Cost / Diagnostics**        | `cost/CostDashboardService.kt`, `cost/B2Pricing.kt`, `data/FileLogger.kt`                                                              | Estimates B2 cost and writes on-device diagnostic logs.                                                                                                                                   |

---

## 5. Data flows

### 5.1 New photo taken — auto-upload flow

```
Camera app
    |
    | writes image
    v
MediaStore (system)
    |
    | onChange event
    v
UploadForegroundService.ContentObserver
    |
    | debounce 3s
    v
MediaStoreScanner.scanImages(since = lastScanTimestamp)
    |
    v
DuplicateDetector.isDuplicate()
    |
    | new item
    v
UploadDao.insert(UploadRecord STATUS_PENDING)
    |
    v
UploadWorker.processQueue()
    |
    +--> UploadModeGate decides: upload now / defer
    |
    v
processPhotoRecord()
    |
    +--> read bytes from MediaStore URI
    +--> S3Uploader.upload(photos/YYYY/MM/DD/IMG.jpg)
    +--> ThumbnailGen.createWebPThumbnail()
    +--> S3Uploader.upload(thumbnails/YYYY/MM/DD/IMG.webp)
    |
    v
UploadDao.setUploadedPaths(STATUS_UPLOADED)
    |
    v
GalleryRepository.invalidate()  --> GalleryActivity refreshes
```

### 5.2 Manual batch upload from gallery

```
GalleryActivity
    |
    | user selects LocalOnly items + taps Upload
    v
GalleryActivity.onUploadSelected()
    |
    +--> insert or reset UploadRecord rows to STATUS_PENDING
    v
UploadForegroundService.processQueueNow()
    |
    v
UploadWorker.processQueue()  (same engine as auto-upload)
```

### 5.3 Nightly scan + upload

```
WorkManager (scheduled at 2 AM daily)
    |
    v
NightlyScanWorker.doWork()
    |
    +--> MediaStoreScanner.scanImages / scanVideos
    +--> DuplicateDetector.isDuplicate
    +--> UploadDao.insert(STATUS_PENDING)
    |
    v
UploadWorker.processQueue()
```

### 5.4 Index backup and recovery

**Backup:**

```
WorkManager (scheduled at 3 AM daily)
    |
    v
IndexSyncWorker.doWork()
    |
    +--> copy uploads.db to cache
    +--> compute SHA-256
    +--> if changed: S3Uploader.upload(index/photo-storage-index.sqlite)
    +--> update lastSyncedIndexHash / lastIndexSyncAt
```

**Restore:**

```
MainActivity routes to IndexRecoveryActivity
    |
    v
IndexRecoveryService.hasRemoteIndex()
    |
    +--> S3Uploader.headObject(index/photo-storage-index.sqlite)
    |
    v
IndexRecoveryService.downloadAndRestore()
    |
    +--> S3Uploader.downloadObject(...) to cache
    +--> validate table/schema version
    +--> close local DB, overwrite uploads.db
    +--> reconcile localPresent flags against MediaStore
```

### 5.5 Sharing flow

**Single item:**

```
GalleryActivity
    |
    | selects cloud-backed item + Share link
    v
ShareLinkDialog picks TTL
    |
    v
ShareLinkService.createLink()
    |
    +--> S3Uploader.presignGetUrl(photoB2Path, ttl)
    +--> ShareLinkDao.insert(record)
    |
    v
clipboard + share chooser
```

**Gallery of multiple items:**

```
GalleryActivity
    |
    | selects multiple cloud-backed items
    v
ShareGalleryService.createGalleryLink()
    |
    +--> presign URLs for each full + thumbnail
    +--> ShareGalleryHtmlGenerator.generate HTML from template
    +--> S3Uploader.upload(shares/gallery-${UUID}.html)
    +--> presign URL for HTML page
    +--> ShareGalleryDao.insert(record)
```

### 5.6 Deletion flows

**Manual delete from gallery (merged mode):**

```
GalleryActivity
    |
    | selects items + Delete
    v
DeletionEngine.deleteBatch()
    |
    +--> collect local URIs
    +--> MediaStore.createDeleteRequest() IntentSender
    +--> user approves
    +--> delete B2 objects (photo + thumbnail)
    +--> softDeleteCloud / hardDelete DB record
```

**Automatic local-storage cleanup:**

```
WorkManager (daily at 7 PM)
    |
    v
LocalDeleteWorker.doWork()
    |
    +--> evaluate strategy (never / immediate / after_days / after_count)
    +--> mark records pendingLocalDelete = 1
    +--> show notification
    |
    v
User opens LocalDeleteReviewActivity
    |
    +--> reviews/unchecks items
    +--> confirms -> MediaStore.createDeleteRequest()
    +--> on approval: set localPresent = false
```

---

## 6. UploadRecord state machine

The `uploads` table is the central state machine for backup progress.

```
                    +-------------+
                    |   pending   |
                    +------+------+
                           |
                           | UploadWorker picks it up
                           v
                    +-------------+
                    |  uploading  |
                    +------+------+
                           |
           +---------------+---------------+
           |                               |
           | success                       | failure
           v                               v
    +-------------+                +-------------+
    |   uploaded  |                |    failed   |
    +------+------+                +------+------+
           |                              |
           |                              | retryCount < 5
           |                              v
           |                       +-------------+
           |                       |  pending    | (with nextRetryAt)
           |                       +-------------+
           |
           | user/cloud delete
           v
    +-------------+
    | cloud_deleted |
    +------+------+
           |
           | > 24 hours
           v
    +-------------+
    |  hard-deleted |
    +-------------+
```

Special flags that overlay the state machine:

- `localPresent` — whether the original file still exists on the device.
- `pendingLocalDelete` — item has been suggested for local deletion and is awaiting user review.
- `compressed` — the uploaded video was a transcoded copy, not the original.

---

## 7. Key architectural decisions

| Decision | Rationale |
|----------|-----------|
| **Backblaze B2 via S3-compatible API** | Low-cost object storage; S3 semantics make it portable to other S3-compatible providers. |
| **AWS SDK for Kotlin** | Coroutine-native, but requires `checksumAlgorithm = null` because B2 rejects AWS's default CRC32 headers. |
| **Raw SQLite instead of Room** | Intentionally small dependency surface; schema is simple and the team preferred explicit SQL. |
| **No DI framework** | Manual service location in `PhotoBackupApp` keeps build times and APK size down. |
| **No ViewModel / single-Activity architecture** | Traditional multi-Activity app; business logic lives in workers/services/repositories. |
| **Foreground service + WorkManager hybrid** | Foreground service reacts immediately to new photos; WorkManager handles scheduled bulk/backup work. |
| **Local SQLite DB as source of truth** | The app knows what is backed up without listing the entire B2 bucket. The DB itself is backed up to B2 for recovery. |
| **SHA-256 + filename/size/date deduplication** | Filename/size/date is fast; SHA-256 catches renames and collisions. |
| **WebP thumbnails at 200x200** | Small, efficient thumbnails that load quickly in the gallery grid. |
| **Presigned URLs for sharing** | No backend server needed; B2 handles authentication and expiry. |

---

## 8. Security and data handling

| Concern | Handling |
|---------|----------|
| Cloud credentials | Stored in `EncryptedSharedPreferences` (AES256_GCM). Static B2 keys; no rotation flow. |
| Network traffic | `usesCleartextTraffic="false"`; all S3 traffic is HTTPS. |
| Metadata privacy | Filenames, sizes, dates, and the full local index are uploaded to the user's own B2 bucket. |
| Local delete | System `MediaStore.createDeleteRequest()` requires explicit user approval. |
| Share links | Time-limited presigned URLs (1 hour / 1 day / 1 week). |
| Release signing | Environment-driven (`KEYSTORE_PATH`, `KEYSTORE_PASSWORD`, etc.) in CI. |
| Crash reporting | Firebase Crashlytics only; no analytics event logging. |

---

## 9. Scalability, limits, and operational notes

| Area | Limit / behavior |
|------|------------------|
| **Concurrent uploads** | The foreground service serializes uploads with a `Mutex`; only one `UploadWorker.processQueue()` runs at a time. |
| **Retry policy** | Exponential backoff: `2^retryCount` minutes, max 5 attempts. Auth failures are immediately permanent. |
| **Video compression** | Media3 Transformer runs on the main looper thread but the heavy work is native; output goes to `cacheDir/transcoded`. |
| **Thumbnail cache** | Coil disk cache capped at 200 MB; memory cache at 20% of app memory. |
| **Index backup frequency** | Daily at 3 AM, only if the DB SHA-256 changed. |
| **Gallery performance** | `GalleryRepository` caches per-view-mode results and invalidates on upload/delete/setting changes. |
| **Bucket filtering** | The user can restrict backup to selected albums; changing the selection clears the pending queue. |
| **Offline behavior** | Pending records stay in SQLite; uploads resume when network constraints and retry timers are satisfied. |

---

## 10. Use-case scenario: repeated `make e2e-baseline` runs

This section walks through a concrete behavior that can be surprising when the
E2E baseline scenario is executed multiple times against the same B2 test
bucket, and explains why the architecture produces that behavior.

### 10.1 Scenario

A developer runs `make e2e-baseline` twice on the same device/emulator and the
same B2 test bucket. After the second run, the bucket contains objects from both
runs even though the same reference images were used.

### 10.2 What the harness does

`e2e-baseline` invokes `e2e-runner/scenarios/photo-upload-baseline.sh`, which
starts by resetting state:

1. `clear_main_app_data()` runs `adb shell pm clear com.hriyaan.photostorage`,
   deleting `uploads.db`, `EncryptedSharedPreferences`, and all app-local state.
2. `wipe_all_test_media()` removes previously injected test media from the
   Android `MediaStore`.
3. The companion app injects new test photos. Each injection is salted uniquely
   with the run tag, so the filenames and content hashes differ from the
   previous run.
4. The Maestro flow logs in fresh and enables auto-upload.
5. The app scans `MediaStore` and uploads everything it sees.

### 10.3 Why files are uploaded again

The app’s deduplication logic is local-only:

- `DuplicateDetector.isDuplicate()` checks `UploadDao.findByFilenameAndSize()`
  and `UploadDao.findBySha256()` — both query the local `uploads` table.
- `UploadWorker.processPhotoRecord()` uploads any `STATUS_PENDING` record via
  `S3Uploader.upload()` without first `HEAD`ing the object in B2.

Because `pm clear` wiped `uploads.db`, the app has no record of prior uploads.
Every item in `MediaStore` is treated as new. Even if the DB had survived, the
companion app salts each run uniquely (`ps_test_<run-id>_...`), so the new items
would still not match existing rows.

The test bucket is not cleaned by the baseline scenario, and
`verify_bucket.py` only flags “unexpected” objects for the *current* run tag,
so previous runs’ objects accumulate silently.

### 10.4 Why the index backup does not prevent this

`IndexSyncWorker` uploads `uploads.db` to B2 once per day at 03:00 local time,
*only if the DB hash changed*. It is a disaster-recovery mechanism for reinstalls
and new devices, not a live deduplication source:

- Recovery is triggered manually through `IndexRecoveryActivity`.
- The baseline Maestro flow does not go through recovery; it logs in fresh and
  starts with an empty DB.
- Uploading the index after every photo upload would mean repeatedly uploading a
  growing DB file. For ~70K photos (~100 GB of originals) the DB is only
  ~22–31 MB, but 70K repeated uploads of it would still be ~1.5 PB of index
  traffic.

### 10.5 Scale and sharding notes

There is no database sharding. The app uses a single `uploads.db` file:

- Design target was ~10K photos.
- At 70K photos the DB is roughly **20–35 MB** depending on whether `sha256` is
  populated.
- The schema leaves the door open to future partitioning by year, but that is
  not implemented.

### 10.6 How to avoid bucket accumulation in tests

The harness-level fixes are simpler than changing production deduplication:

- Run `make e2e-clean-all` before each baseline to delete all `ps_test_*`
  objects from the test bucket.
- Or add a `cleanup_bucket.py --all-test-media` step to
  `photo-upload-baseline.sh` before injecting media.

Changing the app to consult B2 before every upload would contradict the
“local SQLite DB as source of truth” decision and would add cost, latency, and
complexity for a problem that is specific to the test harness.

---

## 11. File map

For the detailed implementation of each component, see:

- `android-app-architecture.md` in this folder — operational piece-by-piece breakdown with line-number references.
- `mvp-architecture.md` — original MVP design notes (now superseded by the implementation).
- `mobile-end-to-end-testing-architecture.md` — test harness design.

---

*Last updated: 2026-07-05*
