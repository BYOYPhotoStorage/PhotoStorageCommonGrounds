# Phase 2 Implementation Overview

This document is the shared brief for every agent working on Phase 2 (Auto-Upload & Resilience). Read it once, then jump to the task spec you've been assigned in [`phase2-tasks/`](./phase2-tasks/).

Source-of-truth references: [`phase2-prd.md`](../phase2-prd.md), [`../architecture/mvp-architecture.md`](../architecture/mvp-architecture.md), and the [`../decisions/`](../decisions/) ADRs.

---

## What we are building

A background upload pipeline that detects new photos automatically, queues them, uploads them, and retries on failure — all without requiring the user to tap each photo.

This phase sits on top of the MVP. All MVP code (Tasks 01–09) is in place and compiles. Phase 2 tasks modify and extend that code.

---

## What this agent needs to build

You have been assigned **one task** from [`phase2-tasks/`](./phase2-tasks/) (numbered 10–19). Each task spec is self-contained: it lists the files you create or modify, the public contract you must expose, the dependencies you may rely on (by interface, not implementation), and the acceptance criteria.

The 10 tasks are:

| #   | Task                                                                            | Owns                                                         |
| --- | ------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| 10  | [Database migration & queue schema](./phase2-tasks/10-database-migration.md)    | DB schema v2, `UploadRecord` v2, `UploadDao` queue methods   |
| 11  | [PrefsStore extensions](./phase2-tasks/11-prefsstore-extensions.md)             | `auto_upload_enabled`, `wifi_only`, `last_scan_timestamp`    |
| 12  | [Notification manager](./phase2-tasks/12-notification-manager.md)               | Channels, helper, icons                                      |
| 13  | [Foreground service & ContentObserver](./phase2-tasks/13-foreground-service.md) | `UploadForegroundService`, observer, debounced enqueue       |
| 14  | [Upload worker & retry](./phase2-tasks/14-upload-worker.md)                     | `UploadWorker`, exponential backoff, resume logic            |
| 15  | [Duplicate detection](./phase2-tasks/15-duplicate-detection.md)                 | `DuplicateDetector`, Level 1 + Level 2                       |
| 16  | [WorkManager nightly scan](./phase2-tasks/16-workmanager-nightly-scan.md)       | `NightlyScanWorker`, scheduling                              |
| 17  | [Boot receiver & manifest](./phase2-tasks/17-boot-receiver.md)                  | `BootCompletedReceiver`, permissions, manifest updates       |
| 18  | [Gallery UI Phase 2](./phase2-tasks/18-gallery-ui-phase2.md)                    | Auto-upload toggle, retry UI, status bar                     |
| 19  | [Integration & wire-up](./phase2-tasks/19-integration.md)                       | `PhotoBackupApp` updates, `MainActivity` routing, smoke test |

### Parallelism rules

**Tasks 10, 11, 12, 15** can run fully in parallel — they touch disjoint files and only define new contracts.

**Tasks 13, 14, 16** can run in parallel with each other once Tasks 10, 11, 12 are complete (or at least their contracts are stable). They consume the DB, Prefs, and notification contracts but do not modify them.

**Task 17** depends on Task 13 (service class must exist) and Task 16 (worker class must exist) for manifest declarations, but the receiver code itself is independent.

**Task 18** depends on Tasks 10 and 11 (needs the new DB queries and Prefs methods).

**Task 19** is last and only does wiring. Start it only after Tasks 10–18 have produced their files.

---

## Architecture summary

```
┌────────────── UI ──────────────────────────────┐
│  GalleryActivity        (Task 18, extends 08)  │
│  MainActivity           (Task 19, extends 09)  │
└────────────────────────────────────────────────┘
                 │
┌────────── Background ──────────────────────────┐
│  UploadForegroundService   (Task 13)           │
│    ├── ContentObserver     (detects new photo) │
│    └── NightlyScanWorker   (Task 16)           │
│  UploadWorker              (Task 14)           │
│    ├── DuplicateDetector   (Task 15)           │
│    ├── S3Uploader          (Task 06, reused)   │
│    ├── ThumbnailGen        (Task 05, reused)   │
│    └── UploadDao           (Task 10)           │
│  BootCompletedReceiver     (Task 17)           │
└────────────────────────────────────────────────┘
                 │
┌────────── Infrastructure ──────────────────────┐
│  UploadDao (v2)            (Task 10)           │
│  PrefsStore (v2)           (Task 11)           │
│  NotificationManager       (Task 12)           │
└────────────────────────────────────────────────┘
```

---

## Project conventions (unchanged from MVP)

- **Language:** Kotlin only.
- **Package root:** `com.hriyaan.photostorage`.
- **Gradle DSL:** Kotlin DSL.
- **Min SDK:** 33. **Target/compile SDK:** 34.
- **Coroutines:** All I/O runs on `Dispatchers.IO`. UI uses `Dispatchers.Main` / `lifecycleScope`.
- **No DI framework.** Construct dependencies manually; share singletons through `PhotoBackupApp`.
- **No Room, no Retrofit, no Hilt/Koin.**
- **Comments:** Avoid them. Use clear names.
- **Errors:** Service-layer methods return `kotlin.Result<T>`; UI translates failures to toasts.

---

## Contract surface

These are the **only** types and signatures other tasks may rely on.

### `data` package (Task 10 — extends Task 02)

