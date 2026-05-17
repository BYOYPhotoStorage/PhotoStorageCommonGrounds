# Task 14 — Upload Worker & Retry Logic

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Create the worker that drains the upload queue, uploads photos + thumbnails to B2, handles failures with exponential backoff, and updates notifications.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/worker/UploadWorker.kt`

## Public contract

```kotlin
package com.hriyaan.photostorage.worker

import com.hriyaan.photostorage.data.UploadDao
import com.hriyaan.photostorage.data.UploadRecord
import com.hriyaan.photostorage.b2.S3Uploader
import com.hriyaan.photostorage.thumbnail.ThumbnailGen
import com.hriyaan.photostorage.notification.UploadNotificationManager

class UploadWorker(
    private val context: Context,
    private val uploadDao: UploadDao,
    private val s3Uploader: S3Uploader,
    private val thumbnailGen: ThumbnailGen,
    private val notificationManager: UploadNotificationManager
) {
    /** Processes all pending and retryable items in the queue. */
    suspend fun processQueue()
}
```

## Behavior

### `processQueue`

1. Check if `PrefsStore.isWifiOnly()` is true and the current network is metered. If so, skip processing entirely. Update the foreground notification to say "Waiting for Wi-Fi — N pending" and return.
2. Query the queue for items to process:
   - `uploadDao.getPendingQueue()`
   - `uploadDao.getFailedRetryable()`
3. Concatenate and sort by `created_at ASC` (oldest first).
4. If the queue is empty, update the foreground notification to "All caught up" and return.
5. Show a progress notification: "Uploading photo 1 of N".
6. For each item:
   a. Update the foreground notification with current progress.
   b. Set `status = 'uploading'` via `uploadDao.updateStatus`.
   c. Call `processSingle(record)`.
   d. On success: item is already marked `uploaded` by `processSingle`.
   e. On failure: `processSingle` already handles retry/failure marking.
7. After the loop:
   - If any items succeeded, show completion notification with the success count.
   - If any items are permanently failed, show permanent failure notification.
   - Cancel progress notification.
   - Update foreground notification with new pending count.

### `processSingle(record)` (private)

```kotlin
private suspend fun processSingle(record: UploadRecord): Result<Unit>
```

1. Parse `record.localUri` into a `Uri`.
2. Open the photo via `contentResolver.openInputStream(uri)`. If this fails, mark failed and return.
3. Compute `photoKey` and `thumbKey` via `S3KeyBuilder`.
4. Upload the original:
   - Read bytes from the input stream.
   - `s3Uploader.upload(key = photoKey, contentType = "image/jpeg", contentLength = bytes.size, body = ByteStream.fromBytes(bytes))`.
5. Generate and upload the thumbnail:
   - `thumbnailGen.createWebPThumbnail(uri)` → `thumbBytes`.
   - `s3Uploader.upload(key = thumbKey, contentType = "image/webp", contentLength = thumbBytes.size, body = ByteStream.fromBytes(thumbBytes))`.
6. On success:
   - `uploadDao.setUploadedPaths(record.id, photoKey, thumbKey, System.currentTimeMillis())`.
   - Return `Result.success(Unit)`.
7. On failure (any exception):
   - `val newRetryCount = record.retryCount + 1`
   - If `newRetryCount >= 5`:
     - `uploadDao.updateStatus(record.id, UploadDao.STATUS_PERMANENTLY_FAILED)`
     - Show permanent failure notification.
   - Else:
     - `val nextRetry = System.currentTimeMillis() + (2.0.pow(newRetryCount).toLong() * 60 * 1000)`
     - `uploadDao.updateRetry(record.id, newRetryCount, nextRetry)`
     - `uploadDao.updateStatus(record.id, UploadDao.STATUS_FAILED)`
   - Return `Result.failure(e)`.

## Retry schedule

| Retry # | Delay |
|---------|-------|
| 1 | 2 minutes |
| 2 | 4 minutes |
| 3 | 8 minutes |
| 4 | 16 minutes |
| 5 | 32 minutes → permanently failed |

Formula: `nextRetryAt = now + (2^retryCount * 60 * 1000) ms`

## Network checking

```kotlin
private fun isOnMeteredNetwork(): Boolean {
    val cm = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
    return cm.isActiveNetworkMetered
}
```

If `wifi_only = true` and `isOnMeteredNetwork() = true`, do not process. The item stays `pending` or `failed` and will be retried when Wi-Fi becomes available (either via the next ContentObserver fire or the nightly scan).

## Constraints

- Process items **sequentially** (one at a time). Parallel uploads are out of scope.
- All DB and network calls run in `Dispatchers.IO` (the caller — typically `UploadForegroundService` — launches this in IO).
- Do not hold the photo bytes in memory longer than necessary — upload and discard.
- Handle `SecurityException` when opening the URI (photo may have been deleted since enqueue).
- Handle `S3Exception` with 403 as an auth failure — show auth notification and do NOT retry (credentials are bad).

## Dependencies (by interface)

- `UploadDao` (Task 10) — queue queries and status updates
- `S3Uploader`, `S3KeyBuilder` (Task 06 / MVP) — B2 upload
- `ThumbnailGen` (Task 05 / MVP) — thumbnail generation
- `UploadNotificationManager` (Task 12) — notifications
- `PrefsStore` (Task 11) — `isWifiOnly()`

## Acceptance criteria

- [ ] `processQueue()` drains all `pending` and retryable `failed` items, oldest first.
- [ ] Each successful upload marks the record `uploaded` with both B2 paths set.
- [ ] Each failure increments `retry_count`, sets `next_retry_at`, and marks `failed`.
- [ ] After 5 failures, the record is marked `permanently_failed` and a notification is shown.
- [ ] When `wifi_only = true` and on mobile data, no uploads occur and the notification says "Waiting for Wi-Fi".
- [ ] A 403 S3Exception triggers the auth failure notification and marks the item `permanently_failed` (no retry).
- [ ] Progress notification updates as items are processed.
- [ ] Completion notification shows the number of successful uploads.

## Out of scope

- Parallel / concurrent uploads.
- Chunked / resumable uploads.
- Bandwidth limiting.
- Upload speed reporting.
