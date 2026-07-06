# Phase 4 Implementation Overview

This document is the shared brief for every agent working on Phase 4 (Settings & Polish). Read it once, then jump to the task spec you've been assigned in [`phase4-tasks/`](./phase4-tasks/).

Source-of-truth references: [`phase4-prd.md`](./phase4-prd.md), [`phase3-implementation-overview.md`](./phase3-implementation-overview.md), [`../ideas/project-clarification-qa.md`](../ideas/project-clarification-qa.md).

---

## What we are building

A real first-backup flow (today-only vs entire-history) before auto-upload starts, configurable upload timing (immediate / scheduled / hybrid), full video support with quality and duration rules, configurable local-deletion strategies, a B2 cost dashboard, time-limited share links for individual photos, and a single Settings screen that consolidates every toggle introduced in Phases 2–4.

This phase sits on top of Phase 3. All Phase 3 code (Tasks 20–30) is in place. Phase 4 tasks modify and extend that code; they also touch some Phase 2 files (`UploadWorker`, `UploadForegroundService`, `NightlyScanWorker`) — those changes are owned by exactly one Phase 4 task each, so the linear execution order avoids two Phase 4 tasks touching the same file.

---

## Execution model — strictly sequential

Unlike Phase 3, Phase 4 tasks are intended to be executed by **one agent at a time, in numerical order**. Each task depends only on tasks numbered lower than itself. There is no parallel execution path.

The 12 tasks are:

| #   | Task                                                                          | Owns                                                                    |
| --- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 31  | [Database migration v3 → v4](./phase4-tasks/31-database-migration.md)         | `media_type`, `pending_local_delete`, `compressed`, `share_links` table |
| 32  | [PrefsStore extensions](./phase4-tasks/32-prefsstore-extensions.md)           | All Phase 4 preference keys                                             |
| 33  | [B2 presigned URL extension](./phase4-tasks/33-b2-presigned-url.md)           | `S3Uploader.presignGetUrl`                                              |
| 34  | [Video media support](./phase4-tasks/34-video-media-support.md)               | `MediaType`, `MediaItem`, second `ContentObserver`, video scan          |
| 35  | [Video transcoder](./phase4-tasks/35-video-transcoder.md)                     | `Transcoder` (AndroidX Media3 `Transformer` wrapper)                    |
| 36  | [Upload pipeline extensions](./phase4-tasks/36-upload-pipeline-extensions.md) | `UploadModeGate`, `UploadWorker` video + mode integration               |
| 37  | [Local deletion strategy](./phase4-tasks/37-local-deletion-strategy.md)       | `LocalDeleteWorker`, `LocalDeleteReviewActivity`, batched delete prompt |
| 38  | [Cost dashboard](./phase4-tasks/38-cost-dashboard.md)                         | `B2Pricing`, `CostDashboardService`, `CostDashboardActivity`            |
| 39  | [Sharing](./phase4-tasks/39-sharing.md)                                       | `ShareLinkService`, `ShareLinkDialog`, `ActiveShareLinksActivity`       |
| 40  | [First-backup flow](./phase4-tasks/40-first-backup-flow.md)                   | `FirstBackupActivity`, `InitialBackfillWorker`                          |
| 41  | [Settings screen](./phase4-tasks/41-settings-screen.md)                       | `SettingsActivity` (consolidates Phases 2–4)                            |
| 42  | [Integration & wire-up](./phase4-tasks/42-integration.md)                     | `PhotoBackupApp` updates, `MainActivity` routing, smoke test            |

### Dependency chain

```
31 → 32 → 33 → 34 → 35 → 36 → 37 → 38 → 39 → 40 → 41 → 42
```

Each task only consumes contracts defined by lower-numbered tasks. No task modifies a file owned by a different Phase 4 task — file ownership is disjoint across Phase 4 (see the File ownership map below). The execution chain therefore has no merge conflicts and no waiting points.

### Why sequential (and not parallel like Phase 3)

Phase 4 has tighter coupling at the upload pipeline level than Phase 3 did at the gallery level. Tasks 34–36 in particular form a chain (data types → transcoder → pipeline integration) that benefits from one agent holding the full picture. The remaining tasks could in principle run in parallel after Task 36, but the user-facing surface (settings screen, first-backup flow, integration) needs to know which preferences and services actually exist — sequential execution makes that contract enforcement automatic.

---

## Architecture summary

