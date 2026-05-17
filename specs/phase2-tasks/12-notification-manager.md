# Task 12 — Notification Manager

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Set up Android notification channels and a helper class that the foreground service and upload worker use to show progress, completion, and error notifications.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/notification/UploadNotificationManager.kt`
- `app/src/main/java/com/hriyaan/photostorage/notification/NotificationChannels.kt`
- `app/src/main/res/drawable/ic_notification_upload.xml` (vector icon — a simple cloud or upload arrow)

## Public contract

```kotlin
package com.hriyaan.photostorage.notification

class UploadNotificationManager(private val context: Context) {
    /** Creates notification channels if they don't exist (call once, e.g. in Application.onCreate). */
    fun ensureChannels()

    /** Builds the persistent foreground notification. Used by UploadForegroundService. */
    fun buildForegroundNotification(pendingCount: Int): Notification

    /** Shows a non-persistent progress notification for an active batch upload. */
    fun showProgressNotification(current: Int, total: Int)

    /** Shows a heads-up notification that N photos were backed up. */
    fun showCompletionNotification(count: Int)

    /** Shows a heads-up notification that some uploads permanently failed. */
    fun showPermanentFailureNotification(failedCount: Int)

    /** Shows a heads-up notification that B2 auth failed. */
    fun showAuthFailureNotification()

    /** Cancels the progress notification. */
    fun cancelProgressNotification()
}
```

## Notification channels

Create two channels in `NotificationChannels`:

| Channel ID | Name | Importance | Behavior |
|------------|------|------------|----------|
| `upload_service` | "Photo backup" | `IMPORTANCE_LOW` | Persistent foreground notification. No sound, no vibration. |
| `upload_events` | "Backup events" | `IMPORTANCE_DEFAULT` | Completion, failure, auth errors. Standard notification behavior. |

## Implementation details

### Foreground notification (`buildForegroundNotification`)

```kotlin
NotificationCompat.Builder(context, CHANNEL_SERVICE)
    .setContentTitle("Photo backup is active")
    .setContentText("$pendingCount pending")
    .setSmallIcon(R.drawable.ic_notification_upload)
    .setOngoing(true)
    .setSilent(true)
    .build()
```

When `pendingCount == 0`, show `"All caught up"` instead of the count.

### Progress notification (`showProgressNotification`)

Use `NotificationCompat.Builder` with `.setProgress(total, current, false)` and `PRIORITY_LOW`. Auto-cancel when the batch finishes.

### Completion notification (`showCompletionNotification`)

Only show when `count >= 2` OR the batch took longer than 5 seconds. Single-photo quick uploads should be silent.

```kotlin
.setContentTitle("Photos backed up")
.setContentText("$count photos uploaded")
.setAutoCancel(true)
```

### Permanent failure notification (`showPermanentFailureNotification`)

```kotlin
.setContentTitle("Backup failed")
.setContentText("$failedCount photos could not be backed up. Tap to retry.")
.setContentIntent(/* PendingIntent to open GalleryActivity */)
.setAutoCancel(true)
```

### Auth failure notification (`showAuthFailureNotification`)

```kotlin
.setContentTitle("Backup credentials expired")
.setContentText("Tap to reconnect your Backblaze account.")
.setContentIntent(/* PendingIntent to open OnboardingActivity */)
.setAutoCancel(true)
```

## Constraints

- Use `NotificationCompat` from `androidx.core:core` (already available via appcompat).
- Channel creation is idempotent — safe to call `ensureChannels()` multiple times.
- Do not request `POST_NOTIFICATIONS` permission here; that is Task 17's responsibility in the manifest + GalleryActivity.
- Notification IDs: use constants, not magic numbers. Suggested:
  - Foreground: `1` (matches `startForeground(id, notification)`)
  - Progress: `2`
  - Completion: `3`
  - Permanent failure: `4`
  - Auth failure: `5`

## Acceptance criteria

- [ ] `ensureChannels()` creates two channels visible in Android Settings → Apps → Photo Backup → Notifications.
- [ ] `buildForegroundNotification(0)` returns a non-null `Notification` with `ongoing = true`.
- [ ] `buildForegroundNotification(5)` shows "5 pending" as content text.
- [ ] `showProgressNotification(3, 10)` displays a determinate progress bar.
- [ ] `showCompletionNotification(23)` displays a dismissible notification.
- [ ] `showPermanentFailureNotification(5)` includes a `PendingIntent` that launches `GalleryActivity`.
- [ ] All notifications use `ic_notification_upload` as the small icon.

## Out of scope

- Rich actions on notifications (pause, dismiss, etc.).
- Grouped / stacked notifications.
- Custom notification layouts.
