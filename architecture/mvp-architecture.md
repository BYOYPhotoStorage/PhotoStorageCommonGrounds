# MVP Architecture

This document describes the architecture for the minimal viable product (MVP) of the photo backup app.

---

## Principles

1. **Minimal abstractions.** No dependency injection frameworks, no complex repositories, no over-engineering. Three similar lines is better than a premature abstraction.
2. **Prove the pipeline.** The goal is to validate B2 upload end-to-end. Everything else is secondary.
3. **Room to grow.** The schema and file layout should not block post-MVP features.

---

## High-Level Components

```
┌─────────────────────────────────────────┐
│               UI Layer                  │
│  ┌──────────────┐  ┌────────────────┐  │
│  │  Onboarding  │  │ GalleryScreen  │  │
│  │   Activity   │  │   Activity     │  │
│  └──────────────┘  └────────────────┘  │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│            Service Layer                │
│  ┌──────────────┐  ┌────────────────┐  │
│  │  S3Uploader  │  │ ThumbnailGen   │  │
│  │(AWS SDK for  │  │  (Bitmap ops)  │  │
│  │   Kotlin)     │  │                │  │
│  └──────────────┘  └────────────────┘  │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│            Data Layer                   │
│  ┌──────────────┐  ┌────────────────┐  │
│  │   UploadDao  │  │  MediaStore    │  │
│  │   (SQLite)   │  │   (system API) │  │
│  └──────────────┘  └────────────────┘  │
│  ┌──────────────┐                      │
│  │   PrefsStore │                      │
│  │(Encrypted    │                      │
│  │SharedPrefs)  │                      │
│  └──────────────┘                      │
└─────────────────────────────────────────┘
```

---

## Data Flow: Upload

```
User taps photo in GalleryScreen
         │
         ▼
GalleryScreen reads local URI from MediaStore
         │
         ▼
ThumbnailGen.createWebPThumbnail(uri) ──► WebP bytes
         │
         ▼
S3Uploader.upload(photoFile, "photos/YYYY/MM/DD/filename.jpg")
         │
         ▼
S3Uploader.upload(thumbnailBytes, "thumbnails/YYYY/MM/DD/filename.webp")
         │
         ▼
UploadDao.insert(record with status=uploaded, paths, timestamp)
         │
         ▼
GalleryScreen refreshes grid, shows "uploaded" badge
```

---

## Package Structure

```
com.photobackup.app
├── data
│   ├── UploadDao.kt              # SQLite CRUD
│   ├── UploadDatabase.kt         # Room or raw SQLite helper
│   ├── MediaStoreQuery.kt        # Query local gallery
│   └── PrefsStore.kt             # Encrypted credential storage
├── b2
│   ├── S3ClientFactory.kt        # Builds AWS S3Client with B2 endpoint
│   ├── S3Uploader.kt             # Upload orchestration (PutObject, etc.)
│   └── S3Config.kt               # Region, endpoint, bucket name
├── thumbnail
│   └── ThumbnailGen.kt           # Bitmap → WebP
├── ui
│   ├── OnboardingActivity.kt
│   └── GalleryActivity.kt
└── model
    └── UploadRecord.kt           # Data class for DB row
```

---

## SQLite Schema

Single table, single database file: `uploads.db`

```sql
CREATE TABLE uploads (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    local_uri TEXT NOT NULL,
    filename TEXT NOT NULL,
    size INTEGER NOT NULL,
    date_taken INTEGER NOT NULL,
    photo_b2_path TEXT,
    thumbnail_b2_path TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    uploaded_at INTEGER
);

CREATE INDEX idx_status ON uploads(status);
CREATE INDEX idx_filename_size ON uploads(filename, size);
```

**Why this schema:**
- `filename + size` is the MVP duplicate check. Fast, no hash computation needed.
- `status` is indexed because the gallery screen queries by status to show badges.
- Nullable `photo_b2_path` allows records to exist before upload completes.

---

## B2 Integration (S3-Compatible API)

The app uses Backblaze B2's **S3-compatible API** via the **AWS SDK for Kotlin**. This provides standard S3 semantics, built-in retry logic, and future portability to other S3-compatible providers (Wasabi, Storj, MinIO, etc.).

### Authentication

B2 Application Key ID → S3 `Access Key ID`
B2 Application Key → S3 `Secret Access Key`

These are passed to the SDK via `StaticCredentialsProvider`.

### S3Client Configuration

```kotlin
S3Client {
    region = "us-west-004"                          // bucket region
    endpointUrl = Url.parse("https://s3.us-west-004.backblazeb2.com")
    credentialsProvider = StaticCredentialsProvider {
        accessKeyId = keyId
        secretAccessKey = appKey
    }
    // Use OkHttp engine for Android compatibility
    httpClient(OkHttpEngine)
}
```

### Upload Flow

```kotlin
s3.putObject(PutObjectRequest {
    bucket = bucketName
    key = "photos/2025/01/01/IMG_1234.jpg"
    body = file.asByteStream()
    contentType = "image/jpeg"
    // Disable checksum to avoid B2 incompatibility
    checksumAlgorithm = null
})
```

### Credential Validation (Onboarding)

Call `s3.listBuckets()` with the configured credentials. If it succeeds, credentials are valid.

### ⚠️ Known Compatibility Issue

