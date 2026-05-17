# Task 29 — Thumbnail Cache

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Wire up a Coil `ImageLoader` that:
1. Decodes local thumbnails directly from MediaStore URIs (no caching needed beyond Coil's defaults — these are already on disk).
2. Fetches B2-stored thumbnails through a custom `Fetcher` that uses `S3Uploader.downloadObject`, then caches them on a 200 MB disk LRU.
3. Exposes a `prefetch` helper for the gallery to warm the cache as the user scrolls into a cloud-only window.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/thumbnail/ThumbnailCacheFactory.kt`
- `app/src/main/java/com/hriyaan/photostorage/thumbnail/B2ThumbnailFetcher.kt`

## Dependencies to add (in Task 30, not here)

```kotlin
implementation("io.coil-kt:coil:2.7.0")
```

You may reference Coil types in this task; the dependency will be present in the final build via Task 30's gradle change.

## `B2ThumbnailFetcher`

```kotlin
class B2ThumbnailFetcher(
    private val path: String,
    private val context: Context,
    private val s3Uploader: S3Uploader
) : Fetcher {

    override suspend fun fetch(): FetchResult {
        val cacheDir = File(context.cacheDir, "thumbnails").also { it.mkdirs() }
        val cacheFile = File(cacheDir, cacheKey(path))

        if (!cacheFile.exists()) {
            val tmp = File(cacheDir, "${cacheKey(path)}.tmp")
            s3Uploader.downloadObject(path, tmp).getOrThrow()
            tmp.renameTo(cacheFile)
        }

        // Touch mtime for LRU
        cacheFile.setLastModified(System.currentTimeMillis())

        return SourceResult(
            source = ImageSource(cacheFile.source().buffer(), context),
            mimeType = "image/webp",
            dataSource = DataSource.DISK
        )
    }

    private fun cacheKey(p: String): String =
        MessageDigest.getInstance("SHA-256").digest(p.toByteArray()).joinToString("") { "%02x".format(it) }

    class Factory(
        private val context: Context,
        private val s3Uploader: S3Uploader
    ) : Fetcher.Factory<ThumbnailSource.B2Path> {
        override fun create(data: ThumbnailSource.B2Path, options: Options, imageLoader: ImageLoader): Fetcher =
            B2ThumbnailFetcher(data.path, context, s3Uploader)
    }
}
```

> Coil 2.x `Fetcher` API: implement `fetch(): FetchResult`. Register via `Fetcher.Factory<DataType>`. Treat `ThumbnailSource.B2Path` (from Task 23) as the model type so callers pass it directly into `ImageRequest`.

## `ThumbnailCacheFactory`

```kotlin
class ThumbnailCacheFactory(
    private val context: Context,
    private val s3Uploader: S3Uploader,
    private val prefsStore: PrefsStore
) {
    private val imageLoader: ImageLoader by lazy { build() }

    fun get(): ImageLoader = imageLoader

    fun prefetch(paths: List<ThumbnailSource.B2Path>) {
        if (!shouldPrefetch()) return
        for (p in paths) {
            val request = ImageRequest.Builder(context)
                .data(p)
                .memoryCachePolicy(CachePolicy.ENABLED)
                .diskCachePolicy(CachePolicy.ENABLED)
                .build()
            imageLoader.enqueue(request)
        }
    }

    private fun shouldPrefetch(): Boolean {
        if (!prefsStore.isWifiOnly()) return true
        val cm = context.getSystemService(ConnectivityManager::class.java)
        val active = cm.activeNetwork ?: return false
        val caps = cm.getNetworkCapabilities(active) ?: return false
        return caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_NOT_METERED)
    }

    private fun build(): ImageLoader {
        val thumbnailCacheDir = File(context.cacheDir, "thumbnails").also { it.mkdirs() }
        return ImageLoader.Builder(context)
            .diskCache {
                DiskCache.Builder()
                    .directory(thumbnailCacheDir)
                    .maxSizeBytes(200L * 1024 * 1024) // 200 MB
                    .build()
            }
            .memoryCache { MemoryCache.Builder(context).maxSizePercent(0.20).build() }
            .components {
                add(B2ThumbnailFetcher.Factory(context, s3Uploader))
            }
            .build()
    }
}
```

## Usage from the adapter (Task 24 wires this)

```kotlin
val source = item.thumbnailSource
val request = ImageRequest.Builder(context)
    .data(when (source) {
        is ThumbnailSource.LocalUri -> source.uri
        is ThumbnailSource.B2Path -> source
    })
    .placeholder(R.drawable.thumbnail_placeholder)
    .error(R.drawable.thumbnail_error)
    .target(holder.imageView)
    .build()
imageLoader.enqueue(request)
```

Coil's default handlers cover `Uri`. The custom `Fetcher.Factory<ThumbnailSource.B2Path>` covers our B2 path type.

## Implementation notes

- `200 MB` is hardcoded per PRD constraint #5. If we later make it configurable, this is the single place to change.
- Use Coil's built-in disk cache (`DiskCache`) for LRU and size limits — don't reimplement. Coil keys cache entries by the request's data hash; for B2 paths we use the `ThumbnailSource.B2Path` instance, so Coil hashes its `path` field and the `B2ThumbnailFetcher` only runs on cache miss.
- The `Fetcher` writes to a `.tmp` file first then renames — this prevents Coil from observing a partially-written file if the process is killed mid-download.
- Prefetch only runs when the active network is appropriate per `shouldPrefetch()`.
- `mimeType = "image/webp"` matches the upload format from Q3.

## Constraints

- 200 MB disk cap (PRD constraint #5).
- Do not pre-decode bitmaps — let Coil handle decode + memory cache.
- Do not prefetch on metered networks when `wifi_only_uploads = true`.
- The cache directory is `context.cacheDir/thumbnails/` — Android may clear it under storage pressure (acceptable: cache misses re-fetch).

## Dependencies (by interface)

- `S3Uploader.downloadObject` (Task 22)
- `PrefsStore.isWifiOnly` (Phase 2 Task 11)
- Coil 2.x (added in Task 30 via `app/build.gradle.kts`)
- `ThumbnailSource` and related types (Task 23)

## Acceptance criteria

- [ ] First Cloud-view render of a tile downloads the thumbnail from B2 and caches it on disk under `context.cacheDir/thumbnails/`.
- [ ] Second render of the same tile is served from disk cache — no B2 request fires (verify by checking S3 client logs or instrumented mock).
- [ ] Coil's disk cache does not exceed 200 MB; oldest entries are evicted.
- [ ] `prefetch(paths)` warms the cache when network is unmetered and Wi-Fi-only is on; it is a no-op on metered networks under that setting.
- [ ] Switching between Local and Cloud views does not re-download thumbnails already cached.
- [ ] A failed download cleans up the `.tmp` file (no half-written entries remain).

## Out of scope

- Memory-only LRU sizing (Coil defaults to ~20% of heap; that's fine).
- Pre-warming the cache for the entire gallery on app launch.
- HEIC/AVIF support beyond what Coil + Android decoders already provide.
- A "clear thumbnail cache" UI action — out of scope for Phase 3.
