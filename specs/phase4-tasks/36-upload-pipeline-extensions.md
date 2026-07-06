# Task 36 — Upload Pipeline Extensions

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Make the upload pipeline aware of three new things: the user-selected upload mode (immediate / scheduled / hybrid), videos as a first-class media type (including the new B2 path layout `videos/...` and `.compressed.mp4`), and the transcoder for videos that require compression. Extracts the upload-now-vs-defer decision into a tiny `UploadModeGate` so it is testable in isolation. Updates the existing thumbnail generator to extract the first frame of videos.

Photo upload behavior is unchanged from Phase 2/3 in every code path except the gate decision.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/upload/UploadModeGate.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/upload/UploadWorker.kt` (modify — gate consultation + video branch)
- `app/src/main/java/com/hriyaan/photostorage/upload/ThumbnailGenerator.kt` (modify — first-frame extraction for videos)

## `UploadModeGate` contract

```kotlin
class UploadModeGate(
    private val context: Context,
    private val prefsStore: PrefsStore
) {
    sealed class Decision {
        object UploadNow : Decision()
        data class Defer(val reason: String, val until: Long?) : Decision()
    }

    /** Reads upload_mode, wifi_only_uploads, and current network type. Returns a decision. */
    fun decide(now: Long = System.currentTimeMillis()): Decision
}
```

### Decision table

Inputs:
- `mode` = `prefsStore.getUploadMode()` — `immediate` / `scheduled` / `hybrid`
- `wifiOnly` = `prefsStore.isWifiOnly()` (existing Phase 2 method)
- `unmetered` = current default-network's `NET_CAPABILITY_NOT_METERED` (via `ConnectivityManager.activeNetwork` / `NetworkCapabilities`); `false` if no network

| `mode` | `wifiOnly` | `unmetered` | Decision |
|--------|-----------|-------------|----------|
| `immediate` | false | true / false | `UploadNow` |
| `immediate` | true | true | `UploadNow` |
| `immediate` | true | false | `Defer("Waiting for Wi-Fi", null)` |
| `scheduled` | * | * | `Defer("Scheduled batch only", nextNightlyWindow(now))` |
| `hybrid` | false | true | `UploadNow` |
| `hybrid` | false | false | `Defer("Waiting for Wi-Fi", null)` |
| `hybrid` | true | true | `UploadNow` |
| `hybrid` | true | false | `Defer("Waiting for Wi-Fi", null)` |

`nextNightlyWindow(now)` returns the next 02:00 local-time timestamp (matches Phase 2 nightly scan window — the scheduled batch leverages the same wake-up). Implementation can borrow from `IndexSyncScheduler.msUntilNext` in Phase 3 Task 27.

### Read-only

`UploadModeGate.decide` MUST NOT mutate any state. It is a pure function over its three inputs. Callers (the worker) decide what to do with the decision.

## `UploadWorker` modifications

Phase 2's worker loop pulls one `pending` row, uploads it, marks `uploaded`. Phase 4 adds three hooks:

### Hook 1 — gate consultation

At the top of the worker tick, consult the gate:

```kotlin
val gate = UploadModeGate(applicationContext, prefsStore)
when (val decision = gate.decide()) {
    is UploadModeGate.Decision.UploadNow -> { /* proceed */ }
    is UploadModeGate.Decision.Defer -> {
        notificationManager.updateForDeferred(decision.reason, queueDepth)
        return@withContext Result.success() // exit; next tick (observer fire or nightly) will re-evaluate
    }
}
```

The `scheduled` mode's `Defer` is suppressed during the nightly window (02:00–04:00 local) — the worker checks `now in [02:00, 04:00]` and proceeds even if `mode == "scheduled"`. This is how the nightly batch drains the queue. The check lives in the worker, not the gate (the gate is pure).

```kotlin
private fun isInNightlyWindow(now: Long): Boolean {
    val cal = Calendar.getInstance().apply { timeInMillis = now }
    val hour = cal.get(Calendar.HOUR_OF_DAY)
    return hour in 2..3 // 02:00:00 – 03:59:59
}

