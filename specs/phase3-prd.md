# Phase 3 PRD — Gallery & Sync

_Last updated: 2026-05-16_

## Overview

Phase 2 made uploads automatic. The user now opens the app and sees an incomplete picture: the gallery only knows about photos currently on the device, and there is no way to see what is actually in B2. The SQLite index also lives only on the phone, so a reinstall or device swap means starting over.

Phase 3 closes those gaps. The gallery learns three view modes — Local, Cloud, and Merged — so the user has a continuous timeline that survives photos being removed from the device. Deletion becomes context-aware: deleting in Cloud view removes only the cloud copy, deleting in Merged view removes both with an explicit confirmation. The SQLite index is backed up to B2 nightly when it has changed, and on a reinstall the app can restore that index and ask the user how to catch up.

This PRD covers the Android app repo (`BYOYPhotoStorage/PhotoStorageAndroidApp`) and builds on the Phase 2 schema and services.

---

## Goals

1. Gallery supports three view modes: Local, Cloud, Merged (default)
2. Deletion respects the active view mode, with an explicit destructive confirmation in Merged
3. SQLite index is backed up to B2 nightly, only when it has changed since the last sync
4. After a reinstall, the app can restore the index from B2 and offer a catch-up path
5. Thumbnails fetched from B2 are cached locally so repeat viewing does not re-egress

---

## Non-Goals

- Multi-device sync with real-time conflict resolution
- Upload scheduling modes — immediate / nightly / hybrid (Phase 4)
- Video support (Phase 4)
- Full settings screen UI beyond what Phase 3 needs (Phase 4)
- Sharing / share links (Phase 4)
- Albums, search, or metadata editing
- Cost dashboard (Phase 4)
- Differential / incremental SQLite sync — Phase 3 uploads the whole file
- Backfilling the index from a B2 bucket that contains photos this app did not upload

---

## User Flow

### 1. Gallery View Modes

The gallery's app bar gains a mode switcher (segmented control or dropdown). Default is **Merged**. The selection persists in `Encrypted SharedPreferences` as `gallery_view_mode` (`local` / `cloud` / `merged`, default `merged`).

The existing backup status header from the MVP gallery (last backup time, pending count) is retained. In Phase 3 it also surfaces "Index last synced: …" when the index has been backed up.

**Local Only**
- Source: MediaStore query for all images on the device
- Shows what the device knows about, regardless of upload status
- Photos that were uploaded and later removed from the device do not appear here

**Cloud Only**
- Source: `uploads` table where `status = 'uploaded'` AND `cloud_deleted_at IS NULL`
- Thumbnails come from the local thumbnail cache, falling back to a B2 fetch on miss
- If a row has `local_present = true`, the local file may be used to render the thumbnail instead of fetching from B2 (faster, no egress)

**Merged (default)**
- Combines:
  - All `uploaded` rows from the `uploads` table (cloud-backed photos), AND
  - All local MediaStore photos with `date_taken` newer than the latest `uploaded_at` in the index
- Deduplicated by `(filename, size, date_taken)` falling through to SHA-256, reusing Phase 2 dedup keys
- Ordered by `date_taken DESC`
- Each tile shows a small badge for its state:
  - Cloud-only (uploaded, no longer on device)
  - Local-only (not yet uploaded, or upload pending/failed)
  - Synced (present locally and in cloud)

**Switching modes** is instant — all data is indexed locally, no network calls.

### 2. Context-Aware Deletion

Selection (long-press or multi-select) and the delete affordance behave the same across views. The confirmation copy and the actual side effects depend on the active view mode.

**Local view — delete locally only**
- Confirmation: "Delete from this device? Cloud copy will remain."
- On confirm: delete from MediaStore via `ContentResolver.delete()` (Android 11+: scoped storage `MediaStore.createDeleteRequest`)
- The row in `uploads` (if present) is updated to `local_present = false`
- If the photo had no cloud copy and no queue presence, the row is hard-deleted
- The photo disappears from Local view; remains visible in Cloud view if it was uploaded

**Cloud view — delete from cloud only**
- Confirmation: "Delete from cloud? Local copy on this device will remain."
- On confirm:
  1. Issue B2 S3-compatible DELETE for `photo_b2_path` and `thumbnail_b2_path` in parallel
  2. On success, set `status = 'cloud_deleted'` and `cloud_deleted_at = now` (soft-delete row, not hard-deleted yet)
  3. A daily cleanup pass hard-deletes rows older than 24 hours with `status = 'cloud_deleted'`
- Network failure: surface a retryable error, do not mutate the SQLite row, photo remains in Cloud view
- 404 from B2 is treated as success (already gone)

**Merged view — delete from both**
- Destructive confirmation pattern. Modal:
  - Title: "Delete from phone and cloud?"
  - Body: "This will remove the photo from both your phone and your cloud backup. This cannot be undone."
  - Buttons: cancel (default), "Delete from both" (destructive emphasis)
- On confirm: perform both the local delete and the cloud delete
- If only one succeeds (e.g., offline at the moment), surface a partial-failure error. The row reflects the partial state — the user can retry the failed side later from the appropriate view.

