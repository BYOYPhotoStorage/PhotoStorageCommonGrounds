# Task 09 — Integration & Wire-up

> Read [`../mvp-implementation-overview.md`](../mvp-implementation-overview.md) first.

## Goal

Glue the components together and prove end-to-end the upload pipeline works against a real B2 bucket. This is the **last** task — start it only after Tasks 01–08 have produced their files.

## Files you own (touch)

- `app/src/main/java/com/hriyaan/photostorage/PhotoBackupApp.kt` (replace skeleton from Task 01)
- `app/src/main/java/com/hriyaan/photostorage/MainActivity.kt` (replace placeholder from Task 01)
- `app/src/main/AndroidManifest.xml` (verify launcher activity is `MainActivity`)
- (Optional) `app/src/androidTest/java/com/hriyaan/photostorage/SmokeTest.kt`

You should **not** modify files owned by Tasks 02–08. If a wiring problem requires a contract change, raise it back to the owning task — don't patch around it here.

## `PhotoBackupApp`

```kotlin
package com.hriyaan.photostorage

import android.app.Application
import com.hriyaan.photostorage.data.PrefsStore
import com.hriyaan.photostorage.data.UploadDatabase

class PhotoBackupApp : Application() {
    lateinit var prefsStore: PrefsStore
        private set
    lateinit var uploadDatabase: UploadDatabase
        private set

    override fun onCreate() {
        super.onCreate()
        prefsStore = PrefsStore(this)
        uploadDatabase = UploadDatabase(this)
    }
}
```

Singletons live for the process. Activities access them via `(application as PhotoBackupApp).prefsStore` etc.

## `MainActivity`

```kotlin
package com.hriyaan.photostorage

import android.content.Intent
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import com.hriyaan.photostorage.ui.GalleryActivity
import com.hriyaan.photostorage.ui.OnboardingActivity

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val app = application as PhotoBackupApp
        val target = if (app.prefsStore.hasCredentials()) GalleryActivity::class.java
                     else OnboardingActivity::class.java
        startActivity(Intent(this, target))
        finish()
    }
}
```

No layout — this is a pure routing activity.

## Verification: end-to-end smoke test

This is the moment we validate the architecture. Test against a **real B2 bucket** (use a throwaway bucket and a key restricted to it).

### Manual smoke checklist (run on device or emulator with internet)

1. Fresh install (uninstall first).
2. Launch app → routes to Onboarding.
3. Enter incorrect credentials → see "Invalid Application Key" or similar; app remains on Onboarding.
4. Enter correct credentials → routes to Gallery.
5. Tap a photo → spinner → cloud icon within ~10s on Wi-Fi for a 5 MB photo.
6. In B2 console, verify `photos/YYYY/MM/DD/<filename>` and `thumbnails/YYYY/MM/DD/<basename>.webp` both exist.
7. Force-stop and relaunch app → routes directly to Gallery, the previously-uploaded photo still has the cloud icon.
8. Toggle airplane mode, tap a not-yet-uploaded photo → cell shows failure state. Disable airplane mode, tap again → upload succeeds.

### B2 checksum sanity check

Inspect the network request once with a proxy (e.g. `mitmproxy` or `Charles`) and confirm the `PutObject` request **does not** include an `x-amz-checksum-crc32` header. If it does, B2 will return 400 and uploads silently fail — go back to Task 06 and verify `checksumAlgorithm = null` is actually being honored by the SDK version pinned in Task 01.

## Constraints

- Do not introduce new dependencies. Everything you need is already in Task 01's `build.gradle.kts`.
- Do not rewrite component implementations. If something doesn't fit, it's a contract bug; reopen the owning task.
- Do not log credentials, even during smoke testing.

## Acceptance criteria

- [ ] `MainActivity` routes correctly based on `PrefsStore.hasCredentials()`.
- [ ] `PhotoBackupApp.onCreate` initializes both singletons before any activity reads them.
- [ ] All seven manual smoke-checklist items pass on a real device.
- [ ] B2 console shows both objects (photo + thumbnail) at the expected keys.
- [ ] No `x-amz-checksum-crc32` header on `PutObject` requests.
- [ ] `./gradlew assembleDebug lint` is clean.

## Out of scope

- CI configuration.
- ProGuard / R8 rules tuning (defaults are sufficient for a debug build).
- Crash reporting.
- Anything in the PRD's "Out of Scope" list.