// Modified gate check:
val effectiveDecision = if (mode == "scheduled" && isInNightlyWindow(System.currentTimeMillis())) {
    UploadModeGate.Decision.UploadNow
} else {
    gate.decide()
}
```

### Hook 2 — video branch

For each queued row, decide the upload shape based on `mediaType` and the video preferences:

```kotlin
suspend fun processRow(row: UploadRecord) {
    setStatus(row.id, "uploading")
    try {
        when (row.mediaType) {
            "photo" -> processPhoto(row)       // existing path
            "video" -> processVideo(row)       // new
            else -> throw IllegalStateException("Unknown media_type: ${row.mediaType}")
        }
        markUploaded(row.id)
    } catch (t: Throwable) {
        recordFailure(row, t)
    }
}
```

#### `processVideo` flow

1. Resolve the video duration (`MediaMetadataRetriever` against the local URI).
2. Determine whether to compress:
   ```kotlin
   val mode = prefsStore.getVideoQualityMode()
   val thresholdMin = prefsStore.getVideoDurationThresholdMinutes()
   val compress = when (mode) {
       "original" -> false
       "compressed" -> true
       "duration_based" -> {
           val durationMs = row.durationMs ?: extractedDuration ?: Long.MAX_VALUE
           durationMs > thresholdMin * 60_000L
       }
       else -> false
   }
   ```
3. Generate the video thumbnail (Hook 3 below).
4. Build B2 paths:
   - Photo (unchanged): `photos/YYYY/MM/DD/{filename}`, `thumbnails/YYYY/MM/DD/{filename}.webp`
   - Video original: `videos/YYYY/MM/DD/{filename}`
   - Video compressed: `videos/YYYY/MM/DD/{filename.basename}.compressed.mp4`
   - Video thumbnail (any case): `thumbnails/YYYY/MM/DD/{filename}.webp`
5. If `compress == false`, upload the original via the existing PUT path.
6. If `compress == true`:
   - Build the output file: `cacheDir/transcoded/{filename}.compressed.mp4`
   - Call `Transcoder.transcode(input, output, targetResolution, progress)`.
   - On `Result.failure`:
     - Log at warn level
     - Fall back to uploading the original (set `compressed = false` in the row update)
     - Increment a worker-local fallback counter for the foreground notification (optional: surface a one-line toast on completion)
   - On `Result.success`:
     - Upload the transcoded file to the compressed B2 path
     - Update the row with `compressed = true`, `photo_b2_path = compressedPath`, `original_path_b2 = null`
     - Delete the transcoded file from cache after successful upload
7. Upload the thumbnail to its B2 path.
8. Mark row uploaded.

`processVideo` writes `compressed` and `original_path_b2` correctly:
- Compression succeeded: `compressed = 1`, `photo_b2_path = videos/.../filename.compressed.mp4`, `original_path_b2 = null` (original never uploaded; saves space).
- Compression skipped or failed: `compressed = 0`, `photo_b2_path = videos/.../filename.mp4`, `original_path_b2 = null`.

`original_path_b2` is reserved for a future Phase where we may also upload originals alongside compressed copies. Phase 4 always leaves it null.

### Hook 3 — video thumbnail in `ThumbnailGenerator`

Extend the existing `ThumbnailGenerator` (Phase 2/MVP) with a video branch:

```kotlin
class ThumbnailGenerator(private val context: Context) {
    /** Existing photo path — unchanged. */
    fun generateForPhoto(uri: Uri, dest: File): Result<Unit>

    /** New: extracts first frame of a video as WebP. */
    fun generateForVideo(uri: Uri, dest: File): Result<Unit>
}
```

`generateForVideo`:

```kotlin
fun generateForVideo(uri: Uri, dest: File): Result<Unit> = runCatching {
    val retriever = MediaMetadataRetriever()
    try {
        retriever.setDataSource(context, uri)
        val bitmap = retriever.getFrameAtTime(0L, MediaMetadataRetriever.OPTION_CLOSEST_SYNC)
            ?: throw IOException("No frame at time 0")
        val resized = resizeToThumbnail(bitmap) // same logic as photo path
        FileOutputStream(dest).use { out ->
            resized.compress(Bitmap.CompressFormat.WEBP_LOSSY, 80, out)
        }
        bitmap.recycle()
        resized.recycle()
    } finally {
        runCatching { retriever.release() }
    }
}
```

`resizeToThumbnail` is the existing helper from the photo path — reuse it as-is. The output dimensions and format (WebP, ~80 quality) match what Phase 2 already produces for photos so the gallery looks consistent.

### Failure handling, retry, and dedup

- Retry / backoff semantics are unchanged. Both photo and video paths increment `retry_count` on failure and reschedule via `next_retry_at`.
- A transcode failure is *not* a retryable error by itself — the fallback path (upload original) handles it. Only B2 PUT failures retry.
- The existing dedup check (filename + size + date_taken, then SHA-256) runs against both media types. Videos rarely collide on natural keys; SHA-256 covers it when they do.

### Notification updates

The foreground notification text reflects the active mode:

| Condition | Notification body |
|-----------|-------------------|
| `mode = immediate`, draining | "Backing up — X pending" (unchanged) |
| `mode = scheduled`, queue non-empty, outside window | "Photo backup is active — uploads run nightly. X pending." |
| `mode = scheduled`, queue non-empty, inside window | "Backing up nightly batch — X pending" |
| `mode = hybrid`, on mobile data, queue non-empty | "Waiting for Wi-Fi to upload — X pending" |
| `mode = hybrid`, on Wi-Fi | same as `immediate` |
| All clear | "Photo backup is active" |

`UploadNotificationManager` (existing helper) gains a `updateForDeferred(reason, queueDepth)` method; reuse the existing `update(text)` if that is cleaner. Either way, do not introduce a second notification channel.

## Implementation notes

- The worker is single-concurrency (default from Phase 2). Do not bump it for videos — long transcodes simply mean photos wait. This is intentional (PRD constraint #9).
- After a successful video upload, always delete the transcoded file from `cacheDir/transcoded/` in a `finally` block. Cache pressure can also evict it later, but explicit cleanup is the contract.
- The local URI for a queued video is `content://media/external/video/media/{id}`, retrievable through the MediaStore content URI stored in `local_uri`.
- The retry / backoff policy still caps at 5 retries (Phase 2 constant). After permanent failure, the user can discard via the existing failed-uploads affordance.
- `B2 checksum quirk`: photo and video PUT calls both set `checksumAlgorithm = null`. Phase 4 does not change this.
- The thumbnail extraction may return a portrait/landscape bitmap depending on the video's display rotation; honor the rotation metadata so thumbnails are not sideways. `MediaMetadataRetriever.getFrameAtTime` already rotates if `OPTION_CLOSEST_SYNC` is used.

