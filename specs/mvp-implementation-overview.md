# MVP Implementation Overview

This document is the shared brief for every agent working on the Photo Backup MVP. Read it once, then jump to the task spec you've been assigned in [`mvp-tasks/`](./mvp-tasks/).

Source-of-truth references: [`mvp-prd.md`](./mvp-prd.md), [`../architecture/mvp-architecture.md`](../architecture/mvp-architecture.md), and the [`../decisions/`](../decisions/) ADRs.

---

## What we are building

A minimal Android app that:

1. Reads photos from the device gallery (`MediaStore`).
2. Uploads originals + WebP thumbnails to a Backblaze B2 bucket on user tap.
3. Tracks upload state in a local SQLite DB so status persists across restarts.

This is an MVP to **prove the upload pipeline**. No background uploads, no video, no settings screen, no cloud browsing — see the PRD's Non-Goals section.

---

## What this agent needs to build

You have been assigned **one task** from [`mvp-tasks/`](./mvp-tasks/) (numbered 01–09). Each task spec is self-contained: it lists the files you create, the public contract you must expose, the dependencies you may rely on (by interface, not implementation), and the acceptance criteria.

The 9 tasks are:

| # | Task | Owns |
|---|------|------|
| 01 | [Project scaffolding](./mvp-tasks/01-project-scaffolding.md) | Gradle, Manifest, Application class, package skeleton |
| 02 | [Data layer (SQLite)](./mvp-tasks/02-data-layer.md) | `UploadRecord`, `UploadDatabase`, `UploadDao` |
| 03 | [Credential storage](./mvp-tasks/03-credential-storage.md) | `B2Credentials`, `PrefsStore` |
| 04 | [MediaStore query](./mvp-tasks/04-mediastore-query.md) | `MediaStorePhoto`, `MediaStoreQuery`, permission helper |
| 05 | [Thumbnail generator](./mvp-tasks/05-thumbnail-generator.md) | `ThumbnailGen` |
| 06 | [B2/S3 uploader](./mvp-tasks/06-b2-uploader.md) | `S3Config`, `S3ClientFactory`, `S3Uploader`, `S3KeyBuilder` |
| 07 | [Onboarding UI](./mvp-tasks/07-onboarding-ui.md) | `OnboardingActivity` + layout |
| 08 | [Gallery UI](./mvp-tasks/08-gallery-ui.md) | `GalleryActivity`, adapter, layouts, upload orchestration |
| 09 | [Integration & wire-up](./mvp-tasks/09-integration.md) | `MainActivity` routing, singleton wiring, smoke test |

### Parallelism rules

Task 01 (scaffolding) provides the shared `build.gradle.kts` + `AndroidManifest.xml` and should run first; **tasks 02–08 can run fully in parallel** since they touch disjoint files. Task 09 runs last and only does wiring — it does not modify the components produced by 02–08.

If you are an agent picking up a task, you do **not** need the other components implemented. Code against the contracts defined in this overview. The integration agent will verify everything fits.

---

## Architecture summary

```
┌────────────── UI ─────────────────┐
│  OnboardingActivity   (Task 07)   │
│  GalleryActivity      (Task 08)   │
│  MainActivity         (Task 09)   │
└───────────────────────────────────┘
                │
┌────────── Service ────────────────┐
│  S3Uploader           (Task 06)   │
│  ThumbnailGen         (Task 05)   │
└───────────────────────────────────┘
                │
┌──────────── Data ─────────────────┐
│  UploadDao            (Task 02)   │
│  PrefsStore           (Task 03)   │
│  MediaStoreQuery      (Task 04)   │
└───────────────────────────────────┘
```

Upload data flow: tap → `ThumbnailGen.createWebPThumbnail` → `S3Uploader` (photo + thumbnail) → `UploadDao.insert` → UI refresh.

---

## Project conventions

- **Language:** Kotlin only.
- **Package root:** `com.hriyaan.photostorage`.
- **Gradle DSL:** Kotlin DSL (`build.gradle.kts`).
- **Min SDK:** 33. **Target/compile SDK:** 34.
- **Coroutines:** All I/O (DB, network, bitmap) runs on `Dispatchers.IO`. UI uses `Dispatchers.Main` / `lifecycleScope`.
- **No DI framework.** Construct dependencies manually; share singletons through `PhotoBackupApp` (the `Application` subclass).
- **No Room, no Retrofit, no Hilt/Koin.** Use raw `SQLiteOpenHelper`. Use AWS SDK for Kotlin directly.
- **Comments:** Avoid them. Use clear names. Only comment when *why* is non-obvious.
- **Errors:** Service-layer methods return `kotlin.Result<T>`; UI translates failures to toasts.

---

## Contract surface

These are the **only** types and signatures other tasks may rely on. Each owning task implements them; consuming tasks may treat them as fixtures.

### `data` package