**Edge cases:**
- A photo currently in `uploading` state: from Cloud or Merged view, refuse with "Wait for upload to complete or cancel it from the queue." From Local view, allow local delete and also cancel the queued upload (remove from `upload_queue`).
- A photo with `status = 'permanently_failed'`: not deletable as a normal gallery item; surface in the existing Phase 2 failed-uploads affordance with "Discard" to clear from the queue.
- Multi-select that mixes states (e.g., some cloud-only, some local-only) in Merged view: per-item dispatch — apply the right operation to each, summarize the result ("3 deleted from both, 1 cloud-only deletion failed").

### 3. SQLite Index Sync to B2

The local SQLite database is the source of truth for what has been uploaded. Phase 3 backs it up to B2 so a reinstall does not lose that history.

**B2 path:** `index/photo-storage-index.sqlite` (overwritten on each sync)

**Sync triggers:**
- Nightly `WorkManager` job at **03:00 local time** (1 hour after the Phase 2 nightly scan so the scan finishes first)
- Manual trigger: "Back up index now" button (minimal settings affordance — full settings screen is Phase 4)
- A successful sync stores `last_synced_index_hash` and `last_index_sync_at` in `Encrypted SharedPreferences`

**Sync process:**
1. Flush WAL: `PRAGMA wal_checkpoint(TRUNCATE)`
2. Copy the live DB file to a temp file in `cacheDir` (avoid uploading a file being written to)
3. Compute SHA-256 of the temp file
4. If equal to `last_synced_index_hash`, exit — no upload needed
5. Otherwise, upload the temp file to `index/photo-storage-index.sqlite` via the same B2 S3 API used for photos
6. On success: update `last_synced_index_hash` and `last_index_sync_at`
7. On failure: keep the old hash, retry on the next scheduled run (no manual retry loop in Phase 3)

**Constraints:**
- Network: requires unmetered if `wifi_only_uploads = true` (Phase 2 setting); otherwise any network
- Charging: not required (file is small — single-digit MB at the 10K-photo target scale)

**Visibility:** The gallery status header and the minimal settings affordance show "Index last backed up: 2026-05-15 03:02" (relative time when recent, absolute otherwise).

### 4. Reinstall Recovery Flow

Triggered when the user installs the app fresh or clears its data. Happens immediately after the existing MVP onboarding step that collects and validates B2 credentials.

**Step A — Check for an existing index**
- After credential validation, issue a B2 HEAD on `index/photo-storage-index.sqlite`
- If not present: skip recovery, proceed to Step C with an empty local DB
- If present: show recovery prompt
  - Title: "Restore your previous backup index?"
  - Body: "We found an index from [date]. Restoring lets you see your existing cloud photos in the gallery without rescanning everything."
  - Buttons: "Restore" (primary), "Start fresh"

**Step B — Restore the index** (if user chose Restore)
1. Download `index/photo-storage-index.sqlite` to a temp file with a progress indicator
2. Validate: open as SQLite, check expected tables exist
3. Run Room migrations against the downloaded DB if its `user_version` is older than the app's current schema
4. Replace the local DB file with the validated, migrated copy (close the open `RoomDatabase`, swap files, reopen)
5. Confirm: "Restored [N] photos from your previous backup."
- If validation or migration fails: show "We could not restore the index. You can start fresh and the app will rescan your gallery." with a "Start fresh" fallback

**Step C — Catch-up prompt**
- After restore (or after skipping):
  - Title: "What should we back up?"
  - Body, when an index was restored: "Your index has photos up to [last_uploaded_at]."
  - Options:
    - **"Catch up from [date]"** (default when restoring a non-empty index) — MediaStore scan filtered to `date_taken > last_uploaded_at`, enqueue anything not already in `uploads` via Phase 2 dedup
    - **"Full rescan"** — scan the entire MediaStore, dedup against the restored index, enqueue anything not present
    - **"Skip for now"** — do nothing; the Phase 2 nightly scan will eventually catch up

**Step D — Resume Phase 2 services**
- Start the foreground service if `auto_upload_enabled = true`
- Register `ContentObserver`, schedule the nightly scan and the nightly index sync

**Re-running recovery later:**
- If the user picked "Start fresh" but later wants their index back, expose a "Restore index from cloud" action in settings. Same flow as Step B, with a warning that the local index will be replaced.

### 5. Thumbnail Caching

Cloud-only thumbnails (Cloud view, and cloud-only items in Merged view) are fetched from B2. To avoid repeated egress:

**Cache location:** `context.cacheDir/thumbnails/` — Android-managed, OS may clear under storage pressure (acceptable: misses re-fetch).

**Cache key:** SHA-256 of `thumbnail_b2_path` (stable across app sessions).

**Cache strategy:**
- Disk LRU with a 200 MB cap (hardcoded default for Phase 3)
- On miss: fetch from B2, decode, write to cache, return
- On hit: stream from disk
- Implementation preference: Coil or Glide with a custom `Fetcher` that signs B2 S3 requests via the existing repository, rather than rolling our own pipeline