## Constraints

- B2 path layout changes ONLY for videos: `videos/YYYY/MM/DD/...`. Photos stay at `photos/YYYY/MM/DD/...`.
- Thumbnails always live under `thumbnails/YYYY/MM/DD/...` regardless of media type (consistent paths for Phase 3 Cloud-view thumbnail fetcher).
- `UploadModeGate.decide` is pure / read-only. No `PrefsStore` writes from inside it.
- The worker NEVER blocks on the gate — when the gate returns `Defer`, the worker exits cleanly. The next observer fire / WorkManager wake-up re-evaluates.
- No new manifest permissions in this task. (`READ_MEDIA_VIDEO` is declared by Task 42.)

## Dependencies (by interface)

- `PrefsStore` (Task 32) — `getUploadMode`, `getVideoQualityMode`, `getVideoDurationThresholdMinutes`, `getVideoTargetResolution`, `isWifiOnly`
- `UploadDao` (Task 31) — existing `setStatus`, `markUploaded` plus the v4 columns
- `S3Uploader` (Task 33 inherits) — existing PUT path
- `Transcoder` (Task 35) — `transcode`
- `MediaType` enum (Task 34) — to read `row.mediaType`
- `UploadForegroundService` (Phase 2) for notification updates
- `ConnectivityManager` — for unmetered detection inside the gate

## Acceptance criteria

- [ ] `UploadModeGate.decide` returns the correct `Decision` for every row in the decision table above, verified via unit-style tests or manual exercise of all combinations.
- [ ] With `upload_mode = immediate` and the device on Wi-Fi, taking a photo uploads it within 30 seconds (Phase 2 baseline preserved).
- [ ] With `upload_mode = scheduled`, photos taken during the day enqueue but do not upload until the 02:00 nightly window; at 02:00 the queue drains.
- [ ] With `upload_mode = hybrid` on mobile data, photos enqueue but do not upload; switching to Wi-Fi drains the queue.
- [ ] With `video_quality_mode = original`, a recorded video uploads at full size to `videos/YYYY/MM/DD/{name}.mp4`.
- [ ] With `video_quality_mode = compressed`, a recorded video transcodes and uploads to `videos/YYYY/MM/DD/{name}.compressed.mp4`.
- [ ] With `video_quality_mode = duration_based` and threshold 2 minutes: a 30-second video uploads at original; a 5-minute video transcodes.
- [ ] Transcoder failure falls back to uploading the original; `compressed = 0` on the row.
- [ ] Video thumbnails appear in the gallery (Phase 3 Cloud view) at the expected resolution.
- [ ] The transcoded file in `cacheDir/transcoded/` is deleted after upload completes (success or fallback).
- [ ] Foreground notification text reflects the active upload mode (manual visual check).

## Out of scope

- Concurrent uploads (still 1 worker at a time).
- Resumable / chunked uploads.
- Cellular bandwidth shaping.
- HEVC output, two-pass encoding, multi-resolution variants — Task 35 boundary.
- Per-file user override of compression for a specific video.
