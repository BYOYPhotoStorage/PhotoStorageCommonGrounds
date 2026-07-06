# Task 34 — Video Media Support

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Bring videos into the upload pipeline at the *detection* and *scanning* layer. Introduce a unified `MediaItem` type that abstracts photos and videos, a `MediaStoreScanner` that queries both `MediaStore.Images` and `MediaStore.Video`, and a second `ContentObserver` for the video collection. Modify the Phase 2 nightly scan and foreground service to consume the new scanner.

The actual upload pipeline changes (transcode invocation, video thumbnail extraction) belong to Task 36; this task only delivers the data layer and detection wiring.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/media/MediaType.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/media/MediaItem.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/media/MediaStoreScanner.kt` (new — or modify if Phase 2 already created a similar helper at this path)
- `app/src/main/java/com/hriyaan/photostorage/upload/UploadForegroundService.kt` (modify — register/unregister video observer)
- `app/src/main/java/com/hriyaan/photostorage/worker/NightlyScanWorker.kt` (modify — include videos when enabled)

## New types

```kotlin
enum class MediaType { PHOTO, VIDEO;
    fun toDbValue(): String = when (this) { PHOTO -> "photo"; VIDEO -> "video" }
    companion object {
        fun fromDbValue(value: String): MediaType =
            if (value == "video") VIDEO else PHOTO
    }
}

data class MediaItem(
    val uri: Uri,
    val filename: String,
    val size: Long,
    val dateTaken: Long,
    val mediaType: MediaType,
    val durationMs: Long? // null for photos; non-null for videos when MediaStore exposes it
)
```

## `MediaStoreScanner` contract

```kotlin
class MediaStoreScanner(private val context: Context) {
    /** Photos with date_added > since (or all photos if since is null). Ordered DESC by date_added. */
    suspend fun scanImages(since: Long?): List<MediaItem>

    /** Videos with date_added > since (or all if since is null). Empty list if !prefs.videosEnabled. */
    suspend fun scanVideos(since: Long?): List<MediaItem>
}
```

### `scanImages`

Projection: `_ID, DISPLAY_NAME, SIZE, DATE_TAKEN, DATE_ADDED`. Query against `MediaStore.Images.Media.EXTERNAL_CONTENT_URI`. Filter by `DATE_ADDED > since` when `since` is not null. Order by `DATE_ADDED DESC`.

Returned `MediaItem.uri` is `ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id)`. `durationMs` is always null.

If `DATE_TAKEN` is null, fall back to `DATE_ADDED * 1000`.

### `scanVideos`

Reads `(context.applicationContext as PhotoBackupApp).prefsStore.getVideosEnabled()`. If false, return empty list. Do **not** open the cursor.

Projection: `_ID, DISPLAY_NAME, SIZE, DATE_TAKEN, DATE_ADDED, DURATION`. Query against `MediaStore.Video.Media.EXTERNAL_CONTENT_URI`. Same filter and order semantics as `scanImages`. Returned URI uses `MediaStore.Video.Media.EXTERNAL_CONTENT_URI`. `durationMs` comes from `DURATION` (already in milliseconds per MediaStore convention).

If a video row has `SIZE = 0` or null, skip it (corrupted entry).

## `UploadForegroundService` changes

Phase 2 registers a `ContentObserver` on `MediaStore.Images.Media.EXTERNAL_CONTENT_URI`. Phase 4 adds a second observer for videos when `videos_enabled = true`.

```kotlin
class UploadForegroundService : Service() {
    private var imageObserver: ContentObserver? = null
    private var videoObserver: ContentObserver? = null

    override fun onStartCommand(...): Int {
        ...
        registerImageObserver()
        if (prefsStore.getVideosEnabled()) registerVideoObserver()
        listenForVideosEnabledChanges() // see below
        ...
    }

    private fun registerVideoObserver() {
        if (videoObserver != null) return
        val observer = object : ContentObserver(handler) {
            override fun onChange(selfChange: Boolean) {
                debouncedTriggerScan()
            }
        }
        contentResolver.registerContentObserver(
            MediaStore.Video.Media.EXTERNAL_CONTENT_URI,
            true,
            observer
        )
        videoObserver = observer
    }

