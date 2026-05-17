# Phase 3 Implementation Overview

This document is the shared brief for every agent working on Phase 3 (Gallery & Sync). Read it once, then jump to the task spec you've been assigned in [`phase3-tasks/`](./phase3-tasks/).

Source-of-truth references: [`phase3-prd.md`](./phase3-prd.md), [`phase2-implementation-overview.md`](./phase2-implementation-overview.md), [`../ideas/project-clarification-qa.md`](../ideas/project-clarification-qa.md).

---

## What we are building

A gallery that knows about both local and cloud photos, deletion that respects the active view mode, an SQLite index that is backed up to B2 nightly, a reinstall-recovery flow that restores that index, and a thumbnail cache that avoids repeated egress.

This phase sits on top of Phase 2. All Phase 2 code (Tasks 10–19) is in place. Phase 3 tasks modify and extend that code.

---

## What this agent needs to build

You have been assigned **one task** from [`phase3-tasks/`](./phase3-tasks/) (numbered 20–30). Each task spec is self-contained: it lists the files you create or modify, the public contract you must expose, the dependencies you may rely on (by interface, not implementation), and the acceptance criteria.

The 11 tasks are:

| #   | Task                                                                                | Owns                                                                  |
| --- | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| 20  | [Database migration v2 → v3](./phase3-tasks/20-database-migration.md)               | `local_present`, `cloud_deleted_at`, new DAO queries                  |
| 21  | [PrefsStore extensions](./phase3-tasks/21-prefsstore-extensions.md)                 | `gallery_view_mode`, `last_synced_index_hash`, `last_index_sync_at`   |
| 22  | [B2 client DELETE + GET extensions](./phase3-tasks/22-b2-client-extensions.md)      | `deleteObject`, `headObject`, `downloadObject`                        |
| 23  | [Gallery repository & data types](./phase3-tasks/23-gallery-repository.md)          | `GalleryItem`, `GalleryViewMode`, `GalleryRepository`                 |
| 24  | [Gallery UI Phase 3](./phase3-tasks/24-gallery-ui-phase3.md)                        | Mode switcher, tile badges, index-sync status, selection mode         |
| 25  | [Deletion engine](./phase3-tasks/25-deletion-engine.md)                             | `DeletionEngine`, mode-aware delete, partial-failure handling         |
| 26  | [Soft-delete cleanup worker](./phase3-tasks/26-softdelete-cleanup-worker.md)        | `SoftDeleteCleanupWorker`, 24h hard-delete pass                       |
| 27  | [SQLite index sync worker](./phase3-tasks/27-index-sync-worker.md)                  | `IndexSyncWorker`, hash gate, WAL checkpoint, manual trigger          |
| 28  | [Reinstall recovery flow](./phase3-tasks/28-reinstall-recovery.md)                  | `IndexRecoveryActivity`, B2 HEAD, validate + swap, catch-up prompt    |
| 29  | [Thumbnail cache](./phase3-tasks/29-thumbnail-cache.md)                             | Coil `ImageLoader` + B2 `Fetcher`, 200 MB disk LRU, prefetch          |
| 30  | [Integration & wire-up](./phase3-tasks/30-integration.md)                           | `PhotoBackupApp` updates, `MainActivity` routing, smoke test          |

### Parallelism rules

**Tasks 20, 21, 22** are foundation and fully parallel — they touch disjoint files and only define new contracts.

**Tasks 23, 25, 26, 27, 29** can run in parallel with each other once Tasks 20–22 contracts are stable. They consume the new DAO / Prefs / B2 contracts but do not modify them.

**Task 24** depends on Task 23 (it consumes `GalleryRepository`) and Task 25 (it dispatches deletes to `DeletionEngine`).

**Task 28** depends on Tasks 20, 21, 22 (DB swap, prefs, B2 client).

**Task 30** is last and only does wiring. Start it only after Tasks 20–29 have produced their files.

---

## Architecture summary

