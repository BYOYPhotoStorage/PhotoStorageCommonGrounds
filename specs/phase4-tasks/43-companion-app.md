# Task 43 — Companion App (`photo-storage-tester`) for E2E Testing

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) and [`../../decisions/2026-07-03-mobile-testing-strategy-maestro-companion-app.md`](../../decisions/2026-07-03-mobile-testing-strategy-maestro-companion-app.md) first.

## Goal

Create a minimal, standalone Android companion app whose only job is to put the device into a known MediaStore state for end-to-end tests. It must:

1. Pull reference photos and videos from a dedicated B2 bucket (`refImages0307`).
2. Randomly select a requested number of items, download them, and inject them into the camera roll.
3. Tag injected media so tests can identify and clean it up later.
4. Salt each injected copy so the main app treats it as a new upload (unless explicitly testing deduplication).
5. Report exactly what was injected so a host-side cloud harness can verify uploads independently.
6. Delete injected media by tag or wipe all test media.

This app lives outside the main `com.hriyaan.photostorage` APK so test code and broad media permissions never ship to users.

## Scope

- One new Gradle module: `companion/`.
- One Activity + one RecyclerView UI.
- One headless intent API for the orchestration runner.
- Read-only access to the reference B2 bucket (`refImages0307`).
- No write access to the upload bucket (`test0307`) and no cloud verification logic inside the companion app.

## Files you own

New files under `photoStorageApp/companion/`:

- `build.gradle.kts`
- `src/main/AndroidManifest.xml`
- `src/main/java/com/photostorage/tester/MainActivity.kt`
- `src/main/java/com/photostorage/tester/InjectionService.kt`
- `src/main/java/com/photostorage/tester/InjectionViewModel.kt`
- `src/main/java/com/photostorage/tester/MediaInjector.kt`
- `src/main/java/com/photostorage/tester/RefAsset.kt`
- `src/main/java/com/photostorage/tester/RefBucketClient.kt`
- `src/main/java/com/photostorage/tester/InjectionReport.kt`
- `src/main/java/com/photostorage/tester/PoolAdapter.kt`
- `src/main/res/layout/activity_main.xml`
- `src/main/res/layout/item_pool_asset.xml`

Modified files:

- `photoStorageApp/settings.gradle.kts` — add `include(":companion")`.
- `photoStorageApp/gradle/libs.versions.toml` — add `exifinterface` if you prefer the catalog over a hard-coded version.

## Module setup

`photoStorageApp/companion/build.gradle.kts`:

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.photostorage.tester"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.photostorage.tester"
        minSdk = 33
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }

    buildFeatures {
        viewBinding = true
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    implementation(libs.androidx.appcompat)
    implementation(libs.material)
    implementation(libs.androidx.recyclerview)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    implementation(libs.kotlinx.coroutines.android)

    // Same S3/B2 client the main app uses
    implementation(libs.aws.s3)
    implementation(libs.smithy.okhttp)

    // For salting photos without re-encoding
    implementation("androidx.exifinterface:exifinterface:1.3.7")
}
```

## Permissions

`photoStorageApp/companion/src/main/AndroidManifest.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <!-- Needed to query and delete media we injected -->
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
    <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />

    <application
        android:allowBackup="false"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/Theme.Material3.DayNight.NoActionBar"
        android:usesCleartextTraffic="false">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTask">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service
            android:name=".InjectionService"
            android:exported="false"
            android:foregroundServiceType="dataSync" />
    </application>
</manifest>
```

> No `WRITE_EXTERNAL_STORAGE` or `MANAGE_EXTERNAL_STORAGE` is required. On API 33+ an app can insert media into its own MediaStore entries without broad write permission, and it can delete entries it owns.

## Reference media pool on B2

The companion app treats the `refImages0307` bucket as its media pool.

### How testers add media

Users upload photos and videos to `refImages0307` through any B2 client (web UI, `b2` CLI, `aws s3 cp`, the orchestration runner, etc.). Example:

```bash
aws s3 cp ./my-smoke-assets/ s3://refImages0307/smoke/ \
  --endpoint-url=https://s3.us-west-004.backblazeb2.com --recursive
