# Task 01 — Project Scaffolding

> Read [`../mvp-implementation-overview.md`](../mvp-implementation-overview.md) first.

## Goal

Create the Android project skeleton so that tasks 02–08 can compile their code without further setup. After this task, `./gradlew assembleDebug` builds an empty (but launchable) APK.

## Files you own

- `settings.gradle.kts`
- `build.gradle.kts` (project root)
- `gradle.properties`
- `gradle/libs.versions.toml` (optional version catalog)
- `app/build.gradle.kts`
- `app/src/main/AndroidManifest.xml`
- `app/src/main/java/com/photobackup/app/PhotoBackupApp.kt` (skeleton — Task 09 will populate singletons)
- `app/src/main/java/com/photobackup/app/MainActivity.kt` (placeholder — Task 09 replaces logic)
- `app/src/main/res/values/strings.xml` (with `app_name`)
- `app/src/main/res/values/themes.xml` (Material3 default)
- `app/src/main/res/mipmap-*` launcher icon (default `ic_launcher`)
- `.gitignore` if missing

## Requirements

### Gradle

- Kotlin DSL throughout.
- AGP 8.5+, Kotlin 1.9.x or 2.0.x, JDK 17.
- `compileSdk = 34`, `minSdk = 33`, `targetSdk = 34`.
- `applicationId = "com.photobackup.app"`.
- Enable `viewBinding = true`.
- Use the dependency block from [`../../architecture/mvp-architecture.md`](../../architecture/mvp-architecture.md):

  ```kotlin
  dependencies {
      // UI
      implementation("androidx.appcompat:appcompat:1.6.1")
      implementation("com.google.android.material:material:1.11.0")
      implementation("androidx.recyclerview:recyclerview:1.3.2")
      implementation("androidx.activity:activity-ktx:1.9.0")
      implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.4")

      // S3 / B2 S3-compatible API
      implementation("aws.sdk.kotlin:s3:1.6.46")
      implementation("aws.smithy.kotlin:http-client-engine-okhttp:1.0.0")

      // Image loading
      implementation("com.github.bumptech.glide:glide:4.16.0")

      // Encrypted credential storage
      implementation("androidx.security:security-crypto:1.1.0-alpha06")

      // Coroutines (already pulled by AWS SDK, but pin it)
      implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")

      testImplementation("junit:junit:4.13.2")
      androidTestImplementation("androidx.test.ext:junit:1.2.1")
  }
  ```

  No Room. No Retrofit. No Hilt/Koin. No Compose.

### `AndroidManifest.xml`

- Permission declarations:
  - `android.permission.READ_MEDIA_IMAGES`
  - `android.permission.INTERNET`
  - `android.permission.ACCESS_NETWORK_STATE`
- `<application android:name=".PhotoBackupApp">` — references the skeleton Application class.
- Declare three activities (even if their classes are empty stubs that throw `NotImplementedError`): `MainActivity` (launcher, exported), `OnboardingActivity`, `GalleryActivity`. Tasks 07/08/09 fill in the bodies.
- `usesCleartextTraffic="false"` — B2 S3 endpoints are HTTPS-only.

### `PhotoBackupApp.kt`

```kotlin
package com.photobackup.app

import android.app.Application

class PhotoBackupApp : Application() {
    // Task 09 will add lateinit singletons here:
    //   lateinit var prefsStore: PrefsStore
    //   lateinit var uploadDatabase: UploadDatabase
    override fun onCreate() {
        super.onCreate()
    }
}
```

### `MainActivity.kt` (placeholder only)

```kotlin
package com.photobackup.app

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Task 09 replaces this with credential-routing logic.
        finish()
    }
}
```

### Stub activity classes for the manifest to compile

Create empty classes so the `AndroidManifest` references resolve. Tasks 07 and 08 will replace them.

```kotlin
// app/src/main/java/com/photobackup/app/ui/OnboardingActivity.kt
package com.photobackup.app.ui
import androidx.appcompat.app.AppCompatActivity
class OnboardingActivity : AppCompatActivity()
```

```kotlin
// app/src/main/java/com/photobackup/app/ui/GalleryActivity.kt
package com.photobackup.app.ui
import androidx.appcompat.app.AppCompatActivity
class GalleryActivity : AppCompatActivity()
```

> Tasks 07/08 **overwrite** these stubs. Coordinate via the file ownership map in the overview — your stubs are short-lived placeholders.

### Package directory skeleton

Create empty `.gitkeep` files (or just an empty Kotlin file) inside each package directory so Tasks 02–06 can drop their files in:

```
app/src/main/java/com/photobackup/app/
├── b2/
├── data/
├── thumbnail/
└── ui/
```

## Acceptance criteria

- [ ] `./gradlew assembleDebug` succeeds from a clean clone.
- [ ] APK installs and launches on an Android 13 (API 33) emulator (the placeholder `MainActivity` finishes immediately — that is expected).
- [ ] `AndroidManifest.xml` declares `READ_MEDIA_IMAGES` and `INTERNET`.
- [ ] No code in this task references types from `data`, `b2`, or `thumbnail` packages — those are owned by other tasks and may not yet exist.

## Notes

- Use the AWS SDK version pinned in the architecture doc (`1.6.46`). Newer 1.x versions may also work but the checksum quirk is verified at this version.
- If Gradle complains about `aws.smithy.kotlin:http-client-engine-okhttp` not resolving, double-check that the version is `1.0.0` and that `mavenCentral()` is in `settings.gradle.kts`.