```
┌────────────── UI ──────────────────────────────────────┐
│  GalleryActivity            (Task 24, extends 18)      │
│  IndexRecoveryActivity      (Task 28, new)             │
│  MainActivity               (Task 30, extends 19)      │
└────────────────────────────────────────────────────────┘
                 │
┌──────────── Repository ────────────────────────────────┐
│  GalleryRepository          (Task 23)                  │
│  DeletionEngine             (Task 25)                  │
│  IndexRecoveryService       (Task 28)                  │
│  ThumbnailCache             (Task 29)                  │
└────────────────────────────────────────────────────────┘
                 │
┌────────── Background ──────────────────────────────────┐
│  IndexSyncWorker            (Task 27)                  │
│  SoftDeleteCleanupWorker    (Task 26)                  │
│  UploadForegroundService    (reused from Task 13)      │
│  UploadWorker               (reused from Task 14)      │
│  NightlyScanWorker          (reused from Task 16)      │
└────────────────────────────────────────────────────────┘
                 │
┌────────── Infrastructure ──────────────────────────────┐
│  UploadDao (v3)             (Task 20)                  │
│  PrefsStore (v3)            (Task 21)                  │
│  S3Uploader (DELETE/GET)    (Task 22)                  │
│  UploadNotificationManager  (reused)                   │
└────────────────────────────────────────────────────────┘
```

---

## Project conventions (unchanged from Phase 2)

- **Language:** Kotlin only.
- **Package root:** `com.hriyaan.photostorage`.
- **Gradle DSL:** Kotlin DSL.
- **Min SDK:** 33. **Target/compile SDK:** 34.
- **Coroutines:** All I/O runs on `Dispatchers.IO`. UI uses `Dispatchers.Main` / `lifecycleScope`.
- **No DI framework.** Construct dependencies manually; share singletons through `PhotoBackupApp`.
- **No Room, no Retrofit, no Hilt/Koin.** Continue using `SQLiteOpenHelper` directly. The PRD's "Room migrations" wording refers conceptually to schema migrations — implement them as raw SQL through `onUpgrade`.
- **Comments:** Avoid them. Use clear names.
- **Errors:** Service-layer methods return `kotlin.Result<T>`; UI translates failures to toasts.
- **New dependency allowed:** Coil 2.x (`io.coil-kt:coil`) for thumbnail caching. Added in Task 30.

---

## Contract surface

These are the **only** types and signatures other tasks may rely on.

### `data` package (Task 20 — extends Task 10)

```kotlin
// v3 UploadRecord adds localPresent and cloudDeletedAt
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
    val createdAt: Long = System.currentTimeMillis(),
    val localPresent: Boolean = true,
    val cloudDeletedAt: Long? = null
)

class UploadDao(/* internal */) {
    // All Phase 2 methods unchanged.

    companion object {
        // ...existing constants...
        const val STATUS_CLOUD_DELETED = "cloud_deleted"
    }

    // Phase 3 additions
    fun getCloudView(): List<UploadRecord>
    fun getCloudViewBefore(dateTaken: Long): List<UploadRecord>
    fun getLatestUploadedAt(): Long?
    fun setLocalPresent(id: Long, present: Boolean): Int
    fun softDeleteCloud(id: Long, now: Long): Int
    fun getSoftDeletedOlderThan(cutoff: Long): List<UploadRecord>
    fun hardDelete(id: Long): Int
    fun replaceAll(records: List<UploadRecord>)
}
```

### `data` package (Task 21 — extends Task 11)

```kotlin
class PrefsStore(context: Context) {
    // All Phase 2 methods unchanged.

    fun getGalleryViewMode(): String
    fun setGalleryViewMode(mode: String)
    fun getLastSyncedIndexHash(): String?
    fun setLastSyncedIndexHash(hash: String?)
    fun getLastIndexSyncAt(): Long?
    fun setLastIndexSyncAt(timestamp: Long?)
}
```

### `b2` / `upload` package (Task 22 — extends MVP S3 client)

```kotlin
class S3Uploader(/* existing */) {
    // Existing MVP/Phase 2 upload methods unchanged.

    suspend fun deleteObject(path: String): Result<Unit>
    suspend fun headObject(path: String): Result<Boolean>
    suspend fun downloadObject(path: String, dest: File): Result<Unit>
}
```