```
┌────────────── UI ──────────────────────────────────────┐
│  GalleryActivity            (Task 39, extends 24)      │
│  SettingsActivity           (Task 41, new)             │
│  FirstBackupActivity        (Task 40, new)             │
│  CostDashboardActivity      (Task 38, new)             │
│  ActiveShareLinksActivity   (Task 39, new)             │
│  LocalDeleteReviewActivity  (Task 37, new)             │
│  ShareLinkDialog            (Task 39, new)             │
│  MainActivity               (Task 42, extends 30)      │
└────────────────────────────────────────────────────────┘
                 │
┌──────────── Service / Repository ──────────────────────┐
│  CostDashboardService       (Task 38)                  │
│  ShareLinkService           (Task 39)                  │
│  UploadModeGate             (Task 36)                  │
│  Transcoder                 (Task 35)                  │
│  GalleryRepository          (reused from Task 23)      │
│  DeletionEngine             (reused from Task 25)      │
└────────────────────────────────────────────────────────┘
                 │
┌────────── Background ──────────────────────────────────┐
│  UploadWorker               (Task 36, extends 14)      │
│  UploadForegroundService    (Task 34, extends 13)      │
│  NightlyScanWorker          (Task 34, extends 16)      │
│  LocalDeleteWorker          (Task 37, new)             │
│  InitialBackfillWorker      (Task 40, new)             │
│  IndexSyncWorker            (reused from Task 27)      │
│  SoftDeleteCleanupWorker    (reused from Task 26)      │
└────────────────────────────────────────────────────────┘
                 │
┌────────── Infrastructure ──────────────────────────────┐
│  UploadDao (v4)             (Task 31)                  │
│  ShareLinkDao               (Task 31, new)             │
│  PrefsStore (v4)            (Task 32)                  │
│  S3Uploader (presign)       (Task 33)                  │
│  MediaStoreScanner          (Task 34)                  │
│  B2Pricing                  (Task 38, new)             │
└────────────────────────────────────────────────────────┘
```

---

## Project conventions (unchanged from Phase 3)

- **Language:** Kotlin only.
- **Package root:** `com.hriyaan.photostorage`.
- **Gradle DSL:** Kotlin DSL.
- **Min SDK:** 33. **Target/compile SDK:** 34.
- **Coroutines:** All I/O runs on `Dispatchers.IO`. UI uses `Dispatchers.Main` / `lifecycleScope`.
- **No DI framework.** Construct dependencies manually; share singletons through `PhotoBackupApp`.
- **No Room, no Retrofit, no Hilt/Koin.** Continue using `SQLiteOpenHelper` directly.
- **Comments:** Avoid them. Use clear names.
- **Errors:** Service-layer methods return `kotlin.Result<T>`; UI translates failures to toasts.
- **New dependencies allowed in Phase 4 (added in Task 42):**
  - `androidx.media3:media3-transformer` and `androidx.media3:media3-effect` for video transcoding (Task 35)
  - No other new dependencies. Compose is already present (Phase 3); the Settings screen reuses it.

---

## Contract surface

These are the **only** types and signatures other tasks may rely on. Each contract is owned by the task in parentheses; downstream tasks consume by interface, not implementation.

### `data` package (Task 31 — extends Task 20)

```kotlin
// v4 UploadRecord adds mediaType, originalPathB2, pendingLocalDelete, compressed
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
    val cloudDeletedAt: Long? = null,
    val mediaType: String = "photo",         // "photo" | "video"
    val originalPathB2: String? = null,       // for compressed videos, original full-size path
    val pendingLocalDelete: Boolean = false,
    val compressed: Boolean = false
)

class UploadDao {
    // All Phase 3 methods unchanged.

    companion object {
        const val MEDIA_TYPE_PHOTO = "photo"
        const val MEDIA_TYPE_VIDEO = "video"
    }

    // Phase 4 additions
    fun getByMediaType(mediaType: String): List<UploadRecord>
    fun countByMediaType(mediaType: String): Int
    fun sumSizeByMediaType(mediaType: String): Long
    fun markPendingLocalDelete(ids: List<Long>): Int
    fun getPendingLocalDelete(): List<UploadRecord>
    fun getEligibleForLocalDelete(olderThanUploadedAt: Long): List<UploadRecord>
    fun getOldestUploaded(limit: Int): List<UploadRecord>
    fun clearPendingLocalDelete(id: Long): Int
    fun countUploadsSince(timestamp: Long): Int
}

data class ShareLinkRecord(
    val id: Long = 0L,
    val uploadId: Long,
    val photoB2Path: String,
    val url: String,
    val createdAt: Long,
    val expiresAt: Long
)

class ShareLinkDao(/* internal */) {
    fun insert(record: ShareLinkRecord): Long
    fun getActive(now: Long): List<ShareLinkRecord>
    fun getOlderThan(cutoff: Long): List<ShareLinkRecord>
    fun deleteOlderThan(cutoff: Long): Int
}
```

