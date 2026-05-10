# Task 05 — Thumbnail Generator

> Read [`../mvp-implementation-overview.md`](../mvp-implementation-overview.md) first.

## Goal

Produce a small WebP thumbnail (~20–40 KB) for each photo before upload.

## Files you own

- `app/src/main/java/com/photobackup/app/thumbnail/ThumbnailGen.kt`
- (Optional) `app/src/androidTest/java/com/photobackup/app/thumbnail/ThumbnailGenTest.kt`

## Public contract

```kotlin
package com.photobackup.app.thumbnail

class ThumbnailGen(private val context: Context) {
    /**
     * Decodes [uri] and returns a square WebP-compressed thumbnail no larger than
     * [maxDim] x [maxDim]. Output is typically 20–40 KB.
     *
     * @throws java.io.IOException if the source URI cannot be opened or decoded
     */
    fun createWebPThumbnail(uri: Uri, maxDim: Int = 200): ByteArray
}
```

Returns the WebP-encoded bytes. Caller uploads them as-is via `S3Uploader`.

## Implementation

Use the architecture doc's recipe:

```kotlin
val bitmap = context.contentResolver.loadThumbnail(uri, Size(maxDim, maxDim), null)
val out = ByteArrayOutputStream()
bitmap.compress(Bitmap.CompressFormat.WEBP_LOSSY, 80, out)
return out.toByteArray()
```

`loadThumbnail` is API 29+; we target API 33 so it is always available.

### OOM mitigation

If `loadThumbnail` ever fails or is unavailable on a device, fall back to a manual decode:

```kotlin
val opts = BitmapFactory.Options().apply { inJustDecodeBounds = true }
context.contentResolver.openInputStream(uri).use { BitmapFactory.decodeStream(it, null, opts) }
opts.inSampleSize = calculateInSampleSize(opts, maxDim, maxDim)
opts.inJustDecodeBounds = false
val bitmap = context.contentResolver.openInputStream(uri).use {
    BitmapFactory.decodeStream(it, null, opts)
}
```

…then center-crop / scale to `maxDim` and compress as above.

Run on `Dispatchers.IO`; do not call from the main thread.

## Constraints

- Always `WEBP_LOSSY` at quality `80`. Don't expose configuration knobs in the MVP.
- Recycle bitmaps after compress.
- A single thumbnail call must not allocate more than ~10 MB of bitmap memory — that's why we sample down before scaling for fallback path.

## Acceptance criteria

- [ ] For a 5 MB JPEG, `createWebPThumbnail(uri).size` is between 5 KB and 80 KB.
- [ ] Output decodes back to a non-null `Bitmap` via `BitmapFactory.decodeByteArray`.
- [ ] Output dimensions are `<= 200x200`.
- [ ] No `OutOfMemoryError` when called on a 50 MP source image.

## Out of scope

- HEIC-specific tuning.
- Animated WebP.
- Caching (caller uploads immediately and discards).
