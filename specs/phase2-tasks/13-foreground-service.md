# Task 13 — Foreground Service & ContentObserver

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Create a foreground service that runs while auto-upload is enabled, registers a `MediaStore.ContentObserver` to detect new photos in real time, and enqueues them for upload.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/service/UploadForegroundService.kt`

## Public contract

```kotlin
package com.hriyaan.photostorage.service

import android.app.Service
import android.content.Context
import android.content.Intent

class UploadForegroundService : Service() {
    companion object {
        fun start(context: Context) {
            ContextCompat.startForegroundService(
                context,
                Intent(context, UploadForegroundService::class.java)
            )
        }

        fun stop(context: Context) {
            context.stopService(Intent(context, UploadForegroundService::class.java))
        }
    }
}
```

## Behavior

### `onCreate`

1. Ensure notification channels exist via `UploadNotificationManager.ensureChannels()`.
2. Build the foreground notification ("Photo backup is active — X pending") and call `startForeground(1, notification)`.
3. Register a `ContentObserver` on `MediaStore.Images.Media.EXTERNAL_CONTENT_URI` with `notifyForDescendants = true`.
4. Start a coroutine scope (`ServiceScope` or `lifecycleScope` equivalent for services) to run the observer callback work off the main thread.

### ContentObserver callback

When the observer fires (a new photo was added to the gallery):

1. **Debounce:** If an observer fire was handled within the last 3 seconds, ignore this one. Use a `SharedFlow` or simple `Handler` with `removeCallbacksAndMessages` pattern.
2. After debounce, in `Dispatchers.IO`:
   a. Read `PrefsStore.getLastScanTimestamp()`.
   b. Query MediaStore for images with `DATE_ADDED > lastScanTimestamp / 1000` (MediaStore stores seconds, our timestamp is millis). Order by `DATE_ADDED DESC`.
   c. For each photo found, run it through `DuplicateDetector.isDuplicate()`.
   d. If not a duplicate:
      - Compute `photoB2Path` and `thumbnailB2Path` via `S3KeyBuilder`.
      - Insert into `uploads` table with `status = 'pending'`.
   e. Update `lastScanTimestamp` to the newest `dateTakenMs` seen (or `System.currentTimeMillis()` if no photos found, to prevent re-scanning the same window).
3. Trigger the upload worker to process the queue immediately.

### `onStartCommand`

Return `START_STICKY` so Android restarts the service if it is killed due to memory pressure.

### `onDestroy`

Unregister the `ContentObserver`.

### `onBind`

Return `null` — this is a started service, not a bound service.

## Implementation notes

### Service-scoped coroutines

Services don't have `lifecycleScope`. Use a `CoroutineScope(SupervisorJob() + Dispatchers.Main)` as a property, and cancel it in `onDestroy`:

```kotlin
private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
```

### Enqueue → trigger worker

After inserting into the queue, check if an upload is already in progress. If not, launch the `UploadWorker.processQueue()` coroutine within the service scope:

```kotlin
scope.launch {
    uploadWorker.processQueue()
}
```

Track in-flight state with a simple `private var isProcessing = false` boolean, guarded by a `Mutex` or simply checked/set atomically within the coroutine:

```kotlin
private val processLock = Mutex()

private fun triggerWorker() {
    scope.launch {
        processLock.withLock {
            if (isProcessing) return@withLock
            isProcessing = true
            try {
                uploadWorker.processQueue()
            } finally {
                isProcessing = false
            }
        }
    }
}
```

### Dependencies (by interface)

- `UploadDao` (Task 10) — for queue insertion and querying pending count
- `PrefsStore` (Task 11) — for `isAutoUploadEnabled()`, `getLastScanTimestamp()`, `setLastScanTimestamp()`
- `UploadNotificationManager` (Task 12) — for foreground notification
- `DuplicateDetector` (Task 15) — for deduplication before enqueue
- `MediaStoreQuery` (Task 04 / MVP) — for querying the gallery
- `UploadWorker` (Task 14) — for processing the queue
- `S3KeyBuilder` (Task 06 / MVP) — for computing target B2 paths

Construct all dependencies in `onCreate` (no DI framework):

```kotlin
private lateinit var uploadDao: UploadDao
private lateinit var prefsStore: PrefsStore
private lateinit var notificationManager: UploadNotificationManager
private lateinit var duplicateDetector: DuplicateDetector
private lateinit var mediaStoreQuery: MediaStoreQuery
private lateinit var uploadWorker: UploadWorker
```

## Constraints

- Do NOT query MediaStore on the main thread.
- Do NOT enqueue duplicates — always run `DuplicateDetector` first.
- The service must handle the case where `auto_upload_enabled` is toggled off while running (Task 18 or system can call `stop()`).
- `startForeground()` must be called within 5 seconds of `onCreate` or the app crashes (ANR).

## Acceptance criteria

- [ ] `UploadForegroundService.start(context)` starts the service and shows a persistent notification.
- [ ] Taking a photo with the camera app triggers the ContentObserver and enqueues the photo within 5 seconds.
- [ ] The debounce prevents multiple rapid fires from creating duplicate queue entries.
- [ ] `UploadForegroundService.stop(context)` removes the persistent notification and stops the service.
- [ ] `onStartCommand` returns `START_STICKY`.
- [ ] Killing the app and restarting it resumes the service if `auto_upload_enabled = true`.

## Out of scope

- Nightly scan (Task 16).
- Wi-Fi-only network checking (handled by UploadWorker in Task 14).
- Boot receiver (Task 17) — this task defines the service; Task 17 starts it on boot.