### `data` package (Task 32 — extends Task 21)

```kotlin
class PrefsStore(context: Context) {
    // All Phase 3 methods unchanged.

    // First-backup flow
    fun getFirstBackupScope(): String  // "today" | "all", default "today"
    fun setFirstBackupScope(scope: String)
    fun hasCompletedFirstBackupFlow(): Boolean
    fun setFirstBackupFlowCompleted(completed: Boolean)

    // Videos
    fun getVideosEnabled(): Boolean   // default false
    fun setVideosEnabled(enabled: Boolean)
    fun getVideoQualityMode(): String // "original" | "compressed" | "duration_based"
    fun setVideoQualityMode(mode: String)
    fun getVideoDurationThresholdMinutes(): Int // default 2
    fun setVideoDurationThresholdMinutes(minutes: Int)
    fun getVideoTargetResolution(): String // "720p" | "1080p"
    fun setVideoTargetResolution(resolution: String)

    // Upload mode
    fun getUploadMode(): String // "immediate" | "scheduled" | "hybrid"
    fun setUploadMode(mode: String)

    // Local deletion
    fun getLocalDeleteStrategy(): String // "never" | "immediate" | "after_days" | "after_count"
    fun setLocalDeleteStrategy(strategy: String)
    fun getLocalDeleteDays(): Int
    fun setLocalDeleteDays(days: Int)
    fun getLocalDeleteCount(): Int
    fun setLocalDeleteCount(count: Int)
    fun getLocalDeleteSuppressUntil(): Long?
    fun setLocalDeleteSuppressUntil(timestamp: Long?)
    fun getLastLocalDeleteRunAt(): Long?
    fun setLastLocalDeleteRunAt(timestamp: Long?)
    fun getLocalDeleteDismissStreak(): Int
    fun setLocalDeleteDismissStreak(count: Int)

    // Cost dashboard (rolling monthly egress counter)
    fun getEgressBytesMonth(): Long
    fun setEgressBytesMonth(bytes: Long)
    fun getEgressMonthAnchor(): Long
    fun setEgressMonthAnchor(timestamp: Long)
}
```

### `upload` package (Task 33 — extends Task 22)

```kotlin
class S3Uploader {
    // Existing MVP / Phase 2 / Phase 3 methods unchanged.

    /**
     * Generates a presigned GET URL for the given object path. TTL is enforced by B2.
     * The returned URL is opaque — callers persist or share it but do not parse it.
     */
    suspend fun presignGetUrl(path: String, ttlSeconds: Long): Result<String>
}
```

### `media` package (Task 34 — new)

```kotlin
enum class MediaType { PHOTO, VIDEO }

data class MediaItem(
    val uri: Uri,
    val filename: String,
    val size: Long,
    val dateTaken: Long,
    val mediaType: MediaType,
    val durationMs: Long? // null for photos
)

class MediaStoreScanner(private val context: Context) {
    suspend fun scanImages(since: Long?): List<MediaItem>
    suspend fun scanVideos(since: Long?): List<MediaItem>
}
```

### `media` package (Task 35 — new)

```kotlin
class Transcoder(private val context: Context) {
    suspend fun transcode(
        input: Uri,
        output: File,
        targetResolution: String, // "720p" | "1080p"
        progress: (Float) -> Unit = {}
    ): Result<TranscodeResult>
}

data class TranscodeResult(
    val outputFile: File,
    val outputSizeBytes: Long,
    val durationMs: Long
)
```

### `upload` package (Task 36 — new)

```kotlin
class UploadModeGate(
    private val context: Context,
    private val prefsStore: PrefsStore
) {
    /** Called by UploadWorker before each upload attempt. */
    fun decide(now: Long = System.currentTimeMillis()): Decision

    sealed class Decision {
        object UploadNow : Decision()
        data class Defer(val reason: String, val until: Long?) : Decision()
    }
}
```

### `worker` package (Task 37 — new)

```kotlin
class LocalDeleteWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params)

object LocalDeleteScheduler {
    fun schedule(context: Context)
    fun cancel(context: Context)
    fun runNow(context: Context)
}
```

### `ui` package (Task 37 — new)

