# Task 04 — MediaStore Query

> Read [`../mvp-implementation-overview.md`](../mvp-implementation-overview.md) first.

## Goal

Read the device photo gallery via `MediaStore` and expose a list the gallery UI can render.

## Files you own

- `app/src/main/java/com/photobackup/app/data/MediaStorePhoto.kt` (data class — file may colocate with the query class if you prefer one file)
- `app/src/main/java/com/photobackup/app/data/MediaStoreQuery.kt`
- `app/src/main/java/com/photobackup/app/data/PhotoPermission.kt`

## Public contract

```kotlin
package com.photobackup.app.data

data class MediaStorePhoto(
    val id: Long,
    val uri: Uri,                // content://media/external/images/media/<id>
    val filename: String,
    val size: Long,
    val dateTakenMs: Long
)

class MediaStoreQuery(private val context: Context) {
    fun queryAllPhotos(): List<MediaStorePhoto>
}

object PhotoPermission {
    const val PERMISSION = android.Manifest.permission.READ_MEDIA_IMAGES
    fun isGranted(context: Context): Boolean
}
```

## Implementation

Use the projection from the architecture doc:

```kotlin
val projection = arrayOf(
    MediaStore.Images.Media._ID,
    MediaStore.Images.Media.DISPLAY_NAME,
    MediaStore.Images.Media.SIZE,
    MediaStore.Images.Media.DATE_TAKEN
)

val cursor = context.contentResolver.query(
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
    projection,
    null, null,
    "${MediaStore.Images.Media.DATE_TAKEN} DESC"
) ?: return emptyList()
```

Build URIs with `ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id)`.

`queryAllPhotos` is a blocking call. Callers run it in `Dispatchers.IO`.

`PhotoPermission.isGranted` returns
`ContextCompat.checkSelfPermission(context, PERMISSION) == PackageManager.PERMISSION_GRANTED`.

## Constraints

- `DATE_TAKEN` can be null on some devices — fall back to `0L` when it is. The gallery sorts by it; nulls land at the bottom.
- Do not paginate. The MVP loads everything once; if perf is an issue post-MVP, the architecture's "Risks" section calls out adding `LIMIT/OFFSET`.
- Do not deduplicate here — duplicate detection (filename + size) is the gallery adapter's job (Task 08) using `UploadDao.findByFilenameAndSize`.

## Acceptance criteria

- [ ] On an emulator with several test photos, `queryAllPhotos()` returns one entry per gallery image, sorted newest-first.
- [ ] Each `MediaStorePhoto.uri` opens via `contentResolver.openInputStream(uri)` without exception.
- [ ] When `READ_MEDIA_IMAGES` is denied, `queryAllPhotos()` returns an empty list (the query just yields nothing — no crash). `PhotoPermission.isGranted` returns false in that state.
- [ ] No reference to `READ_EXTERNAL_STORAGE` anywhere — that's the wrong permission on API 33+.

## Out of scope

- Video. (Photos only for MVP.)
- ContentObserver / live updates.
- Pagination.
