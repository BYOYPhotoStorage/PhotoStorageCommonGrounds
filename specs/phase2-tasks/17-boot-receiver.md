# Task 17 — Boot Receiver & Manifest Updates

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Restart the foreground service after a device reboot, add required permissions to the manifest, and declare the new service, worker, and receiver.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/receiver/BootCompletedReceiver.kt`
- `app/src/main/AndroidManifest.xml` (modify — add permissions and declarations)

## Boot receiver

```kotlin
package com.hriyaan.photostorage.receiver

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import com.hriyaan.photostorage.data.PrefsStore
import com.hriyaan.photostorage.service.UploadForegroundService
import com.hriyaan.photostorage.worker.NightlyScanScheduler

class BootCompletedReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action != Intent.ACTION_BOOT_COMPLETED) return
        val prefsStore = PrefsStore(context)
        if (prefsStore.isAutoUploadEnabled()) {
            UploadForegroundService.start(context)
            NightlyScanScheduler.schedule(context)
        }
    }
}
```

## Manifest changes

### New permissions

Add these `uses-permission` entries:

```xml
<!-- Notifications -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<!-- Foreground service -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />

<!-- Boot -->
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
```

### Service declaration

```xml
<service
    android:name=".service.UploadForegroundService"
    android:exported="false"
    android:foregroundServiceType="dataSync" />
```

### Receiver declaration

```xml
<receiver
    android:name=".receiver.BootCompletedReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

### Existing activity declarations

Leave the existing `MainActivity`, `OnboardingActivity`, and `GalleryActivity` declarations untouched.

## Constraints

- The receiver must check `intent.action == Intent.ACTION_BOOT_COMPLETED` before doing anything.
- Do not start the service if `auto_upload_enabled = false` — respect the user's choice.
- The receiver runs on the main thread but only does cheap work (read prefs, start service, schedule work). No I/O.

## Dependencies (by interface)

- `PrefsStore` (Task 11) — `isAutoUploadEnabled()`
- `UploadForegroundService` (Task 13) — `start()`
- `NightlyScanScheduler` (Task 16) — `schedule()`

## Acceptance criteria

- [ ] `AndroidManifest.xml` declares all four new permissions.
- [ ] `AndroidManifest.xml` declares `UploadForegroundService` with `foregroundServiceType="dataSync"`.
- [ ] `AndroidManifest.xml` declares `BootCompletedReceiver` with the `BOOT_COMPLETED` intent filter.
- [ ] Rebooting the device with `auto_upload_enabled = true` starts the foreground service (verify via adb: `adb shell dumpsys activity services | grep UploadForegroundService`).
- [ ] Rebooting with `auto_upload_enabled = false` does NOT start the service.
- [ ] No existing manifest entries are removed or broken.

## Out of scope

- Handling `QUICKBOOT_POWERON` (some OEMs use this instead of `BOOT_COMPLETED`).
- Handling app updates (`MY_PACKAGE_REPLACED`) — the service is not persistent across updates.