AWS SDK for Kotlin v1.3+ automatically computes `x-amz-checksum-crc32` headers. **B2 rejects these.** Always set `checksumAlgorithm = null` on `PutObject` and `UploadPart` requests.

If this becomes problematic, the fallback is the **MinIO Java client** which has no checksum issues and is lighter weight.

---

## MediaStore Query

```kotlin
val projection = arrayOf(
    MediaStore.Images.Media._ID,
    MediaStore.Images.Media.DISPLAY_NAME,
    MediaStore.Images.Media.SIZE,
    MediaStore.Images.Media.DATE_TAKEN
)

val cursor = contentResolver.query(
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
    projection,
    null,
    null,
    "${MediaStore.Images.Media.DATE_TAKEN} DESC"
)
```

- Query on every gallery screen launch. The dataset is small enough (local photos only) that caching is unnecessary for MVP.
- Convert `_ID` to content URI: `content://media/external/images/media/$id`

---

## Thumbnail Generation

```kotlin
fun createWebPThumbnail(contentUri: Uri, maxDim: Int = 200): ByteArray {
    val bitmap = contentResolver.loadThumbnail(contentUri, Size(maxDim, maxDim), null)
    val output = ByteArrayOutputStream()
    bitmap.compress(Bitmap.CompressFormat.WEBP_LOSSY, 80, output)
    return output.toByteArray()
}
```

- Target 200x200 px, ~20–40 KB per thumbnail.
- Use Android's `ContentResolver.loadThumbnail()` (API 29+) for efficiency.

---

## UI Structure

### OnboardingActivity

```
LinearLayout (vertical)
├── TextInputEditText: B2 Key ID
├── TextInputEditText: B2 Application Key (passwordToggleEnabled)
├── TextInputEditText: B2 Bucket Name
├── MaterialButton: "Connect"
└── TextView: Error message (hidden by default)
```

- On "Connect": disable button, show indeterminate progress, call `S3ClientFactory.createClient().listBuckets()`.
- On success: save credentials, navigate to GalleryActivity.
- On failure: show error, re-enable button.

### GalleryActivity

```
CoordinatorLayout
├── AppBarLayout
│   └── Toolbar: "Photo Backup"
└── RecyclerView (GridLayoutManager, 3 columns)
    └── Item: Square ImageView + small status icon overlay
```

- RecyclerView adapter takes a list of `GalleryItem`.
- `GalleryItem` is built by joining MediaStore results with `UploadDao` records (filename + size match).
- Tap triggers upload. Show a circular progress overlay during upload.
- Status overlay:
  - No icon: not uploaded
  - Cloud icon: uploaded
  - Spinner: uploading

---

## Credential Storage

Use AndroidX Security library (`EncryptedSharedPreferences`):

```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val prefs = EncryptedSharedPreferences.create(
    context,
    "b2_credentials",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

prefs.edit()
    .putString("key_id", keyId)
    .putString("application_key", appKey)
    .putString("bucket_name", bucketName)
    .apply()
```

---

## Error Handling Strategy

| Error | Behavior |
|-------|----------|
| No network | Toast: "No connection. Tap to retry." Status stays `pending`. |
| S3 403 Forbidden | Invalid credentials. Toast: "Check your B2 keys." |
| S3 404 NoSuchBucket | Wrong bucket name. Toast: "Bucket not found." |
| Disk full during thumbnail gen | Toast: "Storage full." |
| Upload fails mid-stream | Status → `failed`. User can tap to retry. |

No automatic retry in MVP. Manual retry only.

---

## Dependencies

```groovy
dependencies {
    // UI
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.11.0'
    implementation 'androidx.recyclerview:recyclerview:1.3.2'

    // S3 / B2 S3-Compatible API
    implementation 'aws.sdk.kotlin:s3:1.6.46'
    implementation 'aws.smithy.kotlin:http-client-engine-okhttp:1.0.0'

    // Image loading
    implementation 'com.github.bumptech.glide:glide:4.16.0'

    // Security (encrypted prefs)
    implementation 'androidx.security:security-crypto:1.1.0-alpha06'
}
```

No Room, no Koin/Hilt, no Retrofit. Keep the dependency surface tiny.

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| AWS SDK checksum headers rejected by B2 | Always set `checksumAlgorithm = null` on PutObject; test early |
| AWS SDK dependency size (~3-5 MB) | Acceptable tradeoff for coroutine-native API; fallback to MinIO if needed |
| Large photos cause OOM during thumbnail gen | Decode with `BitmapFactory.Options.inSampleSize` before scaling |
| Slow gallery load with many photos | Add pagination (limit 100, offset) in post-MVP if needed |
| S3 credentials invalid mid-session | SDK handles retry; on persistent 403, toast and redirect to onboarding |

---

## Post-MVP Extension Points

The architecture is intentionally flat so these additions don't require rewrites:

- **Background upload:** Add a `WorkManager` worker that calls the same `S3Uploader.upload()` method.
- **ContentObserver:** Register in `GalleryActivity.onStart()`, trigger the same worker.
- **SHA-256 duplicates:** Add `sha256` column to `uploads` table, compute in `S3Uploader` before upload.
- **SQLite backup to B2:** Upload `uploads.db` file via existing `S3Uploader`.
- **Settings screen:** Read/write from `PrefsStore`, add keys for upload mode, deletion policy, etc.
