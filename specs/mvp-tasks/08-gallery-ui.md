# Task 08 — Gallery UI & Upload Orchestration

> Read [`../mvp-implementation-overview.md`](../mvp-implementation-overview.md) first.

## Goal

Show a 3-column grid of local photos with upload status overlays. Tap to upload (this is where the upload pipeline is orchestrated end-to-end).

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/ui/GalleryActivity.kt` (overwrites the stub from Task 01)
- `app/src/main/java/com/hriyaan/photostorage/ui/GalleryAdapter.kt`
- `app/src/main/java/com/hriyaan/photostorage/ui/GalleryItem.kt` (data class — may colocate)
- `app/src/main/res/layout/activity_gallery.xml`
- `app/src/main/res/layout/item_gallery.xml`
- `app/src/main/res/drawable/ic_cloud_uploaded.xml` (vector cloud-check icon)
- Permission rationale dialog string in `res/values/strings.xml`

## Layouts

`activity_gallery.xml`:

```
CoordinatorLayout
├── AppBarLayout
│   └── MaterialToolbar (title="Photo Backup")
└── RecyclerView (id=@+id/galleryRecyclerView)
    layout_behavior=appbar_scrolling_view_behavior
```

The RecyclerView uses `GridLayoutManager(context, 3)` and a square-image adapter.

`item_gallery.xml`:

```
FrameLayout (square via aspect-ratio)
├── ImageView                                  id=@+id/photoImage  (centerCrop)
├── ProgressBar (gone by default)              id=@+id/progress    (centered, indeterminate)
└── ImageView (gone by default)                id=@+id/statusIcon  (bottom-end, small cloud icon)
```

## Data model

```kotlin
data class GalleryItem(
    val photo: MediaStorePhoto,
    val record: UploadRecord?  // null => never uploaded; non-null => match by (filename, size)
)
```

## Behavior

### onCreate

1. Check `PhotoPermission.isGranted(this)`. If denied, request via `ActivityResultContracts.RequestPermission`. On grant, proceed; on deny, show toast and finish.
2. Build singletons (use Task 09's app-level singletons once they exist; for parallel dev, construct inline):
   - `mediaStoreQuery = MediaStoreQuery(applicationContext)`
   - `uploadDao = (application as PhotoBackupApp).uploadDatabase.dao`
   - `prefsStore = (application as PhotoBackupApp).prefsStore`
   - `thumbnailGen = ThumbnailGen(applicationContext)`
   - Build `s3Uploader` from `prefsStore.getCredentials()` + `S3ClientFactory`. Cache the instance for the activity's lifetime.
3. Load gallery: see "Load + join" below.

### Load + join

In `Dispatchers.IO`:

```kotlin
val photos = mediaStoreQuery.queryAllPhotos()
val items = photos.map { photo ->
    GalleryItem(
        photo = photo,
        record = uploadDao.findByFilenameAndSize(photo.filename, photo.size)
    )
}
```

Submit `items` to the adapter on the main thread.

### Adapter

- Constructor: `GalleryAdapter(onTap: (GalleryItem) -> Unit, onLongPress: (GalleryItem) -> Unit)`.
- `bind` decides which overlay to show:
  - `record == null` → no status icon, no spinner
  - `record.status == "uploading"` → spinner visible, no icon
  - `record.status == "uploaded"` → cloud icon visible
  - `record.status == "failed"` → small "!" overlay (use a simple text drawable or distinct icon — pick something obvious)
- Photo image loads via Glide:

  ```kotlin
  Glide.with(holder.photoImage)
      .load(item.photo.uri)
      .centerCrop()
      .into(holder.photoImage)
  ```

### Tap → upload orchestration

```kotlin
private fun onTap(item: GalleryItem) {
    if (item.record?.status == UploadDao.STATUS_UPLOADED) {
        Toast.makeText(this, "Already uploaded", Toast.LENGTH_SHORT).show()
        return
    }

    lifecycleScope.launch {
        // 1. Insert/update DB row to status=uploading
        val rowId = withContext(Dispatchers.IO) {
            val existing = uploadDao.findByFilenameAndSize(item.photo.filename, item.photo.size)
            if (existing != null) {
                uploadDao.updateStatus(existing.id, UploadDao.STATUS_UPLOADING); existing.id
            } else {
                uploadDao.insert(
                    UploadRecord(
                        localUri = item.photo.uri.toString(),
                        filename = item.photo.filename,
                        size = item.photo.size,
                        dateTaken = item.photo.dateTakenMs,
                        photoB2Path = null,
                        thumbnailB2Path = null,
                        status = UploadDao.STATUS_UPLOADING,
                        uploadedAt = null
                    )
                )
            }
        }
        refreshItem(item.photo.id)

        runCatching {
            withContext(Dispatchers.IO) {
                val photoKey = S3KeyBuilder.photoKey(item.photo.filename, item.photo.dateTakenMs)
                val thumbKey = S3KeyBuilder.thumbnailKey(item.photo.filename, item.photo.dateTakenMs)

                // photo
                contentResolver.openInputStream(item.photo.uri)!!.use { stream ->
                    s3Uploader.upload(
                        key = photoKey,
                        contentType = "image/jpeg",
                        contentLength = item.photo.size,
                        body = stream.asByteStream()
                    ).getOrThrow()
                }

                // thumbnail
                val thumbBytes = thumbnailGen.createWebPThumbnail(item.photo.uri)
                s3Uploader.upload(
                    key = thumbKey,
                    contentType = "image/webp",
                    contentLength = thumbBytes.size.toLong(),
                    body = ByteStream.fromBytes(thumbBytes)
                ).getOrThrow()

                uploadDao.setUploadedPaths(
                    id = rowId,
                    photoPath = photoKey,
                    thumbnailPath = thumbKey,
                    uploadedAt = System.currentTimeMillis()
                )
            }
        }.onFailure {
            withContext(Dispatchers.IO) {
                uploadDao.updateStatus(rowId, UploadDao.STATUS_FAILED)
            }
            Toast.makeText(this@GalleryActivity, "Upload failed", Toast.LENGTH_SHORT).show()
        }

        refreshItem(item.photo.id)
    }
}
```

`refreshItem(photoId)` re-queries that single row from the DAO and calls `notifyItemChanged(position)`. (For MVP simplicity you may also re-load the whole list — perf is acceptable given the dataset size.)

### Long-press

Show an `AlertDialog` with `filename`, formatted `size`, and formatted `dateTakenMs`. Single "OK" button. No editing.

## Dependencies (by interface — see overview)

- `MediaStoreQuery`, `MediaStorePhoto`, `PhotoPermission` (Task 04)
- `UploadDao`, `UploadRecord`, `UploadDatabase` (Task 02)
- `PrefsStore`, `B2Credentials` (Task 03)
- `ThumbnailGen` (Task 05)
- `S3Config`, `S3ClientFactory`, `S3Uploader`, `S3KeyBuilder` (Task 06)
- `PhotoBackupApp` singletons (Task 09 — until then, construct dependencies inline)

## Constraints

- All DB / network / bitmap work runs in `Dispatchers.IO`. UI updates run on `Dispatchers.Main` (use `lifecycleScope` + `withContext`).
- Do not block the main thread on `MediaStore` queries — first paint should not stall.
- Do not retain large bitmaps in adapter `ViewHolder`s. Glide's cache is fine.
- The adapter must show the *current* status at any moment — i.e., status changes from `uploading` → `uploaded`/`failed` must update the visible cell without scrolling away.
- Do not start uploading the same photo twice if a tap arrives while one is already in flight. (Track in-flight URIs in a `MutableSet<String>`.)
- Close `s3Uploader` (and the underlying `S3Client`) in `onDestroy`.

## Acceptance criteria

- [ ] Activity launches and shows photos in a 3-col grid.
- [ ] If `READ_MEDIA_IMAGES` is denied, the user sees a rationale and the activity does not crash.
- [ ] Tapping a not-yet-uploaded photo: spinner appears → photo + thumbnail land in B2 at the expected keys → cloud icon appears → DB row reflects `uploaded` with both paths set.
- [ ] Tapping an already-uploaded photo shows a toast and does not re-upload.
- [ ] Killing and re-launching the app preserves the cloud-icon state (relies on `findByFilenameAndSize` joining MediaStore and DB on relaunch).
- [ ] Upload failures (e.g., airplane mode) flip the cell to a failure state and tapping again retries.

## Out of scope

- Multi-select.
- Background upload (post-MVP).
- Sharing / viewing the cloud copy.
- Pull-to-refresh.