```

The companion app lists the bucket at injection time, filters by content type (`image/*` for photos, `video/*` for videos), and picks the requested number of objects.

### Object selection

- By default the app randomly selects `count` objects that match the requested `type`.
- Pass `seed` (int) to make the random selection reproducible across runs.
- Pass `prefix` (string) to restrict listing to a bucket prefix (e.g. `smoke/`, `photos/`).
- The UI can show the current pool listing with checkboxes so a user can manually exclude items.

Downloaded reference objects are cached in the app-private cache directory so repeated injections from the same pool are fast. The cache can be cleared from the UI or with `WIPE_ALL_TEST_MEDIA`.

### B2 credentials

The companion app embeds the same read-only test credentials used by the main app's `OnboardingActivity` test0307 shortcut:

```kotlin
const val DEFAULT_KEY_ID = "0046d07a497f39b0000000001"
const val DEFAULT_APP_KEY = "K004tM1PrvOAYZPiMWLFZsd51gSxZVo"
const val DEFAULT_REGION = "us-west-004"
const val DEFAULT_REF_BUCKET = "refImages0307"
```

The intent API also accepts overrides (`b2KeyId`, `b2AppKey`, `refBucket`) so the runner or a tester can point at a different reference bucket without rebuilding.

## Injection behavior

### Unique content per injection

The main app deduplicates uploads by SHA256. If we inject the same reference bytes twice, the second injection may be skipped. To avoid this, the companion app **salts** every injected copy so each run (and each sequence number within a run) has a unique SHA256.

- **Photos:** open the downloaded copy with `ExifInterface` and set:
  - `TAG_DATETIME_ORIGINAL`
  - `TAG_DATETIME_DIGITIZED`
  - `TAG_USER_COMMENT` to a JSON blob containing the run tag and sequence number.
  Saving the Exif header changes the file bytes and therefore the hash.
- **Videos:** append a valid MP4 `free` atom containing the run tag and sequence number. Trailing `free` atoms are ignored by players and by Media3 Transformer, but they change the file hash.

The salting step is controlled by the intent extra `unique` (boolean, default `true`). Set `unique=false` only when intentionally testing deduplication behavior.

### File naming

Each injected item is written to MediaStore with display name:

```
ps_test_<tag>_<seq>_<originalFilename>
```

Example: `ps_test_a1b2c3_001_DSC_1234.jpg`

The `ps_test_` prefix is reserved for test media and is used by the cleanup commands.

### MediaStore insertion

For each selected reference object:

1. Download the object to a cache file if not already cached.
2. Compute SHA256 of the original downloaded bytes and record it as `sourceSha256`.
3. Copy the asset to a working file.
4. Salt the working file if `unique=true`.
5. Compute SHA256 of the salted file.
6. Insert a new MediaStore entry with:
   - `DISPLAY_NAME` = `ps_test_<tag>_<seq>_<originalFilename>`
   - `MIME_TYPE` = source object MIME type
   - `IS_PENDING` = 0
   - `DATE_ADDED` / `DATE_TAKEN` = current time
7. Copy the salted bytes into `ContentResolver.openOutputStream(uri)`.
8. Record the MediaStore URI, source bucket/key, and both SHA256 values in the report.

Because the file is new in MediaStore, the main app's `UploadForegroundService` content observer fires and the upload worker enqueues it.

## Intent-based API

The companion app exposes actions via explicit intents so the orchestration runner can drive it without UI.

### `com.photostorage.tester.INJECT_PHOTOS`

Extras:

| Extra | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `tag` | string | yes | — | Run identifier, used in filenames and report |
| `count` | int | no | 1 | Number of items to inject |
| `type` | string | no | `"both"` | `"photo"`, `"video"`, or `"both"` |
| `unique` | boolean | no | true | Whether to salt each copy |
| `refBucket` | string | no | `"refImages0307"` | Reference B2 bucket to list |
| `prefix` | string | no | — | Restrict listing to this bucket prefix |
| `seed` | int | no | random | Seed for reproducible random selection |
| `b2KeyId` | string | no | embedded test key | Override B2 key ID |
| `b2AppKey` | string | no | embedded test key | Override B2 application key |
| `b2Region` | string | no | `"us-west-004"` | Override B2 region |

Example:

```bash
adb shell am start-foreground-service \
  -a com.photostorage.tester.INJECT_PHOTOS \
  --es tag "a1b2c3" \
  --ei count 5 \
  --es type "photo" \
  --es prefix "smoke/" \
  --ei seed 42 \
  -n com.photostorage.tester/.InjectionService
```

When injection finishes, the app:

1. Writes `files/reports/<tag>.json`.
2. Logs `I/PhotoStorageTester: INJECTION_COMPLETE tag=<tag> report=<path> success=true`.
3. Broadcasts:

```
action: com.photostorage.tester.INJECTION_COMPLETE
extras: tag (string), reportPath (string), success (boolean), error (string?)
```

### `com.photostorage.tester.DELETE_BY_TAG`

Deletes every MediaStore entry whose `DISPLAY_NAME` starts with `ps_test_<tag>_`.

Extras:

| Extra | Type | Required |
|-------|------|----------|
| `tag` | string | yes |

### `com.photostorage.tester.WIPE_ALL_TEST_MEDIA`

Deletes every MediaStore entry whose `DISPLAY_NAME` starts with `ps_test_`, regardless of tag.

### `com.photostorage.tester.REPORT_INJECTED`

Forces the app to rewrite the report for a given tag by re-querying MediaStore. Useful if the orchestration runner missed the broadcast.

| Extra | Type | Required |
|-------|------|----------|
| `tag` | string | yes |

## Report format

Written to `/sdcard/Android/data/com.photostorage.tester/files/reports/<tag>.json` (also mirrored to the app-private `files/reports/` directory).

```json
{
  "tag": "a1b2c3",
  "injectedAt": "2026-07-04T12:34:56Z",
  "appVersion": "1.0",
  "unique": true,
  "refBucket": "refImages0307",
  "prefix": "smoke/",
  "items": [
    {
      "sequence": 1,
      "sourceBucket": "refImages0307",
      "sourceKey": "smoke/DSC_1234.jpg",
      "sourceSha256": " original-downloaded-sha256...",
      "displayName": "ps_test_a1b2c3_001_DSC_1234.jpg",
      "mimeType": "image/jpeg",
      "sizeBytes": 205100,
      "sha256": " salted-sha256...",
      "mediaStoreUri": "content://media/external/images/media/12345",
      "kind": "photo"
    }
  ]
}
```

The cloud harness uses `displayName` and `sha256` to locate and verify objects in the upload bucket (`test0307`). Because the main app may compress images or transcode videos, the harness falls back to a "presence" match by `displayName` if an exact SHA256 match is not found.

## UI

`MainActivity` is a single-screen debug UI, not a polished product:

- Editable `tag` field (auto-fill with a random 6-character ID if blank).
- `count` number picker.
- `type` dropdown: photo / video / both.
- Optional `prefix` and `seed` fields.
- RecyclerView showing the current `refImages0307` listing with checkboxes.
- **Inject** button — runs injection and shows a progress spinner.
- **Delete by tag** button.
- **Wipe all test media** button.
- **Clear reference cache** button.
- Status log TextView showing the last report path and the last broadcast result.

The UI is optional for CI scenarios; all functionality is reachable via intents.

## Cleanup behavior

Cleanup deletes MediaStore entries and the local reference cache, not upload-bucket objects. Upload-bucket cleanup is the responsibility of the cloud harness so that test failures leave evidence in `test0307` until the harness explicitly cleans up.

## Testing the companion app in isolation

1. Upload a few small images to `refImages0307` (e.g. under `smoke/`).
2. Install the companion APK.
3. Open it, set prefix to `smoke/`, pick a tag, and tap **Inject**.
4. Open the system Photos app and confirm new items appear with the `ps_test_` prefix.
5. Pull the report:
   ```bash
   adb pull /sdcard/Android/data/com.photostorage.tester/files/reports/<tag>.json .
   ```
6. Tap **Delete by tag** and confirm the items disappear from Photos.

## Open decisions handled here

From the mobile-testing ADR:

- **Companion app location:** Same repo as a separate Gradle module (`:companion`). This keeps it version-locked with the main app and lets it reuse the project's version catalog.
- **Media pool location:** A dedicated B2 bucket (`refImages0307`). Testers upload media there directly; the companion app downloads and injects it. No assets are bundled in the APK.
- **Media uniqueness:** Each injected copy is salted by default so the main app sees a new file and uploads it.
- **Cloud verification location:** The companion app reports what was injected, but the actual bucket assertion runs on the host via the cloud harness (Task 44). No upload-bucket logic lives in the companion app.
- **B2 credentials:** Reuses the same read-only test credentials embedded in the main app's `OnboardingActivity` test0307 shortcut.

## Notes

- Keep the companion app intentionally small. Resist adding gallery browsing, upload-bucket browsing, or production features.
- The salting strategy must produce files that still pass the main app's thumbnail generator and video transcoder. If a video format breaks Media3 Transformer, switch to a simpler salt (e.g., overwrite a few bytes in a `udta` box instead of appending).
- The report path uses the external app-specific directory so `adb pull` works without `run-as` on debug builds.
- If `refImages0307` grows large, add server-side prefix filtering and pagination in `RefBucketClient`. For the first version, listing all objects and filtering locally is acceptable.