**Prefetch:**
- When the user is within ~10 tiles of the end of the rendered window in Cloud or Merged view, prefetch the next ~20 thumbnails
- Prefetch is suppressed when `wifi_only_uploads = true` and the device is on a metered network

---

## Technical Requirements

### Database Migration

Migrate the Phase 2 `uploads` schema:
- Add `local_present` BOOLEAN, default `true`
- Add `cloud_deleted_at` INTEGER, nullable
- Extend `status` enum to include `cloud_deleted`
- No data migration needed for existing rows (defaults are correct for the post-Phase-2 state)

A daily cleanup `WorkManager` job hard-deletes rows where `status = 'cloud_deleted' AND cloud_deleted_at < now - 24h`.

### Gallery Data Source

- A single `GalleryRepository` exposes `Flow<List<GalleryItem>>` parameterized by view mode
- `GalleryItem` is a sealed type: `LocalOnly`, `CloudOnly`, `Synced` — UI binds the badge from the variant
- Merged view computation: load `uploads` rows + a MediaStore cursor, merge in memory on a background dispatcher, dedup, sort. Result is cached for the lifetime of the screen and invalidated on `uploads` table writes or `ContentObserver` fires.
- At the 10K-photo scale (Q8), in-memory merge is fine; revisit pagination if the dataset grows.

### B2 Delete API

- Reuse the S3-compatible client from MVP / Phase 2
- Each photo deletion = two DELETE calls (`photos/...` + `thumbnails/...`) issued in parallel
- 404 → treat as success; non-404 4xx → permanent failure surfaced as a non-retryable error; 5xx / network → retryable error surfaced to the user

### SQLite Index Upload

- Reuse the Phase 2 B2 upload code path
- Always upload from a temp copy, never from the live DB file
- The hash check is the only de-dup mechanism for the index — no ETag / conditional PUT in Phase 3

### Reinstall Recovery

- New screen / route after the existing credentials step: `IndexRecoveryActivity` (or Compose equivalent)
- B2 HEAD to detect the index file; download via the existing B2 client with progress
- Room database swap requires closing the open `RoomDatabase`, replacing the file, and re-opening. Guard with a mutex to ensure no writes happen during the swap.
- If the downloaded DB's schema version is newer than the app's, fail with "Please update the app to restore this backup" and offer "Start fresh" as fallback

### Permissions

No new permissions beyond Phase 2. (Local delete uses scoped-storage delete-request APIs on Android 11+, which prompt the user inline — no manifest change needed.)

---

## Success Criteria

- [ ] User can switch between Local, Cloud, and Merged views; selection persists across app restarts
- [ ] Merged view shows a continuous timeline with correct badges for local-only, cloud-only, and synced photos
- [ ] Deleting in Local view removes only the local copy; the cloud copy persists and is visible in Cloud view
- [ ] Deleting in Cloud view removes only the cloud copy; the local file is untouched and visible in Local view
- [ ] Deleting in Merged view shows the destructive confirmation and removes both copies
- [ ] Partial-failure deletes (e.g., offline mid-Merged-delete) surface a clear error and leave consistent state
- [ ] SQLite index uploads to B2 nightly when changed, and the upload is skipped when unchanged
- [ ] Manual "Back up index now" works
- [ ] After reinstall + B2 credential entry, the app detects the existing index and offers to restore it
- [ ] Restored index correctly populates Cloud view without a full rescan
- [ ] Catch-up prompt enqueues the right set of photos (new since `last_uploaded_at` for catch-up; the full delta for full rescan)
- [ ] Cloud thumbnails are cached locally; a second view of the same photo does not hit B2
- [ ] All Phase 2 success criteria continue to pass

---

## Out of Scope (Post-Phase 3)

- Multi-device conflict resolution — last-write-wins on the index file if two devices upload simultaneously
- Trash / recently-deleted view spanning longer than the 24h soft-delete window
- Bulk operations across thousands of photos in a single gesture
- Multiple B2 buckets per app instance (Q14 — one bucket per instance)
- Differential / incremental index sync
- Index sync to alternative providers
- Search, albums, metadata editing

---

## Open Questions

1. Should the SQLite index file be encrypted before upload? B2 at-rest is on (Q3 — no client-side encryption), but the index contains filenames and SHA-256 hashes that some users may consider more sensitive than the photos themselves. - Ans: No need for encryption
2. In Merged view, when a photo exists both locally and in cloud, render once (deduped) is assumed — confirm. - Ans: Yes render once only
3. Should the catch-up prompt remember the user's last choice across reinstalls, or always ask? - Ans: Always ask no need to remember
4. Is 24h the right soft-delete window for cloud deletes, or should it stretch (e.g., 7 days) for accident recovery? Ans: yes 24h is fine
5. Is 200 MB the right thumbnail cache cap, or should it scale to available device storage? Ans: yes 200 MB is good
6. After restoring a foreign-device index, should we trust `local_present` from that DB (likely false everywhere) or re-derive it from the current MediaStore? Ans: So we can merge it with time lines 
	1. -inf <---> tmax_foreign-device  ==> foreign-device index
	2. tmax_foreign-device <--> current_time ==> current MediaStore
