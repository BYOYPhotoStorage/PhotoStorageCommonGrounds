# Phase 2 PRD — Auto-Upload & Resilience

_Last updated: 2026-05-16_

## Overview

Build the background upload pipeline that makes the app useful without manual intervention. The MVP proved the core flow works (tap → upload → thumbnail → track in SQLite). Phase 2 removes the tap: photos are detected automatically, queued, uploaded, and retried on failure. The user opens the app to see status, not to drive uploads.

This PRD covers the Android app repo (`BYOYPhotoStorage/PhotoStorageAndroidApp`).

---

## Goals

1. Detect new photos automatically using `MediaStore.ContentObserver`
2. Queue uploads reliably with SQLite, surviving app kills and reboots
3. Retry failed uploads with exponential backoff
4. Prevent duplicate uploads via two-level detection
5. Keep the user informed via Android notifications
6. Respect network constraints (Wi-Fi-only mode)

---

## Non-Goals

- Cloud browsing / merged gallery view modes (Phase 3)
- Deletion from B2 or local (Phase 3)
- SQLite index backup to B2 (Phase 3)
- Reinstall recovery flow (Phase 3)
- Video support (Phase 4)
- Upload scheduling (immediate vs nightly vs hybrid — Phase 4)
- Compression options (Phase 4)
- Settings screen UI beyond the Wi-Fi toggle (Phase 4)
- Encryption

---

## User Flow

### 1. First Launch After Update (One-Time Setup)

On app launch after Phase 2 install:
- Check if onboarding (B2 credentials) is complete. If not, send to onboarding.
- If credentials exist, check if auto-upload has been enabled:
  - If not yet enabled: show a one-time modal:
    - Title: "Enable automatic backup?"
    - Body: "We'll back up new photos as you take them. You can change this anytime in settings."
    - Buttons: "Enable" / "Not now"
  - On "Enable": set `auto_upload_enabled = true`, start the foreground service, trigger an initial gallery scan
  - On "Not now": set `auto_upload_enabled = false`, do not start the service. User can still tap photos to upload manually (MVP behavior preserved).

### 2. Background Upload Service (Foreground)

When `auto_upload_enabled = true`:
- A foreground service runs continuously with a persistent notification: "Photo backup is active"
- The service registers a `MediaStore.Images.Media.EXTERNAL_CONTENT_URI` `ContentObserver`
- When the observer fires (new photo taken, downloaded, restored from backup):
  1. Query MediaStore for the most recently added image(s) since last scan timestamp
  2. For each new photo, run duplicate detection
  3. If not a duplicate, insert into `upload_queue` table with status `pending`
  4. Trigger the upload worker

The foreground service also schedules a nightly `WorkManager` job:
- Runs at 02:00 local time (configurable hardcoded default)
- Constraints: requires network (unmetered for Wi-Fi-only users)
- Performs a full MediaStore scan to catch anything missed (photos added while app was killed, restored from backup, etc.)
- Any new photos found are enqueued the same way as the ContentObserver path

### 3. Upload Queue & Worker

A dedicated `UploadWorker` (or coroutine-based queue processor) processes the queue:

**Queue table schema** (extends MVP `uploads` table):

| Column | Type | Notes |
|--------|------|-------|
| id | INTEGER PRIMARY KEY | Auto-increment |
| local_uri | TEXT | MediaStore content URI |
| filename | TEXT | Original filename |
| size | INTEGER | File size in bytes |
| date_taken | INTEGER | Unix timestamp |
| photo_b2_path | TEXT | Target B2 path |
| thumbnail_b2_path | TEXT | Target B2 path |
| status | TEXT | `pending` / `uploading` / `uploaded` / `failed` |
| retry_count | INTEGER | Default 0, max 5 |
| next_retry_at | INTEGER | Unix timestamp, nullable |
| sha256 | TEXT | Computed on first read, stored for dedup |
| created_at | INTEGER | Unix timestamp |
| uploaded_at | INTEGER | Unix timestamp, nullable |

**Worker behavior:**
1. Query queue for `status = 'pending'` OR (`status = 'failed'` AND `next_retry_at <= now` AND `retry_count < 5`)
2. Order by `created_at ASC` (oldest first)
3. Process one at a time (or up to N concurrent, default 1 to avoid memory pressure)
4. For each item:
   - Set `status = 'uploading'`
   - Generate thumbnail
   - Upload original to B2
   - Upload thumbnail to B2
   - Set `status = 'uploaded'`, `uploaded_at = now`
   - Update notification with progress
5. On failure:
   - Increment `retry_count`
   - Set `next_retry_at = now + (2^retry_count * 60) seconds` (exponential backoff: 2min, 4min, 8min, 16min, 32min)
   - Set `status = 'failed'`
   - If `retry_count >= 5`, set `status = 'permanently_failed'` and show error notification

**On app cold start:**
- The worker checks the queue and resumes any pending or retryable items automatically
- No user action required

**On device reboot:**
- A `BroadcastReceiver` for `BOOT_COMPLETED` restarts the foreground service if `auto_upload_enabled = true`
- The worker resumes queue processing

### 4. Duplicate Detection

Two-level approach, run before inserting into queue:

**Level 1 (fast):** Compare `filename + size + date_taken` against existing `uploads` table records.
- If match found, skip enqueue. Log at debug level.

**Level 2 (hash):** If Level 1 does not match, compute SHA-256 of the original file.
- If `sha256` matches an existing record, skip enqueue.
- If no match, store the computed `sha256` in the queue record for future dedup.

Level 2 runs lazily: only compute the hash when actually about to upload, not on every ContentObserver fire. The hash is computed once and cached.

### 5. Notifications

