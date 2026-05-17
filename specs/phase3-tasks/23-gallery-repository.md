# Task 23 — Gallery Repository & Data Types

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Create the data layer that powers all three gallery views. Define the `GalleryItem` sealed type, the `GalleryViewMode` enum, and the `GalleryRepository` that returns a `Flow<List<GalleryItem>>` for the active view mode. The repository is the single source of truth for the UI (Task 24) and the deletion engine (Task 25).

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/gallery/GalleryItem.kt`
- `app/src/main/java/com/hriyaan/photostorage/gallery/GalleryViewMode.kt`
- `app/src/main/java/com/hriyaan/photostorage/gallery/GalleryRepository.kt`

## Types

```kotlin
enum class GalleryViewMode(val key: String) {
    LOCAL("local"),
    CLOUD("cloud"),
    MERGED("merged");

    companion object {
        fun fromKey(key: String?): GalleryViewMode = entries.firstOrNull { it.key == key } ?: MERGED
    }
}

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
        val sizeBytes: Long,
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
```

**`id` composition:**
- For `LocalOnly`: `"local:" + mediaStoreUri.toString()`
- For `CloudOnly`: `"cloud:" + uploadRecord.id`
- For `Synced`: `"synced:" + uploadRecord.id`

Stable across reloads so RecyclerView diffing works.

## `GalleryRepository`

```kotlin
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

### `observe(mode)`

- Returns a hot `SharedFlow<List<GalleryItem>>` that emits whenever:
  - `invalidate()` is called
  - The MediaStore changes (registered via the existing Phase 2 `ContentObserver` hook — share the same observer; the repository subscribes to a topic the service publishes to)
- Initial emission is the current snapshot from `load(mode)`
- All work happens on `Dispatchers.IO`
- The flow is scoped to the application (not per-screen); the UI collects it within `lifecycleScope`

### `load(mode)` — Local

```kotlin
1. Query MediaStore.Images.Media.EXTERNAL_CONTENT_URI for:
     _ID, DISPLAY_NAME, DATE_TAKEN, SIZE
   ordered by DATE_TAKEN DESC.
2. For each row, look up an UploadRecord by content URI (or filename+size+date)
   to surface the queue badge in the UI (queued/uploading/failed).
3. Build LocalOnly items with thumbnailSource = LocalUri.
```

### `load(mode)` — Cloud

```kotlin
1. uploadDao.getCloudView() → all uploaded, non-cloud-deleted records
2. For each record:
     - If record.localPresent == true AND the corresponding MediaStore row exists,
       use ThumbnailSource.LocalUri for speed.
     - Otherwise use ThumbnailSource.B2Path(record.thumbnailB2Path).
3. Build CloudOnly items.
```

> **Note:** Even though `localPresent` may be true, the Cloud view still emits the record as `CloudOnly` — the view mode determines the wrapper type, not the underlying state. (Task 24's badge code looks at the variant.)

### `load(mode)` — Merged

```kotlin
1. latestUploadedAt = uploadDao.getLatestUploadedAt() ?: 0L
2. uploaded = uploadDao.getCloudView()                       // ordered by date_taken DESC
3. mediaStoreNewer = MediaStore query WHERE DATE_TAKEN > latestUploadedAt
4. Dedup uploaded vs mediaStoreNewer:
     - Build a Set of "natural keys" from uploaded: (filename, size, dateTaken)
       and (sha256 if present).
     - Drop any mediaStoreNewer entry that matches either key.
5. For each uploaded record:
     - If a matching MediaStore row exists (same content URI OR matching natural key),
       emit Synced(record + mediaStoreUri).
     - Else emit CloudOnly(record).
6. For each remaining MediaStore row, emit LocalOnly.
7. Sort the combined list by dateTaken DESC.
```

> The natural key set is what the user described in PRD open-question #6: the foreign-device index timeline merge falls out of this logic naturally — older cloud rows show up first, newer local rows show up second, and the dedup glues overlapping records into Synced tiles.

### `invalidate()`

- Triggers a re-`load` for the currently observed mode
- Called by:
  - `DeletionEngine` after every successful delete (Task 25)
  - The upload pipeline whenever an upload completes or status changes (the Phase 2 `UploadWorker` should call this after each item — wire this in Task 30)
  - The `ContentObserver` in `UploadForegroundService` after debouncing (wire this in Task 30)

## Implementation notes

- Cache the most recent `load(mode)` result in memory keyed by mode. `invalidate()` clears the cache. A second `observe(mode)` call within the same mode returns the cached snapshot before the next reload completes.
- The MediaStore query in `load` must use a `Cursor` and close it. Use `context.contentResolver.query(...)?.use { ... }`.
- At 10K photos (Q8 scale target), in-memory merge is fine. Do not add pagination in Phase 3.
- The repository is a singleton — instantiate it once in `PhotoBackupApp` (Task 30 wires this).

## Constraints

- No coroutines that outlive the application. All flows are scoped to the application or short-lived scopes.
- Do not return rows where `status = 'cloud_deleted'` from any view mode — the soft-delete window is invisible to the user (Task 26 sweeps them later).
- Merged view dedup is "render once" (PRD constraint #6).
- Items in the upload queue (`status = 'pending' | 'uploading' | 'failed' | 'permanently_failed'`) appear in Local view via `queuedRecord`, not in Cloud view. They appear as `LocalOnly` in Merged view too.

## Dependencies (by interface)

- `UploadDao` (Task 20) — `getCloudView`, `getCloudViewBefore`, `getLatestUploadedAt`, plus existing `getAll`, `findByFilenameAndSize`, `findBySha256`
- `PrefsStore` (Task 21) — not strictly needed inside the repository; the UI reads `gallery_view_mode` and passes it in
- MediaStore — `EXTERNAL_CONTENT_URI`

## Acceptance criteria

- [ ] `observe(LOCAL)` emits all device images, sorted by `date_taken DESC`, none from B2.
- [ ] `observe(CLOUD)` emits only `status='uploaded' AND cloud_deleted_at IS NULL` rows, none from MediaStore directly.
- [ ] `observe(MERGED)` emits a continuous timeline with `Synced`, `CloudOnly`, and `LocalOnly` items as appropriate.
- [ ] Merged view dedup correctly collapses a local + cloud match into a single `Synced` tile.
- [ ] A photo with `status = 'cloud_deleted'` never appears in any view.
- [ ] `invalidate()` triggers a fresh emission to subscribers.
- [ ] `id` strings are stable across reloads (verify by capturing two `load` results and comparing).
- [ ] Code compiles with no new dependencies.

## Out of scope

- Pagination / cursor-based MediaStore reads.
- Date grouping headers (UI concern — Task 24).
- Search / filter.
- Real-time MediaStore observation — Task 23 only consumes the existing Phase 2 observer signal; it does not register a second observer.