```kotlin
// Task 02
data class UploadRecord(
    val id: Long = 0L,
    val localUri: String,
    val filename: String,
    val size: Long,
    val dateTaken: Long,
    val photoB2Path: String?,
    val thumbnailB2Path: String?,
    val status: String,        // "pending" | "uploading" | "uploaded" | "failed"
    val uploadedAt: Long?
)

class UploadDatabase(context: Context) {
    val dao: UploadDao
}

class UploadDao(/* internal */) {
    fun insert(record: UploadRecord): Long
    fun updateStatus(id: Long, status: String, uploadedAt: Long? = null): Int
    fun setUploadedPaths(id: Long, photoPath: String, thumbnailPath: String, uploadedAt: Long): Int
    fun findByFilenameAndSize(filename: String, size: Long): UploadRecord?
    fun getAll(): List<UploadRecord>
}

// Task 03
data class B2Credentials(
    val keyId: String,
    val applicationKey: String,
    val bucketName: String
)

class PrefsStore(context: Context) {
    fun saveCredentials(creds: B2Credentials)
    fun getCredentials(): B2Credentials?
    fun clearCredentials()
    fun hasCredentials(): Boolean
}

// Task 04
data class MediaStorePhoto(
    val id: Long,
    val uri: Uri,
    val filename: String,
    val size: Long,
    val dateTakenMs: Long
)

class MediaStoreQuery(context: Context) {
    fun queryAllPhotos(): List<MediaStorePhoto>
}

object PhotoPermission {
    const val PERMISSION = Manifest.permission.READ_MEDIA_IMAGES
    fun isGranted(context: Context): Boolean
}
```

### `thumbnail` package

```kotlin
// Task 05
class ThumbnailGen(context: Context) {
    fun createWebPThumbnail(uri: Uri, maxDim: Int = 200): ByteArray
}
```

### `b2` package

```kotlin
// Task 06
data class S3Config(
    val region: String,                  // e.g. "us-west-004"
    val endpoint: String,                // e.g. "https://s3.us-west-004.backblazeb2.com"
    val bucketName: String
)

object S3ClientFactory {
    fun create(credentials: B2Credentials, config: S3Config): S3Client
}

class S3Uploader(private val client: S3Client, private val bucket: String) {
    suspend fun validateCredentials(): Result<Unit>
    suspend fun upload(
        key: String,
        contentType: String,
        contentLength: Long,
        body: ByteStream
    ): Result<Unit>
    fun close()
}

object S3KeyBuilder {
    fun photoKey(filename: String, dateTakenMs: Long): String          // "photos/YYYY/MM/DD/<filename>"
    fun thumbnailKey(filename: String, dateTakenMs: Long): String      // "thumbnails/YYYY/MM/DD/<basename>.webp"
}
```

### `ui` package

```kotlin
// Task 07
class OnboardingActivity : AppCompatActivity()

// Task 08
class GalleryActivity : AppCompatActivity()

// Task 09
class MainActivity : AppCompatActivity()
```

### Application root

```kotlin
// Task 01 declares the class. Task 09 fills in the singletons.
class PhotoBackupApp : Application() {
    lateinit var prefsStore: PrefsStore
    lateinit var uploadDatabase: UploadDatabase
}
```

---

## File ownership map

So that two agents never write to the same file:

| Path | Owner |
|------|-------|
| `app/build.gradle.kts`, `build.gradle.kts`, `settings.gradle.kts`, `gradle.properties` | Task 01 |
| `app/src/main/AndroidManifest.xml` | Task 01 (creates), Task 09 (may update entry activity) |
| `app/src/main/java/com/hriyaan/photostorage/PhotoBackupApp.kt` | Task 01 (skeleton), Task 09 (singleton wiring) |
| `app/src/main/java/com/hriyaan/photostorage/MainActivity.kt` | Task 09 |
| `app/src/main/java/com/hriyaan/photostorage/data/UploadRecord.kt`, `UploadDao.kt`, `UploadDatabase.kt` | Task 02 |
| `app/src/main/java/com/hriyaan/photostorage/data/PrefsStore.kt` | Task 03 |
| `app/src/main/java/com/hriyaan/photostorage/data/MediaStoreQuery.kt`, `PhotoPermission.kt` | Task 04 |
| `app/src/main/java/com/hriyaan/photostorage/thumbnail/ThumbnailGen.kt` | Task 05 |
| `app/src/main/java/com/hriyaan/photostorage/b2/S3Config.kt`, `S3ClientFactory.kt`, `S3Uploader.kt`, `S3KeyBuilder.kt` | Task 06 |
| `app/src/main/java/com/hriyaan/photostorage/ui/OnboardingActivity.kt` + `res/layout/activity_onboarding.xml` | Task 07 |
| `app/src/main/java/com/hriyaan/photostorage/ui/GalleryActivity.kt`, `GalleryAdapter.kt` + `res/layout/activity_gallery.xml`, `item_gallery.xml` | Task 08 |

Each task spec restates the files it owns.

---

## Key constraints (do not violate)

1. **B2 checksum quirk.** Every `PutObjectRequest` must set `checksumAlgorithm = null` — see [the ADR](../decisions/2026-05-09-use-s3-compatible-api-over-native-b2-api.md). If checksums are sent, B2 returns `400 Bad Request`. This is verified during Task 09's smoke test.
2. **Permission target.** Use `READ_MEDIA_IMAGES` (API 33+ only). Do not request `READ_EXTERNAL_STORAGE` — it is the wrong permission on our min SDK.
3. **Credentials never logged.** No `Log.d(..., creds.applicationKey)`. Ever.
4. **No master account keys.** Onboarding accepts B2 *Application Keys* only; the UI does not need to enforce this but documentation must say so.
5. **`uploads.db` schema.** Single `uploads` table, schema defined in Task 02. Other tasks do not add columns.