```kotlin
// v2 UploadRecord
data class UploadRecord(
    val id: Long = 0L,
    val localUri: String,
    val filename: String,
    val size: Long,
    val dateTaken: Long,
    val photoB2Path: String?,
    val thumbnailB2Path: String?,
    val status: String,
    val uploadedAt: Long?,
    val retryCount: Int = 0,
    val nextRetryAt: Long? = null,
    val sha256: String? = null,
    val createdAt: Long = System.currentTimeMillis()
)

class UploadDao(/* internal */) {
    // Existing MVP methods
    fun insert(record: UploadRecord): Long
    fun updateStatus(id: Long, status: String, uploadedAt: Long? = null): Int
    fun setUploadedPaths(id: Long, photoPath: String, thumbnailPath: String, uploadedAt: Long): Int
    fun findByFilenameAndSize(filename: String, size: Long): UploadRecord?
    fun getAll(): List<UploadRecord>

    // Phase 2 additions
    fun getPendingQueue(): List<UploadRecord>
    fun getFailedRetryable(): List<UploadRecord>
    fun updateRetry(id: Long, retryCount: Int, nextRetryAt: Long?): Int
    fun findBySha256(sha256: String): UploadRecord?
    fun updateSha256(id: Long, sha256: String): Int
}
```

### `data` package (Task 11 — extends Task 03)

```kotlin
class PrefsStore(context: Context) {
    // Existing MVP methods
    fun saveCredentials(creds: B2Credentials)
    fun getCredentials(): B2Credentials?
    fun clearCredentials()
    fun hasCredentials(): Boolean

    // Phase 2 additions
    fun isAutoUploadEnabled(): Boolean
    fun setAutoUploadEnabled(enabled: Boolean)
    fun isWifiOnly(): Boolean
    fun setWifiOnly(enabled: Boolean)
    fun getLastScanTimestamp(): Long
    fun setLastScanTimestamp(timestamp: Long)
}
```

### `notification` package (Task 12 — new)

```kotlin
class UploadNotificationManager(private val context: Context) {
    fun ensureChannels()
    fun showForegroundNotification(pendingCount: Int)
    fun updateProgressNotification(current: Int, total: Int)
    fun showCompletionNotification(count: Int)
    fun showPermanentFailureNotification(failedCount: Int)
    fun showAuthFailureNotification()
    fun cancelProgressNotification()
}
```

### `service` package (Task 13 — new)

```kotlin
class UploadForegroundService : Service() {
    companion object {
        fun start(context: Context)
        fun stop(context: Context)
    }
}
```

### `worker` package (Task 14 — new)

```kotlin
class UploadWorker(
    private val context: Context,
    private val uploadDao: UploadDao,
    private val s3Uploader: S3Uploader,
    private val thumbnailGen: ThumbnailGen,
    private val notificationManager: UploadNotificationManager
) {
    suspend fun processQueue()
    suspend fun processSingle(record: UploadRecord): Result<Unit>
}
```

### `dedup` package (Task 15 — new)

```kotlin
class DuplicateDetector(private val uploadDao: UploadDao) {
    suspend fun isDuplicate(photo: MediaStorePhoto): DuplicateResult
    suspend fun computeSha256(uri: Uri): String
}

sealed class DuplicateResult {
    object NotDuplicate : DuplicateResult()
    data class Duplicate(val reason: String) : DuplicateResult()
}
```

### `worker` package (Task 16 — new)

```kotlin
class NightlyScanWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result
}

object NightlyScanScheduler {
    fun schedule(context: Context)
    fun cancel(context: Context)
}
```

### `receiver` package (Task 17 — new)

```kotlin
class BootCompletedReceiver : BroadcastReceiver()
```

---

## File ownership map

| Path | Owner |
|------|-------|
| `app/src/main/java/com/hriyaan/photostorage/data/UploadRecord.kt` | Task 10 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/data/UploadDatabase.kt` | Task 10 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/data/UploadDao.kt` | Task 10 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/data/PrefsStore.kt` | Task 11 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/notification/UploadNotificationManager.kt` | Task 12 |
| `app/src/main/java/com/hriyaan/photostorage/notification/NotificationChannels.kt` | Task 12 |
| `app/src/main/res/drawable/ic_notification_upload.xml` | Task 12 |
| `app/src/main/java/com/hriyaan/photostorage/service/UploadForegroundService.kt` | Task 13 |
| `app/src/main/java/com/hriyaan/photostorage/worker/UploadWorker.kt` | Task 14 |
| `app/src/main/java/com/hriyaan/photostorage/dedup/DuplicateDetector.kt` | Task 15 |
| `app/src/main/java/com/hriyaan/photostorage/worker/NightlyScanWorker.kt` | Task 16 |
| `app/src/main/java/com/hriyaan/photostorage/worker/NightlyScanScheduler.kt` | Task 16 |
| `app/src/main/java/com/hriyaan/photostorage/receiver/BootCompletedReceiver.kt` | Task 17 |
| `app/src/main/AndroidManifest.xml` | Task 17 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/ui/GalleryActivity.kt` | Task 18 (modifies) |
| `app/src/main/res/layout/activity_gallery.xml` | Task 18 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/PhotoBackupApp.kt` | Task 19 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/MainActivity.kt` | Task 19 (modifies) |
| `app/build.gradle.kts` | Task 19 (modifies — adds WorkManager dependency) |

---

## Key constraints (do not violate)

1. **B2 checksum quirk.** Every `PutObjectRequest` must set `checksumAlgorithm = null` — unchanged from MVP.
2. **Credentials never logged.** Unchanged from MVP.
3. **DB migration.** Task 10 increments `DB_VERSION` to `2` and drops/recreates the `uploads` table in `onUpgrade`. There is no production user data yet, so data loss is acceptable.
4. **Foreground service type.** Use `FOREGROUND_SERVICE_TYPE_DATA_SYNC` (Android 14+ requirement).
5. **WorkManager dependency.** Added in Task 19 via `implementation("androidx.work:work-runtime-ktx:2.9.0")`.