### `gallery` package (Task 23 — new)

```kotlin
enum class GalleryViewMode { LOCAL, CLOUD, MERGED }

sealed class GalleryItem {
    abstract val id: String
    abstract val dateTaken: Long
    abstract val filename: String
    abstract val thumbnailSource: ThumbnailSource

    data class LocalOnly(
        override val id: String,
        override val dateTaken: Long,
        override val filename: String,
        override val thumbnailSource: ThumbnailSource,
        val mediaStoreUri: Uri,
        val queuedRecord: UploadRecord?
    ) : GalleryItem()

    data class CloudOnly(
        override val id: String,
        override val dateTaken: Long,
        override val filename: String,
        override val thumbnailSource: ThumbnailSource,
        val uploadRecord: UploadRecord
    ) : GalleryItem()

    data class Synced(
        override val id: String,
        override val dateTaken: Long,
        override val filename: String,
        override val thumbnailSource: ThumbnailSource,
        val mediaStoreUri: Uri,
        val uploadRecord: UploadRecord
    ) : GalleryItem()
}

sealed class ThumbnailSource {
    data class LocalUri(val uri: Uri) : ThumbnailSource()
    data class B2Path(val path: String) : ThumbnailSource()
}

class GalleryRepository(
    private val context: Context,
    private val uploadDao: UploadDao,
    private val prefsStore: PrefsStore
) {
    fun observe(mode: GalleryViewMode): Flow<List<GalleryItem>>
    suspend fun load(mode: GalleryViewMode): List<GalleryItem>
    fun invalidate()
}
```

### `gallery` package (Task 25 — new)

```kotlin
class DeletionEngine(
    private val context: Context,
    private val uploadDao: UploadDao,
    private val s3Uploader: S3Uploader,
    private val galleryRepository: GalleryRepository
) {
    suspend fun delete(item: GalleryItem, mode: GalleryViewMode): DeletionResult
    suspend fun deleteBatch(items: List<GalleryItem>, mode: GalleryViewMode): BatchDeletionResult
}

sealed class DeletionResult {
    object Success : DeletionResult()
    data class Refused(val reason: String) : DeletionResult()
    data class Failed(val cause: Throwable, val partialSuccess: Boolean) : DeletionResult()
}

data class BatchDeletionResult(
    val succeeded: Int,
    val refused: Int,
    val failed: Int,
    val summary: String
)
```

### `worker` package (Tasks 26, 27 — new)

```kotlin
class SoftDeleteCleanupWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result
}

object SoftDeleteCleanupScheduler {
    fun schedule(context: Context)
}

class IndexSyncWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result
}

object IndexSyncScheduler {
    fun schedule(context: Context)
    fun cancel(context: Context)
    fun runNow(context: Context)
}
```

### `recovery` package (Task 28 — new)

```kotlin
class IndexRecoveryActivity : AppCompatActivity()

class IndexRecoveryService(
    private val context: Context,
    private val s3Uploader: S3Uploader,
    private val uploadDao: UploadDao,
    private val prefsStore: PrefsStore
) {
    suspend fun hasRemoteIndex(): Result<RemoteIndexInfo?>
    suspend fun downloadAndRestore(progress: (Float) -> Unit = {}): Result<RestoreOutcome>
    suspend fun reconcileLocalPresent()
}

data class RemoteIndexInfo(val lastModified: Long?)
data class RestoreOutcome(val photoCount: Int, val latestUploadedAt: Long?)
```

### `thumbnail` package (Task 29 — new)

```kotlin
class ThumbnailCacheFactory(
    private val context: Context,
    private val s3Uploader: S3Uploader,
    private val prefsStore: PrefsStore
) {
    fun createImageLoader(): coil.ImageLoader
    fun prefetch(paths: List<String>)
}
```

---

## File ownership map