| Scenario | Notification Type | Behavior |
|----------|-------------------|----------|
| Foreground service active | Persistent | "Backing up photos — 3 pending" (ongoing, cannot be dismissed) |
| Upload in progress | Persistent update | Updates with count: "Uploading photo 5 of 23" |
| Batch complete | Heads-up | "23 photos backed up" (auto-dismiss after 5s) |
| Single photo uploaded | Silent | No notification if only 1 photo and batch < 5 seconds |
| All caught up | Silent | No notification if queue drains to 0 |
| Retry scheduled | Silent | No notification (avoids noise) |
| Permanently failed | Heads-up | "5 photos failed to back up. Tap to retry." — tapping opens app to gallery |
| B2 auth failure | Heads-up | "Backup credentials expired. Tap to reconnect." |

Notification channel: `photo_backup` with default importance. The persistent foreground notification uses `IMPORTANCE_LOW` to avoid sound/vibration.

### 6. Wi-Fi-Only Mode

A toggle in settings (can be a minimal UI for now, full settings screen is Phase 4):
- Stored in `Encrypted SharedPreferences` as `wifi_only_uploads`, default `false`
- When `true`:
  - `ContentObserver` still detects photos and enqueues them
  - `UploadWorker` checks network constraints before uploading: if on metered network, skip and reschedule for next unmetered window
  - Nightly `WorkManager` job uses `NetworkType.UNMETERED` constraint
- When `false`: upload immediately on any network
- The foreground service notification shows a small icon indicator when Wi-Fi-only is active and the device is on mobile data (e.g., "Waiting for Wi-Fi — 12 pending")

---

## Technical Requirements

### Foreground Service
- `Service` subclass with `startForeground()` using `Notification.FOREGROUND_SERVICE_TYPE_DATA_SYNC`
- Must declare `android.permission.FOREGROUND_SERVICE` and `android.permission.FOREGROUND_SERVICE_DATA_SYNC` in manifest
- Service lifecycle:
  - Started on app launch if `auto_upload_enabled = true`
  - Started on `BOOT_COMPLETED` if enabled
  - Stopped only if user disables auto-upload
- The service itself does minimal work; it delegates to the upload worker/coroutine and holds the notification

### ContentObserver
- Register on `MediaStore.Images.Media.EXTERNAL_CONTENT_URI` with `notifyForDescendants = true`
- Debounce: if multiple fires occur within 3 seconds, batch them into a single scan
- On fire, query MediaStore for images with `DATE_ADDED > last_scan_timestamp`, ordered by `DATE_ADDED DESC`
- Update `last_scan_timestamp` to the newest `DATE_ADDED` seen
- Handle race condition: if observer fires while a nightly scan is in progress, enqueue is idempotent (duplicate detection handles it)

### WorkManager Nightly Scan
- `PeriodicWorkRequest` with `repeatInterval = 1 day`, `flexInterval = 1 hour`
- Initial delay: calculate time until 02:00 local time
- Constraints: requires network (unmetered if `wifi_only = true`)
- The worker performs the same scan logic as the ContentObserver path, then exits
- Does NOT perform uploads directly; it just enqueues items and the upload worker picks them up

### Upload Resilience
- Use the same B2 S3-compatible API from MVP (not native B2 API — see ADR)
- Uploads are sequential by default to avoid memory pressure on large photo sets
- Each upload is a single operation (no chunked/resumable uploads yet — out of scope)
- On network failure mid-upload: treat as failure, retry from the beginning. B2 handles duplicate overwrites.
- B2 token refresh is handled at the repository layer (same as MVP)

### Database Migration
- Migrate MVP `uploads` table to new schema:
  - Add `retry_count` (default 0)
  - Add `next_retry_at` (nullable)
  - Add `sha256` (nullable — backfill lazily on next read)
  - Add `created_at` (default to `uploaded_at` for existing rows, or current timestamp)
- Rename table or keep name; either way ensure `status` enum values are consistent

### Permissions
- Existing: `READ_MEDIA_IMAGES`, `INTERNET`
- New: `POST_NOTIFICATIONS` (Android 13+), `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_DATA_SYNC`, `RECEIVE_BOOT_COMPLETED`
- Request `POST_NOTIFICATIONS` at the same time as the auto-upload enable modal

---

## Success Criteria

- [ ] Taking a photo with the camera app results in automatic upload within 30 seconds (on Wi-Fi)
- [ ] Upload queue survives force-stop and reboot of the app/device
- [ ] Failed uploads retry with exponential backoff up to 5 times
- [ ] Duplicate photos (same filename+size+date, or same SHA-256) are not re-uploaded
- [ ] Persistent notification shows when service is active and updates with progress
- [ ] Wi-Fi-only mode prevents uploads on mobile data and queues them for next Wi-Fi connection
- [ ] Nightly scan catches photos that were missed while the app was killed
- [ ] Manual tap-to-upload from MVP still works when auto-upload is disabled
- [ ] No photos are lost or stuck in `uploading` state across app restarts

---

## Out of Scope (Post-Phase 2)

- Cloud browsing / merged gallery views
- Deletion from local or cloud
- SQLite index backup to B2
- Reinstall recovery
- Video support
- Upload scheduling modes (immediate / nightly / hybrid)
- Compression / quality options
- Full settings screen with all toggles
- Bandwidth limiting
- Cost dashboard
- Sharing

---

## Open Questions

1. Should the foreground service notification show a "Pause for 1 hour" action button?
2. Should permanently failed items be visible in the gallery with a retry button, or only in a separate "Failed uploads" screen?
3. How many concurrent uploads should we allow? (1 = safest, 2-3 = faster on fast networks but more memory)
4. Should we pre-compute SHA-256 during the scan (slower, more battery) or only at upload time (faster scan, but duplicate detection is slightly delayed)?
