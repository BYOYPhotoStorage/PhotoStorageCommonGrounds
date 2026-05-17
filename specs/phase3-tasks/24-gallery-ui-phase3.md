# Task 24 — Gallery UI Phase 3

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Update the gallery screen to add a view-mode switcher (Local / Cloud / Merged), per-tile state badges, an index-sync status line, a selection mode for multi-select, and context-aware delete that dispatches to `DeletionEngine`. The Phase 2 status bar (auto-upload / Wi-Fi toggles, pending counts) is retained and extended.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/ui/GalleryActivity.kt` (modify)
- `app/src/main/java/com/hriyaan/photostorage/ui/GalleryAdapter.kt` (modify)
- `app/src/main/res/layout/activity_gallery.xml` (modify)
- `app/src/main/res/layout/item_gallery_tile.xml` (modify — add badge view)
- `app/src/main/res/values/strings.xml` (modify — Phase 3 strings)
- `app/src/main/res/menu/menu_gallery.xml` (new — overflow menu)

## Layout additions

### `activity_gallery.xml`

Add a `SegmentedButton`-style row (or a `MaterialButtonToggleGroup`) above the existing status bar:

```xml
<com.google.android.material.button.MaterialButtonToggleGroup
    android:id="@+id/viewModeGroup"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="8dp"
    app:singleSelection="true"
    app:selectionRequired="true">

    <com.google.android.material.button.MaterialButton
        android:id="@+id/modeLocal"
        style="?attr/materialButtonOutlinedStyle"
        android:layout_width="0dp"
        android:layout_weight="1"
        android:layout_height="wrap_content"
        android:text="@string/view_mode_local" />

    <com.google.android.material.button.MaterialButton
        android:id="@+id/modeCloud"
        style="?attr/materialButtonOutlinedStyle"
        android:layout_width="0dp"
        android:layout_weight="1"
        android:layout_height="wrap_content"
        android:text="@string/view_mode_cloud" />

    <com.google.android.material.button.MaterialButton
        android:id="@+id/modeMerged"
        style="?attr/materialButtonOutlinedStyle"
        android:layout_width="0dp"
        android:layout_weight="1"
        android:layout_height="wrap_content"
        android:text="@string/view_mode_merged" />
</com.google.android.material.button.MaterialButtonToggleGroup>
```

Add an `indexSyncStatus` TextView under the existing status bar:

```xml
<TextView
    android:id="@+id/indexSyncStatus"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="8dp"
    android:textSize="12sp"
    tools:text="Index last backed up 2 hours ago" />
```

### `item_gallery_tile.xml`

Overlay a small badge on each tile (bottom-right) — a circular icon-only chip:

```xml
<ImageView
    android:id="@+id/badge"
    android:layout_width="20dp"
    android:layout_height="20dp"
    android:layout_gravity="bottom|end"
    android:layout_margin="4dp"
    android:contentDescription="@string/tile_badge_desc"
    tools:src="@drawable/ic_badge_synced" />
```

Three drawables:
- `ic_badge_synced.xml` — small filled cloud + checkmark
- `ic_badge_cloud_only.xml` — outlined cloud
- `ic_badge_local_only.xml` — small phone icon (or pending/failed icons reusing Phase 2 assets if a `queuedRecord` is present)

### `menu/menu_gallery.xml`

```xml
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:id="@+id/action_backup_index"
        android:title="@string/action_backup_index_now"
        android:showAsAction="never" />
    <item
        android:id="@+id/action_restore_index"
        android:title="@string/action_restore_index"
        android:showAsAction="never" />