    private fun unregisterVideoObserver() {
        videoObserver?.let { contentResolver.unregisterContentObserver(it) }
        videoObserver = null
    }
}
```

### Reacting to the videos_enabled toggle

The setting can change at runtime (Task 41 settings screen). The service must register / unregister the video observer accordingly.

```kotlin
private fun listenForVideosEnabledChanges() {
    prefsStore.registerOnChangedListener { key, _ ->
        if (key == "videos_enabled") {
            if (prefsStore.getVideosEnabled()) registerVideoObserver()
            else unregisterVideoObserver()
        }
    }
}
```

If `PrefsStore` does not currently expose `registerOnChangedListener`, add a thin wrapper around `EncryptedSharedPreferences.registerOnSharedPreferenceChangeListener`. This is the only Phase 4 caller, so the API can be scoped.

### Scan triggering

The existing `debouncedTriggerScan` (Phase 2) calls `MediaStoreScanner.scanImages(since)` and enqueues the results. Update it to also call `scanVideos(since)` and enqueue those — both go through the existing dedup + queue insert path. Order of insertion does not matter (queue is processed by `created_at ASC`).

```kotlin
private suspend fun runScan() {
    val since = prefsStore.getLastScanTimestamp()
    val items = scanner.scanImages(since) + scanner.scanVideos(since)
    items.forEach { enqueueIfNew(it) }
    prefsStore.setLastScanTimestamp(System.currentTimeMillis())
}
```

`enqueueIfNew` (Phase 2 helper) needs to set `media_type` on the new row. Pass `item.mediaType.toDbValue()` into the existing `insert` call. The duplicate detector's `(filename, size, dateTaken)` key already partitions photos and videos by filename in practice; SHA-256 is still computed lazily by the worker, not here.

## `NightlyScanWorker` changes

The Phase 2 nightly scan ran the same `scanImages` logic. Phase 4 calls `scanImages` + `scanVideos`. Constraints (Wi-Fi-only respect, network requirement) are unchanged.

```kotlin
override suspend fun doWork(): Result = withContext(Dispatchers.IO) {
    val scanner = MediaStoreScanner(applicationContext)
    val since = prefsStore.getLastScanTimestamp()
    val items = scanner.scanImages(since) + scanner.scanVideos(since)
    items.forEach { enqueueIfNew(it) }
    prefsStore.setLastScanTimestamp(System.currentTimeMillis())
    Result.success()
}
```

`enqueueIfNew` shares its implementation with the foreground-service path; factor it into `MediaStoreScanner` or a sibling helper if not already.

## Implementation notes

- All scanner methods run on `Dispatchers.IO` (`suspend` + `withContext(Dispatchers.IO)`). Cursor handling uses `cursor.use { }`.
- The `READ_MEDIA_VIDEO` permission is declared by Task 42 but requested only when the user toggles `videos_enabled = true` (Task 41 wires the runtime request). If the permission is missing, `scanVideos` returns an empty list and logs at warn level — do not throw.
- Videos and photos share the existing `upload_queue` and `uploads` tables (Task 31's `media_type` column distinguishes them).
- Do not change the Phase 2 `last_scan_timestamp` semantics — it is a single watermark for both media types. Edge case: if videos are toggled on after photos have been scanned past a watermark, the next nightly scan will only see new videos from that point. The first-backup-flow + InitialBackfillWorker (Task 40) is responsible for filling in any historical backlog; the toggle does not backfill on its own.
- `MediaStore.Video.Media.DURATION` is sometimes unset; treat null as "unknown" — `MediaItem.durationMs = null` and let Task 35's transcoder treat unknown duration as over-threshold.

## Constraints

- The image observer's behavior must remain identical to Phase 2 — Phase 4 only adds the video path.
- Do not eagerly compute SHA-256 in the scanner. Hashing happens in the upload worker (Phase 2 dedup path).
- Do not register the video observer when `videos_enabled = false` — this avoids unnecessary work and surface area when the user has explicitly disabled video backup.
- Do not change the B2 path layout in this task — that is owned by Task 36.

## Dependencies (by interface)

- `PrefsStore` (Task 32) — `getVideosEnabled`, `getLastScanTimestamp`, `setLastScanTimestamp`, `registerOnChangedListener`
- `UploadDao` (Task 31) — `insert` (with `media_type`)
- `UploadForegroundService` (Phase 2 Task 13)
- `NightlyScanWorker` (Phase 2 Task 16)

## Acceptance criteria

- [ ] `MediaItem` data class compiles and round-trips through scanner methods.
- [ ] `MediaStoreScanner.scanImages(null)` returns every image MediaStore knows about.
- [ ] `MediaStoreScanner.scanImages(since)` returns only images added after `since`.
- [ ] `MediaStoreScanner.scanVideos` returns empty when `videos_enabled = false`, regardless of MediaStore contents.
- [ ] `MediaStoreScanner.scanVideos(null)` returns every video MediaStore exposes when enabled and the permission is granted.
- [ ] The foreground service registers and unregisters the video observer in response to `videos_enabled` toggles at runtime.
- [ ] `NightlyScanWorker` running with videos enabled enqueues both photos and videos taken since `last_scan_timestamp`.
- [ ] Each enqueued row's `media_type` column matches the source (`photo` or `video`).
- [ ] Without `READ_MEDIA_VIDEO`, `scanVideos` returns an empty list and logs a warning instead of crashing.

## Out of scope

- Video transcoding (Task 35).
- Video thumbnail generation (Task 36 — updates `ThumbnailGenerator`).
- B2 path layout for videos (Task 36).
- Per-video UI affordances in the gallery (Task 36 wires the play-icon overlay).
- Backfilling historical videos when the toggle is turned on later — covered by Task 40's `InitialBackfillWorker` only when triggered, not as a side effect of the toggle.
