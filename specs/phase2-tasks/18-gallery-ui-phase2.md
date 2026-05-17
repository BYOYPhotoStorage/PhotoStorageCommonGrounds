# Task 18 — Gallery UI Phase 2

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Update the gallery screen to expose Phase 2 controls: an auto-upload enable/disable toggle, a Wi-Fi-only toggle, a status bar showing backup health, and the ability to retry permanently failed uploads.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/ui/GalleryActivity.kt` (modify)
- `app/src/main/res/layout/activity_gallery.xml` (modify — add status bar)
- `app/src/main/res/values/strings.xml` (modify — add Phase 2 strings)

## Layout additions

Add a status bar above the RecyclerView in `activity_gallery.xml`:

```xml
<!-- Inside CoordinatorLayout, above the RecyclerView -->
<LinearLayout
    android:id="@+id/statusBar"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:padding="12dp"
    android:background="?attr/colorSurfaceVariant"
    android:gravity="center_vertical"
    app:layout_behavior="@string/appbar_scrolling_view_behavior">

    <TextView
        android:id="@+id/statusText"
        android:layout_width="0dp"
        android:layout_weight="1"
        android:layout_height="wrap_content"
        android:textSize="14sp"
        tools:text="12 pending · Waiting for Wi-Fi" />

    <com.google.android.material.switchmaterial.SwitchMaterial
        android:id="@+id/autoUploadSwitch"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Auto"
        android:textSize="12sp" />

    <com.google.android.material.switchmaterial.SwitchMaterial
        android:id="@+id/wifiOnlySwitch"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Wi-Fi"
        android:textSize="12sp" />
</LinearLayout>
```

> Note: the `app:layout_behavior` should move from the RecyclerView to a parent container, or adjust the layout so the status bar is fixed and the RecyclerView scrolls beneath it. The simplest approach: wrap both in a `LinearLayout` (vertical) inside the `CoordinatorLayout`.

## Behavior

### Status bar initialization (`onCreate`)

After the existing MVP `onCreate` setup:

1. Find `statusBar`, `statusText`, `autoUploadSwitch`, `wifiOnlySwitch`.
2. Set `autoUploadSwitch.isChecked = prefsStore.isAutoUploadEnabled()`.
3. Set `wifiOnlySwitch.isChecked = prefsStore.isWifiOnly()`.
4. Update `statusText` based on current state (see "Status text logic" below).
5. If `autoUploadSwitch.isChecked`, call `UploadForegroundService.start(this)`.

### Auto-upload toggle

```kotlin
autoUploadSwitch.setOnCheckedChangeListener { _, isChecked ->
    prefsStore.setAutoUploadEnabled(isChecked)
    if (isChecked) {
        UploadForegroundService.start(this)
        NightlyScanScheduler.schedule(this)
        // Also trigger an immediate scan to catch up
        lifecycleScope.launch(Dispatchers.IO) {
            // Trigger the service's scan logic or just let the observer pick up
        }
    } else {
        UploadForegroundService.stop(this)
        NightlyScanScheduler.cancel(this)
    }
    updateStatusText()
}
```

### Wi-Fi-only toggle

```kotlin
wifiOnlySwitch.setOnCheckedChangeListener { _, isChecked ->
    prefsStore.setWifiOnly(isChecked)
    updateStatusText()
}
```

### Status text logic

```kotlin
private fun updateStatusText() {
    val pending = uploadDao.getPendingQueue().size
    val failed = uploadDao.getFailedRetryable().size
    val permanentlyFailed = uploadDao.getAll().count { it.status == UploadDao.STATUS_PERMANENTLY_FAILED }

    val parts = mutableListOf<String>()
    if (pending > 0) parts += "$pending pending"
    if (failed > 0) parts += "$failed retrying"
    if (permanentlyFailed > 0) parts += "$permanentlyFailed failed"
    if (parts.isEmpty()) parts += "All caught up"

    if (prefsStore.isAutoUploadEnabled() && prefsStore.isWifiOnly()) {
        parts += "· Wi-Fi only"
    }
    if (!prefsStore.isAutoUploadEnabled()) {
        parts += "· Manual mode"
    }

    statusText.text = parts.joinToString(" ")
}
```

> `getFailedRetryable` requires a `now` parameter — pass `System.currentTimeMillis()`.

Call `updateStatusText()` periodically (e.g., every 5 seconds while the activity is visible) using a `Handler` or coroutine:

```kotlin
private val statusRefreshJob: Job? = null