| Path | Owner |
|------|-------|
| `app/src/main/java/com/hriyaan/photostorage/data/UploadRecord.kt` | Task 20 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/data/UploadDatabase.kt` | Task 20 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/data/UploadDao.kt` | Task 20 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/data/PrefsStore.kt` | Task 21 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/upload/S3Uploader.kt` | Task 22 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/gallery/GalleryItem.kt` | Task 23 |
| `app/src/main/java/com/hriyaan/photostorage/gallery/GalleryRepository.kt` | Task 23 |
| `app/src/main/java/com/hriyaan/photostorage/gallery/GalleryViewMode.kt` | Task 23 |
| `app/src/main/java/com/hriyaan/photostorage/ui/GalleryActivity.kt` | Task 24 (modifies) |
| `app/src/main/res/layout/activity_gallery.xml` | Task 24 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/ui/GalleryAdapter.kt` | Task 24 (modifies) |
| `app/src/main/res/layout/item_gallery_tile.xml` | Task 24 (modifies) |
| `app/src/main/res/values/strings.xml` | Tasks 24, 28 (modify) |
| `app/src/main/java/com/hriyaan/photostorage/gallery/DeletionEngine.kt` | Task 25 |
| `app/src/main/java/com/hriyaan/photostorage/worker/SoftDeleteCleanupWorker.kt` | Task 26 |
| `app/src/main/java/com/hriyaan/photostorage/worker/SoftDeleteCleanupScheduler.kt` | Task 26 |
| `app/src/main/java/com/hriyaan/photostorage/worker/IndexSyncWorker.kt` | Task 27 |
| `app/src/main/java/com/hriyaan/photostorage/worker/IndexSyncScheduler.kt` | Task 27 |
| `app/src/main/java/com/hriyaan/photostorage/recovery/IndexRecoveryActivity.kt` | Task 28 |
| `app/src/main/java/com/hriyaan/photostorage/recovery/IndexRecoveryService.kt` | Task 28 |
| `app/src/main/res/layout/activity_index_recovery.xml` | Task 28 |
| `app/src/main/java/com/hriyaan/photostorage/thumbnail/ThumbnailCacheFactory.kt` | Task 29 |
| `app/src/main/java/com/hriyaan/photostorage/thumbnail/B2ThumbnailFetcher.kt` | Task 29 |
| `app/src/main/java/com/hriyaan/photostorage/PhotoBackupApp.kt` | Task 30 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/MainActivity.kt` | Task 30 (modifies) |
| `app/build.gradle.kts` | Task 30 (modifies — adds Coil dependency) |
| `app/src/main/AndroidManifest.xml` | Task 30 (modifies — declares IndexRecoveryActivity) |

---

## Key constraints (do not violate)

1. **B2 path layout is set by MVP/Q3.** Photos at `photos/YYYY/MM/DD/filename.jpg`, thumbnails at `thumbnails/YYYY/MM/DD/filename.webp`. The new index path is `index/photo-storage-index.sqlite`.
2. **No client-side encryption** for either photos or the index file (PRD open-question #1 was resolved as "no need").
3. **DB migration v2 → v3** drops and recreates the table — Phase 2 has not shipped beyond local development, so existing rows are not preserved (same pattern as Phase 2 Task 10).
4. **Soft-delete window is 24 hours.** Do not extend it (PRD open-question #4 resolved as 24h).
5. **Thumbnail cache cap is 200 MB** (PRD open-question #5 resolved).
6. **Merged-view dedup is "render once"** — a photo present both locally and in cloud renders as `Synced`, not as two tiles (PRD open-question #2 resolved).
7. **Catch-up prompt always asks** — do not persist the last choice across reinstalls (PRD open-question #3 resolved).
8. **Foreign-device index restore uses a time-based merge** (PRD open-question #6): photos with `dateTaken <= max(uploadedAt) of restored index` come from the restored index; later photos come from current MediaStore. `localPresent` is reconciled by Task 28's `reconcileLocalPresent()` after restore.
9. **B2 checksum quirk** still applies to PUT — set `checksumAlgorithm = null` (unchanged from MVP).
10. **Foreground service** continues to be the place that runs the live upload loop. Phase 3 workers are independent of the foreground service.