```kotlin
class LocalDeleteReviewActivity : AppCompatActivity()
```

### `cost` package (Task 38 — new)

```kotlin
object B2Pricing {
    const val STORAGE_PER_GB_PER_MONTH_USD: Double = 0.006
    const val EGRESS_PER_GB_USD: Double = 0.01
    const val LAST_UPDATED: String = "2026-05-17"
    fun monthlyStorageCostUsd(bytes: Long): Double
    fun egressCostUsd(bytes: Long): Double
}

class CostDashboardService(
    private val uploadDao: UploadDao,
    private val prefsStore: PrefsStore
) {
    suspend fun computeSnapshot(): CostSnapshot
}

data class CostSnapshot(
    val photoCount: Int,
    val videoCount: Int,
    val totalBytes: Long,
    val pendingCount: Int,
    val pendingBytes: Long,
    val permanentlyFailedCount: Int,
    val monthlyStorageCostUsd: Double,
    val monthlyEgressCostUsd: Double?, // null when no egress has been recorded
    val byYear: List<YearBreakdown>
)

data class YearBreakdown(val year: Int, val bytes: Long, val photoCount: Int, val videoCount: Int)

class CostDashboardActivity : AppCompatActivity()
```

### `share` package (Task 39 — new)

```kotlin
enum class ShareLinkTtl(val seconds: Long, val labelRes: Int) {
    ONE_HOUR(3_600L, R.string.share_ttl_1h),
    ONE_DAY(86_400L, R.string.share_ttl_24h),
    ONE_WEEK(604_800L, R.string.share_ttl_7d)
}

class ShareLinkService(
    private val s3Uploader: S3Uploader,
    private val shareLinkDao: ShareLinkDao
) {
    suspend fun createLink(item: GalleryItem, ttl: ShareLinkTtl): Result<ShareLinkRecord>
    suspend fun activeLinks(now: Long = System.currentTimeMillis()): List<ShareLinkRecord>
}

class ShareLinkDialog : DialogFragment()
class ActiveShareLinksActivity : AppCompatActivity()
```

### `ui` / `worker` package (Task 40 — new)

```kotlin
class FirstBackupActivity : AppCompatActivity()
class InitialBackfillWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params)
```

### `ui` package (Task 41 — new)

```kotlin
class SettingsActivity : AppCompatActivity()
```

---

## File ownership map

Each Phase 4 task is the **sole** Phase 4 owner of every file it modifies. Some files are also touched here that originated in Phase 2 / Phase 3 — that is intentional evolution, not a contract change.