override fun onResume() {
    super.onResume()
    statusRefreshJob = lifecycleScope.launch {
        while (isActive) {
            updateStatusText()
            delay(5000)
        }
    }
}

override fun onPause() {
    super.onPause()
    statusRefreshJob?.cancel()
}
```

### Retry permanently failed uploads

Long-press on a gallery item with `status == permanently_failed` shows a dialog with a "Retry" option:

```kotlin
private fun onLongPress(item: GalleryItem) {
    if (item.record?.status == UploadDao.STATUS_PERMANENTLY_FAILED) {
        AlertDialog.Builder(this)
            .setTitle(R.string.info_title)
            .setMessage(/* existing info */)
            .setPositiveButton(R.string.retry) { _, _ ->
                lifecycleScope.launch(Dispatchers.IO) {
                    uploadDao.updateRetry(item.record.id, 0, null)
                    uploadDao.updateStatus(item.record.id, UploadDao.STATUS_PENDING)
                    UploadForegroundService.start(this@GalleryActivity)
                }
            }
            .setNegativeButton(R.string.cancel, null)
            .show()
    } else {
        // Existing long-press behavior (info dialog)
        showInfoDialog(item)
    }
}
```

### Request notification permission (Android 13+)

In `onCreate`, after permission checks:

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    if (ContextCompat.checkSelfPermission(this, Manifest.permission.POST_NOTIFICATIONS)
        != PackageManager.PERMISSION_GRANTED
    ) {
        requestPermissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
    }
}
```

## Constraints

- Do not block the main thread for DB queries — `updateStatusText()` reads from the DB, so call it from a coroutine or pre-fetch the counts.
- The auto-upload toggle must immediately start/stop the service — no delay.
- Existing MVP gallery behavior (tap to upload, long-press info) must continue to work.
- `POST_NOTIFICATIONS` permission is optional — if denied, the app still works but notifications are silently suppressed by the OS.

## Dependencies (by interface)

- `PrefsStore` (Task 11) — `isAutoUploadEnabled`, `setAutoUploadEnabled`, `isWifiOnly`, `setWifiOnly`
- `UploadDao` (Task 10) — `getPendingQueue`, `getFailedRetryable`, `updateRetry`, `updateStatus`
- `UploadForegroundService` (Task 13) — `start`, `stop`
- `NightlyScanScheduler` (Task 16) — `schedule`, `cancel`
- Existing MVP dependencies (Tasks 04, 06, 08)

## Acceptance criteria

- [ ] The gallery shows a status bar with pending/retrying/failed counts.
- [ ] The auto-upload switch reflects and controls `PrefsStore.isAutoUploadEnabled()`.
- [ ] Toggling auto-upload ON starts the foreground service and shows the persistent notification.
- [ ] Toggling auto-upload OFF stops the foreground service and removes the notification.
- [ ] The Wi-Fi-only switch reflects and controls `PrefsStore.isWifiOnly()`.
- [ ] Long-pressing a permanently failed photo shows a "Retry" button that resets retry count and re-enqueues.
- [ ] On Android 13+, the app requests `POST_NOTIFICATIONS` permission.
- [ ] Manual tap-to-upload still works when auto-upload is disabled.
- [ ] Status text updates every 5 seconds while the gallery is visible.

## Out of scope

- Animated status transitions.
- Pull-to-refresh for the gallery.
- Detailed upload history / log view.
