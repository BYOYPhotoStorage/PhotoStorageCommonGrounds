# Task 39 — Sharing

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Generate time-limited B2 presigned URLs for individual photos and videos. Surface a "Share link" affordance in the gallery's long-press menu (cloud-backed items only). Persist issued links so the user can find them later on an "Active share links" screen. No revoke — B2 presigned URLs cannot be revoked; the UI is explicit about this.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/share/ShareLinkService.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/share/ShareLinkTtl.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/ui/ShareLinkDialog.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/ui/ActiveShareLinksActivity.kt` (new)
- `app/src/main/res/layout/dialog_share_link.xml` (new)
- `app/src/main/res/layout/activity_active_share_links.xml` (new)
- `app/src/main/java/com/hriyaan/photostorage/ui/GalleryActivity.kt` (modify — add "Share link" entry to long-press action menu)
- `app/src/main/res/values/strings.xml` (modify — add this task's keys only)

## `ShareLinkTtl`

```kotlin
enum class ShareLinkTtl(val seconds: Long, val labelRes: Int) {
    ONE_HOUR(3_600L, R.string.share_ttl_1h),
    ONE_DAY(86_400L, R.string.share_ttl_24h),
    ONE_WEEK(604_800L, R.string.share_ttl_7d);

    fun expiryFromNow(now: Long = System.currentTimeMillis()): Long = now + seconds * 1_000L
}
```

The TTL choices are hardcoded for Phase 4. Adding custom TTLs is a future feature.

## `ShareLinkService` contract

```kotlin
class ShareLinkService(
    private val s3Uploader: S3Uploader,
    private val shareLinkDao: ShareLinkDao
) {
    /**
     * Creates and persists a presigned link for the given gallery item.
     * Returns Result.failure if the item is not cloud-backed or signing fails.
     */
    suspend fun createLink(item: GalleryItem, ttl: ShareLinkTtl): Result<ShareLinkRecord>

    /**
     * Returns active links — non-expired plus expired-within-last-24h (so the UI
     * can show a brief tombstone before they vanish).
     */
    suspend fun activeLinks(now: Long = System.currentTimeMillis()): List<ShareLinkRecord>
}
```

### `createLink` semantics

1. Resolve the upload record id and the B2 photo path:
   ```kotlin
   val (uploadId, b2Path) = when (item) {
       is GalleryItem.CloudOnly -> item.uploadRecord.id to (item.uploadRecord.photoB2Path ?: return Result.failure(...))
       is GalleryItem.Synced -> item.uploadRecord.id to (item.uploadRecord.photoB2Path ?: return Result.failure(...))
       is GalleryItem.LocalOnly -> return Result.failure(IllegalStateException("Not cloud-backed"))
   }
   ```
2. Call `s3Uploader.presignGetUrl(b2Path, ttl.seconds).getOrElse { return Result.failure(it) }`.
3. Build a `ShareLinkRecord` with `createdAt = now`, `expiresAt = ttl.expiryFromNow()`.
4. Persist via `shareLinkDao.insert(record)` and re-fetch the assigned id.
5. Return `Result.success(record.copy(id = newId))`.

### `activeLinks` semantics

Use `shareLinkDao.getActive(now)` — which already keeps rows visible for 24h past expiry (Task 31).

## `ShareLinkDialog`

A `DialogFragment` that shows the TTL chooser and the action buttons.

### Layout

`dialog_share_link.xml`:

```xml
<LinearLayout android:orientation="vertical" android:padding="24dp">
  <TextView android:text="@string/share_link_title" android:textSize="20sp" android:textStyle="bold" .../>
  <TextView android:text="@string/share_link_body" android:paddingTop="8dp" .../>

  <RadioGroup android:id="@+id/ttl_group" android:paddingTop="16dp">
    <RadioButton android:id="@+id/ttl_1h"  android:text="@string/share_ttl_1h"  android:checked="true" />
    <RadioButton android:id="@+id/ttl_24h" android:text="@string/share_ttl_24h" />
    <RadioButton android:id="@+id/ttl_7d"  android:text="@string/share_ttl_7d"  />
  </RadioGroup>

  <LinearLayout android:orientation="horizontal" android:gravity="end" android:paddingTop="16dp">
    <Button android:id="@+id/cancel"  android:text="@string/share_link_cancel"  style="?android:attr/buttonBarButtonStyle"/>
    <Button android:id="@+id/create"  android:text="@string/share_link_create"  style="?android:attr/buttonBarButtonStyle"/>
  </LinearLayout>
</LinearLayout>
```

### Behavior

```kotlin
class ShareLinkDialog : DialogFragment() {
    companion object {
        const val ARG_ITEM_ID = "item_id"
        fun newInstance(itemId: String) = ShareLinkDialog().apply {
            arguments = bundleOf(ARG_ITEM_ID to itemId)
        }
    }

    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog {
        val view = layoutInflater.inflate(R.layout.dialog_share_link, null)
        val dialog = AlertDialog.Builder(requireContext()).setView(view).create()
        view.findViewById<Button>(R.id.cancel).setOnClickListener { dismiss() }
        view.findViewById<Button>(R.id.create).setOnClickListener { onCreate(view) }
        return dialog
    }

    private fun onCreate(view: View) {
        val ttl = when (view.findViewById<RadioGroup>(R.id.ttl_group).checkedRadioButtonId) {
            R.id.ttl_24h -> ShareLinkTtl.ONE_DAY
            R.id.ttl_7d -> ShareLinkTtl.ONE_WEEK
            else -> ShareLinkTtl.ONE_HOUR
        }
        val itemId = requireArguments().getString(ARG_ITEM_ID)!!
        val app = requireActivity().application as PhotoBackupApp
        viewLifecycleOwner.lifecycleScope.launch {
            val item = app.galleryRepository.findById(itemId)
                ?: run { toast(R.string.share_link_error_not_found); dismiss(); return@launch }
            val result = app.shareLinkService.createLink(item, ttl)
            result
                .onSuccess { record ->
                    copyToClipboard(record.url)
                    toast(getString(R.string.share_link_created, ttl.labelText()))
                    startActivity(buildShareIntent(record.url))
                }
                .onFailure { toast(R.string.share_link_error_generic) }
            dismiss()
        }
    }

    private fun buildShareIntent(url: String): Intent =
        Intent.createChooser(
            Intent(Intent.ACTION_SEND).apply {
                type = "text/plain"
                putExtra(Intent.EXTRA_TEXT, url)
            },
            getString(R.string.share_link_chooser_title)
        )
}
```

`GalleryRepository.findById(id)` is a small helper this task expects to exist or asks Task 23's owner — wait, we are sequential and Task 23 (Phase 3) is already done. Either it already exists (likely — repositories typically have `findById`) or this task adds a one-line helper to `GalleryRepository`. To keep ownership clean, add the helper as an extension function in the `share/` package:

```kotlin
suspend fun GalleryRepository.findItemById(id: String): GalleryItem? =
    load(GalleryViewMode.MERGED).firstOrNull { it.id == id }
```

This avoids modifying Task 23's owned file.

## `ActiveShareLinksActivity`

Lists active links. Each row shows: filename (from the joined `uploads` row), created-at relative time, expires-at relative time, a "Copy" button, and an "Open in browser" button.

### Layout

`activity_active_share_links.xml`:

```xml
<LinearLayout android:orientation="vertical">
  <TextView android:text="@string/share_links_active_title" android:textSize="20sp" android:textStyle="bold" android:padding="16dp"/>
  <TextView android:text="@string/share_links_disclaimer" android:padding="16dp" android:textColor="?android:attr/textColorSecondary" android:textSize="13sp"/>
  <androidx.recyclerview.widget.RecyclerView
      android:id="@+id/list"
      android:layout_width="match_parent"
      android:layout_height="0dp"
      android:layout_weight="1" />
  <TextView android:id="@+id/empty" android:visibility="gone" android:text="@string/share_links_empty" android:gravity="center" android:padding="24dp"/>
</LinearLayout>
```

### Behavior

```kotlin
class ActiveShareLinksActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_active_share_links)
        val app = application as PhotoBackupApp
        lifecycleScope.launch {
            val links = app.shareLinkService.activeLinks()
            if (links.isEmpty()) {
                findViewById<View>(R.id.empty).visibility = View.VISIBLE
            } else {
                renderList(links)
            }
        }
    }
}
```

Per row, "Copy" calls `ClipboardManager.setPrimaryClip(...)` with the URL; "Open in browser" launches an `Intent.ACTION_VIEW` against the URL. A row whose `expires_at < now` displays as muted with a "(expired)" suffix and the buttons hidden.

## GalleryActivity changes

Add a "Share link" entry to the existing long-press selection-mode action menu (introduced in Phase 3 Task 24). Visibility rules:

- Single-selection only — disable the entry when more than one item is selected.
- Cloud-only or Synced items only — disable for `LocalOnly` (no cloud copy).

```kotlin
private fun onShareLinkClicked() {
    val selected = selectionState.selectedItems
    if (selected.size != 1) return
    val item = selected.first()
    if (item is GalleryItem.LocalOnly) {
        toast(R.string.share_link_local_only_warning)
        return
    }
    ShareLinkDialog.newInstance(item.id).show(supportFragmentManager, "share_link")
}
```

The action menu wiring (toolbar item, click listener) is identical in shape to the existing "Delete" entry — copy that scaffolding.

## Strings

```xml
<string name="share_link_title">Share link</string>
<string name="share_link_body">Anyone with this link can view the photo until it expires.</string>
<string name="share_ttl_1h">1 hour</string>
<string name="share_ttl_24h">24 hours</string>
<string name="share_ttl_7d">7 days</string>
<string name="share_link_cancel">Cancel</string>
<string name="share_link_create">Create link</string>
<string name="share_link_created">Link copied. Expires in %1$s.</string>
<string name="share_link_chooser_title">Share link</string>
<string name="share_link_error_not_found">Photo no longer available.</string>
<string name="share_link_error_generic">Could not create link. Try again later.</string>
<string name="share_link_local_only_warning">This photo is not in the cloud yet.</string>
<string name="share_links_active_title">Active share links</string>
<string name="share_links_disclaimer">Links cannot be revoked once shared. They expire automatically. Use short durations for sensitive photos.</string>
<string name="share_links_empty">No active links.</string>
```

## Implementation notes

- The dialog uses `AlertDialog` rather than Material because the project's existing dialogs (Phase 3 delete confirmation) use vanilla `AlertDialog`. Match the precedent.
- Never log the URL at any level. If logging the share event is desirable, log `(uploadId, ttl)` only.
- The active-links list is a one-shot fetch on activity start. No live `Flow` subscription — links are append-only within the activity's lifetime.
- For expired-but-visible rows, "Open in browser" remains disabled (the link will 403); only the visible "Copy" stays interactive for the user's own records.

## Constraints

- Single-item only in Phase 4. Multi-selection share is out of scope (would need an album concept first).
- Never extend or refresh an existing link — issuing a new link is the only path (PRD §6 constraint).
- Sharing for `LocalOnly` items falls back to the standard Android share sheet pointing at the local file. That existing behavior (or the lack of it) is *not* modified by Phase 4. If Phase 3 already delegates `LocalOnly` to a system share intent, leave that alone; this task only adds the cloud-presign path.
- No watermarking, no analytics, no access logging — none of those are possible without a server.

## Dependencies (by interface)

- `S3Uploader.presignGetUrl` (Task 33)
- `ShareLinkDao` (Task 31)
- `ShareLinkRecord` (Task 31)
- `GalleryRepository` (Phase 3 Task 23) — for `findItemById` extension helper
- `GalleryItem` (Phase 3 Task 23) — sealed type discrimination

## Acceptance criteria

- [ ] Long-pressing a `Synced` or `CloudOnly` tile and tapping "Share link" opens the TTL dialog.
- [ ] Long-pressing a `LocalOnly` tile and tapping "Share link" shows a toast and does not open the dialog.
- [ ] Selecting "1 hour" / "24 hours" / "7 days" and tapping "Create link" produces a presigned URL that opens the photo in a browser for the chosen duration.
- [ ] The created URL is copied to clipboard and the system share sheet is launched.
- [ ] The new row appears in `share_links` and on the Active share links screen.
- [ ] An expired link is shown muted with "(expired)" for 24 hours and then disappears from the list.
- [ ] No URL appears in logcat at any log level.
- [ ] The disclaimer copy on the active-links screen mentions non-revocability.

## Out of scope

- Multi-item / album sharing.
- Server-mediated share pages (analytics, access logs, watermark).
- Editing the TTL of an existing link.
- "Public gallery" mode.
- Custom TTL beyond the three presets.
- Sharing a thumbnail-only URL (always shares the original).