| Path | Owner |
|------|-------|
| `app/src/main/java/com/hriyaan/photostorage/data/UploadRecord.kt` | Task 31 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/data/UploadDatabase.kt` | Task 31 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/data/UploadDao.kt` | Task 31 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/data/ShareLinkRecord.kt` | Task 31 (new) |
| `app/src/main/java/com/hriyaan/photostorage/data/ShareLinkDao.kt` | Task 31 (new) |
| `app/src/main/java/com/hriyaan/photostorage/data/PrefsStore.kt` | Task 32 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/upload/S3Uploader.kt` | Task 33 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/media/MediaType.kt` | Task 34 (new) |
| `app/src/main/java/com/hriyaan/photostorage/media/MediaItem.kt` | Task 34 (new) |
| `app/src/main/java/com/hriyaan/photostorage/media/MediaStoreScanner.kt` | Task 34 (new) |
| `app/src/main/java/com/hriyaan/photostorage/upload/UploadForegroundService.kt` | Task 34 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/worker/NightlyScanWorker.kt` | Task 34 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/media/Transcoder.kt` | Task 35 (new) |
| `app/src/main/java/com/hriyaan/photostorage/upload/UploadModeGate.kt` | Task 36 (new) |
| `app/src/main/java/com/hriyaan/photostorage/upload/UploadWorker.kt` | Task 36 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/upload/ThumbnailGenerator.kt` | Task 36 (modifies — video frame support) |
| `app/src/main/java/com/hriyaan/photostorage/worker/LocalDeleteWorker.kt` | Task 37 (new) |
| `app/src/main/java/com/hriyaan/photostorage/worker/LocalDeleteScheduler.kt` | Task 37 (new) |
| `app/src/main/java/com/hriyaan/photostorage/ui/LocalDeleteReviewActivity.kt` | Task 37 (new) |
| `app/src/main/res/layout/activity_local_delete_review.xml` | Task 37 (new) |
| `app/src/main/java/com/hriyaan/photostorage/cost/B2Pricing.kt` | Task 38 (new) |
| `app/src/main/java/com/hriyaan/photostorage/cost/CostDashboardService.kt` | Task 38 (new) |
| `app/src/main/java/com/hriyaan/photostorage/ui/CostDashboardActivity.kt` | Task 38 (new) |
| `app/src/main/res/layout/activity_cost_dashboard.xml` | Task 38 (new) |
| `app/src/main/java/com/hriyaan/photostorage/share/ShareLinkService.kt` | Task 39 (new) |
| `app/src/main/java/com/hriyaan/photostorage/share/ShareLinkTtl.kt` | Task 39 (new) |
| `app/src/main/java/com/hriyaan/photostorage/ui/ShareLinkDialog.kt` | Task 39 (new) |
| `app/src/main/java/com/hriyaan/photostorage/ui/ActiveShareLinksActivity.kt` | Task 39 (new) |
| `app/src/main/res/layout/dialog_share_link.xml` | Task 39 (new) |
| `app/src/main/res/layout/activity_active_share_links.xml` | Task 39 (new) |
| `app/src/main/java/com/hriyaan/photostorage/ui/GalleryActivity.kt` | Task 39 (modifies — long-press menu adds Share link) |
| `app/src/main/java/com/hriyaan/photostorage/ui/FirstBackupActivity.kt` | Task 40 (new) |
| `app/src/main/java/com/hriyaan/photostorage/worker/InitialBackfillWorker.kt` | Task 40 (new) |
| `app/src/main/res/layout/activity_first_backup.xml` | Task 40 (new) |
| `app/src/main/java/com/hriyaan/photostorage/ui/SettingsActivity.kt` | Task 41 (new) |
| `app/src/main/res/values/strings.xml` | Tasks 37, 38, 39, 40, 41 (each adds its own keys; do not edit other tasks' keys) |
| `app/src/main/java/com/hriyaan/photostorage/PhotoBackupApp.kt` | Task 42 (modifies) |
| `app/src/main/java/com/hriyaan/photostorage/MainActivity.kt` | Task 42 (modifies) |
| `app/build.gradle.kts` | Task 42 (modifies — adds AndroidX Media3) |
| `app/src/main/AndroidManifest.xml` | Task 42 (modifies — declares new activities, adds `READ_MEDIA_VIDEO`) |

---

## Key constraints (do not violate)

1. **B2 path layout extends Q3 conservatively.** Photos stay at `photos/YYYY/MM/DD/filename.jpg`; thumbnails at `thumbnails/YYYY/MM/DD/filename.webp`. New video paths: `videos/YYYY/MM/DD/filename.mp4` for originals, `videos/YYYY/MM/DD/filename.compressed.mp4` for transcoded copies. The SQLite index path is unchanged (`index/photo-storage-index.sqlite`).
2. **No client-side encryption** for any media or for share links (PRD Non-Goal).
3. **DB migration v3 → v4** drops and recreates `uploads` (same pattern as Phase 3 Task 20 — pre-distribution, no real user data). `share_links` is created fresh.
4. **Soft-delete window is still 24 hours** (Phase 3 constraint #4). Phase 4 does not change `cloud_deleted_at` semantics.
5. **Thumbnail cache cap is still 200 MB** (Phase 3 constraint #5).
6. **One share link, no revoke.** B2 presigned URLs cannot be revoked once issued — surface TTL clearly to the user and do not pretend otherwise.
7. **Pre-Phase-4 install migration is via defaults.** No prefs migration code is needed — readers return the documented default when a key is absent. `videos_enabled = false`, `first_backup_scope = today`, `upload_mode = immediate`, `local_delete_strategy = never`.
8. **Local deletion is always user-confirmed.** Android 11+ scoped storage prohibits silent deletes of MediaStore-owned files. Task 37 batches confirmations into a single `MediaStore.createDeleteRequest` per day.
9. **Video transcoding never blocks photo uploads.** The pipeline uses a single sequential worker (Phase 2 default); long transcodes simply mean later photos wait their turn. Concurrent worker count remains 1.
10. **B2 checksum quirk** still applies to PUT — set `checksumAlgorithm = null` (unchanged from MVP).
11. **`first_backup_scope = all` only backfills photos**, not videos, even when `videos_enabled = true` (PRD Open Question #8 — documented as current intent; revisit if it surfaces in testing).
12. **`UploadModeGate.decide` is read-only.** It must not mutate `PrefsStore`, MediaStore, or the queue. It returns a decision; the worker acts on it.