</menu>
```

## `GalleryAdapter` updates

- Bind to `GalleryItem` (the sealed type from Task 23) instead of the Phase 2 `UploadRecord`-keyed list.
- Resolve thumbnail loading via the Coil `ImageLoader` from Task 29 — accept it as a constructor param.
- Set the badge drawable based on the variant:
  - `Synced` → `ic_badge_synced`
  - `CloudOnly` → `ic_badge_cloud_only`
  - `LocalOnly` with `queuedRecord` of status `pending|uploading|failed` → existing Phase 2 status icon
  - `LocalOnly` otherwise → `ic_badge_local_only`
- Use `DiffUtil` on `GalleryItem.id` (Task 23's stable id contract).
- Selection: maintain a `Set<String>` of selected ids; the activity owns this set and pushes it into the adapter.

## `GalleryActivity` behavior

### `onCreate`

After existing Phase 2 setup:

1. Bind `viewModeGroup`. Initial selection comes from `prefsStore.getGalleryViewMode()` mapped to the toggle ID.
2. Bind the index-sync TextView and update its content from `prefsStore.getLastIndexSyncAt()`.
3. Construct/inject `galleryRepository` and `deletionEngine` (singletons from `PhotoBackupApp` — wired in Task 30).
4. Collect `galleryRepository.observe(currentMode())` in `lifecycleScope.launch { repeatOnLifecycle(STARTED) { ... } }` and pass the list to the adapter.

### Mode switching

```kotlin
viewModeGroup.addOnButtonCheckedListener { _, checkedId, isChecked ->
    if (!isChecked) return@addOnButtonCheckedListener
    val mode = when (checkedId) {
        R.id.modeLocal -> GalleryViewMode.LOCAL
        R.id.modeCloud -> GalleryViewMode.CLOUD
        else -> GalleryViewMode.MERGED
    }
    prefsStore.setGalleryViewMode(mode.key)
    currentMode = mode
    // Restart the observe(...) collection on the new mode — cancel and re-launch
}
```

### Selection mode

- Long-press a tile → enter selection mode, add that item to the selection set, swap the action bar for a contextual action bar with a delete icon and a count.
- Tap a tile while in selection mode → toggle its membership.
- Back / close action bar → exit selection mode.
- Delete icon in the action bar → dispatch to `DeletionEngine`:

```kotlin
private fun onDeleteSelected() {
    val items = selection.mapNotNull { id -> adapter.findById(id) }
    val mode = currentMode
    val title = when (mode) {
        GalleryViewMode.LOCAL -> getString(R.string.delete_local_title)
        GalleryViewMode.CLOUD -> getString(R.string.delete_cloud_title)
        GalleryViewMode.MERGED -> getString(R.string.delete_merged_title)
    }
    val body = when (mode) {
        GalleryViewMode.LOCAL -> getString(R.string.delete_local_body)
        GalleryViewMode.CLOUD -> getString(R.string.delete_cloud_body)
        GalleryViewMode.MERGED -> getString(R.string.delete_merged_body)
    }
    val positive = if (mode == GalleryViewMode.MERGED)
        getString(R.string.delete_both_button) else getString(R.string.delete_button)

    AlertDialog.Builder(this)
        .setTitle(title)
        .setMessage(body)
        .setPositiveButton(positive) { _, _ ->
            lifecycleScope.launch(Dispatchers.IO) {
                val result = deletionEngine.deleteBatch(items, mode)
                withContext(Dispatchers.Main) {
                    Toast.makeText(this@GalleryActivity, result.summary, Toast.LENGTH_LONG).show()
                    exitSelectionMode()
                }
            }
        }
        .setNegativeButton(R.string.cancel, null)
        .show()
}
```

### Index sync status line

```kotlin
private fun renderIndexSyncStatus() {
    val ts = prefsStore.getLastIndexSyncAt()
    indexSyncStatus.text = if (ts == null) {
        getString(R.string.index_never_synced)
    } else {
        getString(R.string.index_synced_at, DateUtils.getRelativeTimeSpanString(ts))
    }
}
```

Call from `onResume` and after a manual sync trigger completes.

### Overflow menu wiring

- `R.id.action_backup_index` → `IndexSyncScheduler.runNow(this)`, then refresh the status line on completion.
- `R.id.action_restore_index` → start `IndexRecoveryActivity` (Task 28) with a flag indicating "user-initiated restore" so it shows a confirmation that the local index will be replaced.

## New strings

```xml
<string name="view_mode_local">Local</string>
<string name="view_mode_cloud">Cloud</string>
<string name="view_mode_merged">Merged</string>
<string name="action_backup_index_now">Back up index now</string>
<string name="action_restore_index">Restore index from cloud</string>
<string name="index_never_synced">Index not yet backed up</string>
<string name="index_synced_at">Index last backed up %1$s</string>
<string name="delete_local_title">Delete from this device?</string>
<string name="delete_local_body">Cloud copy will remain.</string>
<string name="delete_cloud_title">Delete from cloud?</string>
<string name="delete_cloud_body">Local copy on this device will remain.</string>
<string name="delete_merged_title">Delete from phone and cloud?</string>
<string name="delete_merged_body">This will remove the photo from both your phone and your cloud backup. This cannot be undone.</string>
<string name="delete_button">Delete</string>
<string name="delete_both_button">Delete from both</string>
<string name="tile_badge_desc">Photo state</string>
```

## Constraints

- Mode switching must be instant — no network call. (Task 23 guarantees this.)
- Selection mode UX matches Android Material guidelines (contextual action bar with count).
- Delete confirmation copy is exactly the strings above (matches PRD wording).
- Do not block the main thread for the deletion call — always `Dispatchers.IO`.
- The tap-to-upload behavior from MVP and the Phase 2 manual-retry long-press flow must continue to work when selection mode is NOT active. (Long-press enters selection mode in Phase 3; the old "long-press shows info" dialog moves to the overflow menu on a single-selection item.)

## Dependencies (by interface)

- `GalleryRepository` (Task 23) — `observe`, `load`
- `DeletionEngine` (Task 25) — `delete`, `deleteBatch`
- `PrefsStore` (Task 21) — `getGalleryViewMode`, `setGalleryViewMode`, `getLastIndexSyncAt`
- `IndexSyncScheduler` (Task 27) — `runNow`
- `ThumbnailCacheFactory` (Task 29) — provides the Coil `ImageLoader` used by the adapter
- Phase 2 components remain wired (status bar, toggles)

## Acceptance criteria

- [ ] The mode switcher reflects `gallery_view_mode` on launch and writes it on change.
- [ ] Switching modes re-renders the gallery without a network call.
- [ ] Tile badges correctly show Synced / Cloud / Local states based on the variant.
- [ ] Long-press enters selection mode; the action bar shows a count and a delete icon.
- [ ] Tapping the delete icon shows the correct confirmation copy per mode.
- [ ] Merged-mode deletion uses the destructive "Delete from both" button (not the default neutral label).
- [ ] Result toast summarizes succeeded / refused / failed counts.
- [ ] The overflow menu shows "Back up index now" and "Restore index from cloud".
- [ ] Manual "Back up index now" triggers `IndexSyncScheduler.runNow` and refreshes the status line.
- [ ] Phase 2 status bar (auto-upload / Wi-Fi toggles, pending counts) continues to work.

## Out of scope

- Per-date headers / sticky timeline sections.
- Search bar.
- A full settings screen — overflow menu is sufficient for Phase 3.
- Drag-to-select.
- Bulk operations counting toward Wi-Fi-only constraints (handled by the engine and the existing Wi-Fi-only flag).
