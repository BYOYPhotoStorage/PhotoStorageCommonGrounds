# Phase 4 PRD — Settings & Polish

_Last updated: 2026-05-17_

## Overview

Phase 2 made uploads automatic. Phase 3 made the gallery aware of cloud state and the index survivable across reinstalls. The app now works silently in the background, but every behavior decision the app makes is hardcoded: it uploads immediately on any allowed network, it ignores videos entirely, it never reclaims space on the device, and the user has no way to see what the backup is costing them or share a single photo with someone else.

Phase 4 is the "controls and visibility" phase. It introduces a real settings screen, a proper first-backup flow that asks the user how much history to back up before the upload pipeline starts, configurable upload timing (immediate / scheduled / hybrid), full video support with quality and duration rules, configurable local-deletion strategies, a B2 cost dashboard, and time-limited share links for individual photos.

This PRD covers the Android app repo (`BYOYPhotoStorage/PhotoStorageAndroidApp`) and builds on the Phase 2 services and the Phase 3 schema, repositories, and recovery flow.

---

## Goals

1. Offer an explicit first-backup choice (today only vs. entire gallery history) before auto-upload starts on a fresh install
2. Let the user pick the upload timing mode: immediate, scheduled (nightly), or hybrid (Wi-Fi triggers immediate, mobile data defers to nightly)
3. Back up videos with configurable quality (original, compressed, duration-based) and the option to disable video uploads entirely
4. Reclaim local storage with a configurable auto-deletion strategy (never / immediate / after N days / after N new photos)
5. Surface estimated B2 storage cost and usage breakdown
6. Generate time-limited share links for individual photos
7. Consolidate every toggle behind a proper Settings screen instead of the minimal affordances accumulated in Phases 2–3

---

## Non-Goals

- Album / multi-photo share collections (no album concept exists yet — sharing is per-photo in Phase 4)
- Real-time multi-device sync or shared buckets (Q14: one bucket per app instance)
- Bandwidth throttling / monthly upload quota warnings
- Multiple B2 buckets per app instance
- Client-side encryption (Q3 — B2 at-rest is sufficient)
- Face detection, tagging, search, EXIF editing
- RAW / DNG support
- Other S3-compatible providers (Wasabi, Storj, MinIO)
- Desktop companion app
- Public gallery / always-on share pages
- Migration of historical uploads to use new video paths (Phase 4 starts videos fresh)

---

## User Flow

### 1. First-Backup Flow (P0)

Triggered on a fresh install after the existing onboarding step that collects and validates B2 credentials, and after the Phase 3 reinstall-recovery branch has determined there is no remote index to restore (or the user chose "Start fresh").

The Phase 2 "Enable automatic backup?" modal is replaced by a short three-step onboarding tail:

**Step A — Enable auto-backup**
- Title: "Back up new photos automatically?"
- Body: "We'll watch your gallery and upload new photos in the background."
- Buttons: "Enable" (primary), "Not now"
- On "Not now": skip Steps B and C, set `auto_upload_enabled = false`, finish onboarding. (User can still tap photos manually — MVP behavior preserved.)

**Step B — Choose backup scope**
- Shown only if Step A is "Enable"
- Title: "What should we back up?"
- Body: "Just photos taken from now on, or your entire gallery history?"
- Options (single-select):
  - **"From today only"** (default) — only new MediaStore entries with `date_added >= now` will be enqueued
  - **"Entire gallery history"** — a one-time backfill scan enqueues every photo (and every video, if Step C enables them) currently in MediaStore that is not already in `uploads`
- The choice is recorded in `Encrypted SharedPreferences` as `first_backup_scope` (`today` / `all`) for diagnostics; it is not re-prompted later

**Step C — Include videos**
- Title: "Back up videos too?"
- Body: "Videos can use much more space than photos. You can change this in settings later."
- Toggle, default off. Sets `videos_enabled` (Phase 4 introduces this preference — see §3)

**Finish**
- Set `auto_upload_enabled = true`
- If Step B = "Entire gallery history": start a one-shot `InitialBackfillWorker` (constrained the same as the nightly scan) that walks MediaStore once and enqueues everything new
- Start the foreground service and register observers per Phase 2

**Pre-Phase-4 users**
- Users who already enabled auto-upload under Phase 2/3 are not re-prompted. `first_backup_scope` is backfilled to `today` on first launch after the Phase 4 update, and `videos_enabled` defaults to `false`. (Pre-distribution: there are no real users yet — see [`feedback_schema_migrations`](../memory) — so this is a safety default, not a true migration.)

### 2. Upload Mode Settings (P1)

Per Q&A Q6, upload timing becomes user-configurable. The default remains the Phase 2 behavior (immediate).

**Setting:** `upload_mode` — `immediate` / `scheduled` / `hybrid`. Stored in `Encrypted SharedPreferences`. Default `immediate`.

**Mode definitions:**

| Mode | ContentObserver fires | Nightly scan | Effect |
|------|----------------------|--------------|--------|
| `immediate` (default) | Enqueue + start upload worker now | Catches gaps | Phase 2 behavior |
| `scheduled` | Enqueue only (status `pending`) | Drains the queue during the 02:00 window | Photos backed up overnight; manual tap-to-upload still works |
| `hybrid` | If on **unmetered network**: upload now. Otherwise: enqueue + defer to nightly. | Drains anything deferred | Phone uploads immediately on Wi-Fi but never on mobile data, regardless of `wifi_only_uploads` |

**Interaction with `wifi_only_uploads` (Phase 2):**
- `wifi_only_uploads` is independent and still respected: it forces the upload worker to skip metered networks during any active upload attempt
- `hybrid` is a superset of "Wi-Fi-only for immediate uploads": with `hybrid`, mobile-data uploads are always deferred to the nightly window even if `wifi_only_uploads = false`
- If `upload_mode = scheduled` and `wifi_only_uploads = true`, the nightly worker waits for an unmetered window (existing WorkManager constraint)

**Settings UI:**
- Settings → "Backup timing"
- Radio group with three options and a one-line description under each
- Changing the mode takes effect on the next observer fire / worker tick; in-flight uploads are not interrupted

**Foreground notification copy:**
- `immediate`: unchanged from Phase 2
- `scheduled`: "Photo backup is active — uploads run nightly. 12 pending."
- `hybrid` on mobile data: "Waiting for Wi-Fi to upload — 12 pending."
- `hybrid` on Wi-Fi: same as `immediate`

### 3. Video Handling (P1)

Per Q4, videos are first-class media. Phase 4 adds them.

**Settings:**

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `videos_enabled` | boolean | `false` | Set during Step C of first-backup flow |
| `video_quality_mode` | enum: `original` / `compressed` / `duration_based` | `duration_based` | What to upload |
| `video_duration_threshold_minutes` | int | `2` | For `duration_based`: videos ≤ threshold upload at full size, longer videos are compressed |
| `video_target_resolution` | enum: `720p` / `1080p` | `720p` | Compression target |

**MediaStore observation:**
- A second `ContentObserver` is registered on `MediaStore.Video.Media.EXTERNAL_CONTENT_URI`
- The Phase 2 detection / enqueue / dedup pipeline is reused; videos go into the same `upload_queue` table with a new `media_type` column (`photo` / `video`)
- If `videos_enabled = false`, the video observer is unregistered and the nightly scan skips `MediaStore.Video`

**B2 path layout (extends Q3):**
- Original videos: `videos/YYYY/MM/DD/filename.mp4`
- Compressed videos: `videos/YYYY/MM/DD/filename.compressed.mp4`
- Thumbnails: `thumbnails/YYYY/MM/DD/filename.webp` (first frame at the same path layout as photos)

**Compression:**
- AndroidX Media3 `Transformer` is the chosen API (modern, MediaCodec-backed, no native binaries)
- For `compressed` and `duration_based` (when over threshold), transcode to H.264 + AAC at the configured target resolution, preserving aspect ratio
- Transcoding runs inside the upload worker on `Dispatchers.IO`, writing the transcoded file to `cacheDir` before upload
- On transcode failure: fall back to uploading the original (log and increment a Phase 4 metric `video_transcode_fallback_count`)
- The transcoded file is deleted from `cacheDir` after a successful upload

**Thumbnail generation:**
- For videos, the existing thumbnail generator is extended to extract the first frame via `MediaMetadataRetriever.getFrameAtTime(0)` and encode as WebP at the same dimensions used for photo thumbnails
- Thumbnail goes to the existing `thumbnails/` path layout

**Gallery:**
- Video tiles in any view mode (Local / Cloud / Merged) get a small play-icon overlay on the thumbnail
- Tapping a cloud-only video opens a stub player screen (or `Intent.ACTION_VIEW` against the signed B2 URL) — full in-app playback is out of scope; we open the system video player

**Edge cases:**
- HEIC / HEVC source: handled by `Transformer` natively
- Video too large to fit in `cacheDir`: skip transcode, upload original; emit a notification "1 video too large to compress, uploaded at full size"
- Video duration unknown (corrupt metadata): treat as over-threshold and compress

### 4. Local Deletion Strategy (P2)

Per Q4 and the roadmap: auto-reclaim local space after successful backup, configurable.

**Setting:** `local_delete_strategy` — `never` / `immediate` / `after_days` / `after_count`. Default `never`. Stored in `Encrypted SharedPreferences`.

**Companion settings:**
- `local_delete_days` (int, default `30`, used when strategy = `after_days`)
- `local_delete_count` (int, default `100`, used when strategy = `after_count` — "delete the oldest backed-up media after this many *new* media items have arrived since the previous cleanup")

**Behavior:**
- `never` (default): no automatic local deletion; only context-aware delete from Phase 3 runs
- `immediate`: the moment an upload completes successfully (and the row is `uploaded`), mark the row `pending_local_delete = true`; a daily worker collects all pending rows and presents one batched `MediaStore.createDeleteRequest` to the user. (Android 11+ scoped storage requires user confirmation per request — we batch to avoid prompting the user constantly.)
- `after_days`: same mechanism, but the daily worker only includes rows where `uploaded_at <= now - local_delete_days * 86400`
- `after_count`: daily worker counts new media items added to `uploads` since the last cleanup pass; if ≥ `local_delete_count`, queue the oldest uploaded rows up to that count for batch deletion

**Daily prompt UX:**
- One Android notification per day at 19:00 local time when there are pending deletions: "12 backed-up photos ready to free up space (350 MB). Tap to review."
- Tapping opens a review screen listing the items with thumbnails and a "Free up space" CTA that issues a single `createDeleteRequest`
- User can deselect individual items before confirming
- If the user dismisses the notification three days in a row, suppress the prompt for the next 7 days (avoids nagging)

**Successful deletion side effects:**
- Set `local_present = false` on the `uploads` row (Phase 3 column)
- Clear `pending_local_delete`
- The row remains visible in Cloud view and Merged view (as cloud-only)

**Edge cases:**
- Upload incomplete or `permanently_failed`: never queued for local delete
- Photo already manually deleted (file missing from MediaStore at prompt time): silently drop from the pending batch
- User cancels the system delete dialog: leave `pending_local_delete = true`, retry tomorrow

### 5. Cost Dashboard (P2)

A read-only screen surfacing what the backup is costing.

**Location:** Settings → "Storage & cost"

**Inputs (computed on screen open, cached for 5 minutes):**
- Total cloud-backed photo count = `uploads` rows where `status = 'uploaded'`
- Total cloud-backed video count = same, partitioned by `media_type = 'video'`
- Total cloud-backed bytes = `SUM(size)` over the above rows
- Local-only count = `uploads` rows with `local_present = true AND status != 'uploaded'`
- Estimated monthly storage cost = `total_GB * $0.006` (B2 pricing as of 2026-05; surface a footnote linking to [`research/backblaze-b2-cost-rationale.md`](../research/backblaze-b2-cost-rationale.md) for the source)

**Displayed sections:**
1. **At a glance**
   - Total backed up: "9,842 photos, 314 videos · 18.2 GB"
   - Estimated monthly cost: "≈ $0.11 / month"
2. **By media type**
   - Photos: count, total size, monthly cost
   - Videos: count, total size, monthly cost
3. **By year** (P3 — optional within Phase 4)
   - Bar chart of bytes uploaded per year, top 5 years
4. **Pending**
   - Items waiting to upload: count + size
   - Failed (permanent): count
5. **Egress notes**
   - "Thumbnail viewing has used ≈ X MB this month" (only shown if thumbnail cache miss telemetry has at least one data point — see §6 below; otherwise omit the section)
   - One-line caveat: "B2 pricing may change. These numbers are estimates."

**No write actions on this screen.** It is purely informational. Pricing constants live in a single Kotlin object `B2Pricing` so a future B2 price change is one diff.

### 6. Sharing (P3)

Generate a time-limited public URL for a single photo or video.

**Entry point:** Long-press a tile in any view mode → "Share link" appears in the action menu (alongside the existing Phase 3 "Delete"). Only available when the item has `status = 'uploaded'` (cloud-backed); local-only items show the standard Android share sheet pointing at the local file (unchanged).

**Share-link dialog:**
- Title: "Share link"
- Body: "Anyone with this link can view the photo until it expires."
- TTL chooser (radio): "1 hour" (default), "24 hours", "7 days"
- Button: "Create link" → generates a B2 S3 presigned GET URL for `photo_b2_path` with the chosen TTL
- On creation: copy URL to clipboard, show a toast "Link copied. Expires in 24 hours.", and open the system share sheet with the URL pre-filled

**Active links screen:** Settings → "Active share links"
- Lists every share link issued by this app instance: thumbnail, filename, created at, expires at, "Copy" / "Open" actions
- Expired links auto-hide after 24 hours past expiry
- A "revoke" action is intentionally omitted — B2 presigned URLs cannot be revoked; the help text on the screen reads: "Links cannot be revoked once shared. They expire automatically. Use short durations for sensitive photos."

**Storage:**
- New `share_links` table (see §Technical Requirements)
- Phase 4 does not delete expired rows automatically; the active-links screen filters them out. A future cleanup pass can hard-delete rows older than 30 days past expiry.

**Constraints:**
- Only one TTL choice per link; cannot extend an existing link (issue a new one)
- Videos use the same flow; the URL points at the original video object, not the thumbnail
- No watermarking, no access logging on the B2 side (would require a server)

### 7. Settings Screen (Polish)

A single `SettingsActivity` consolidates every toggle introduced in Phases 2–4. Replaces the minimal affordances scattered across the Phase 2 Wi-Fi toggle and the Phase 3 "Back up index now" button.

**Sections, in order:**

1. **Backup**
   - Auto-upload (toggle, default on after first-backup flow)
   - Backup timing (`immediate` / `scheduled` / `hybrid`) — §2
   - Wi-Fi only (existing Phase 2 toggle)
2. **Videos**
   - Include videos (toggle) — §3
   - Quality mode (radio)
   - Duration threshold (slider, only shown when mode = `duration_based`)
   - Target resolution (radio: 720p / 1080p)
3. **Storage management**
   - Auto-delete strategy (radio) — §4
   - Days / count fields (conditional)
4. **Storage & cost** — §5 (entry point to a sub-screen)
5. **Backup index** — Phase 3 controls (last sync, "Back up index now", "Restore index from cloud")
6. **Active share links** — §6 (entry point to a sub-screen)
7. **Account**
   - "Re-enter credentials" (rerun the onboarding credential step without wiping data)
   - "Sign out" (clears credentials, prompts for confirmation, keeps the SQLite index intact)
8. **About**
   - Version, B2 endpoint, link to project README

**UX details:**
- Each section header is bold; rows are standard Android preference rows
- Implementation: Compose `LazyColumn` with manual rows (the project does not use AndroidX Preference library)
- All settings persist in `Encrypted SharedPreferences` via Phase 3's `PrefsStore`, extended in Phase 4

---

## Technical Requirements

### Database Migration (v3 → v4)

The Phase 3 schema is dropped and recreated (pre-distribution policy — see [`feedback_schema_migrations`](../memory)). Phase 4 additions:

`uploads` table additions:
- `media_type` TEXT NOT NULL DEFAULT `'photo'` — `'photo'` or `'video'`
- `original_path_b2` TEXT NULL — for compressed videos, the path of the original (if also uploaded)
- `pending_local_delete` BOOLEAN NOT NULL DEFAULT `0`
- `compressed` BOOLEAN NOT NULL DEFAULT `0`

New `share_links` table:

| Column | Type | Notes |
|--------|------|-------|
| id | INTEGER PRIMARY KEY | |
| upload_id | INTEGER | FK to `uploads.id` |
| photo_b2_path | TEXT | denormalized for orphan resilience |
| url | TEXT | full presigned URL |
| created_at | INTEGER | unix millis |
| expires_at | INTEGER | unix millis |

Index on `expires_at` for the active-links screen filter.

`PrefsStore` additions:
- `first_backup_scope` (`today` / `all`)
- `videos_enabled`, `video_quality_mode`, `video_duration_threshold_minutes`, `video_target_resolution`
- `upload_mode`
- `local_delete_strategy`, `local_delete_days`, `local_delete_count`, `local_delete_suppress_until` (timestamp for the 3-strike dismiss rule)
- `cost_dashboard_egress_bytes_month` (rolling counter, reset on the 1st of each month)

### Upload Pipeline Changes

- The Phase 2 `UploadWorker` gains an `UploadModeGate` decision at the top of each tick: given `upload_mode`, current network type, and pending queue, decide whether to run now or defer
- Videos travel the same path as photos but pass through a `MediaTranscoder` step before the B2 PUT when compression is required
- New `InitialBackfillWorker` — one-shot `WorkRequest` scheduled when first-backup scope = "Entire gallery history"; walks `MediaStore.Images` (and `MediaStore.Video` if enabled) once, enqueues everything not already in `uploads`, then deletes itself

### Daily Local-Delete Worker

- `PeriodicWorkRequest` at 19:00 local time
- No network constraint (operates only on local DB + MediaStore)
- Collects `pending_local_delete = true` rows, surfaces a single notification, runs only if the count > 0
- Uses Android 11+ `MediaStore.createDeleteRequest`; lower SDKs are not supported (project min SDK is 33)

### Sharing — Presigned URLs

- Implemented via the existing S3-compatible client used for uploads
- AWS SDK / S3 V4 presigner with the configured B2 endpoint
- TTL is enforced by B2 — the app does no expiry tracking beyond UI display
- Generated URLs are never logged

### Video Transcoding

- New module `media/Transcoder.kt` wrapping AndroidX Media3 `Transformer`
- Output is always MP4 H.264 + AAC at the configured target resolution
- Transcoded artifacts live under `cacheDir/transcoded/` and are deleted on upload completion
- Transcode is interruptible: if cancelled or killed mid-transcode, the next worker run starts over from the source (no resume mid-transcode)

### Cost Calculations

- `B2Pricing` Kotlin object with constants and a `lastUpdated` field
- All values displayed are estimates; no live B2 API call to fetch pricing
- The cost screen issues one aggregate SQL query and caches the result for 5 minutes

### Settings Screen

- New `SettingsActivity` registered in `AndroidManifest.xml`
- Compose-based (project precedent: existing screens are Compose)
- All preference reads go through `PrefsStore`; no direct `SharedPreferences` access from UI code

### Permissions

- No new manifest permissions for §1–5 (`POST_NOTIFICATIONS` from Phase 2 already covers the auto-delete prompt)
- Video transcoding requires no additional permission (`READ_MEDIA_VIDEO` is added; project currently declares only `READ_MEDIA_IMAGES`)
- `READ_MEDIA_VIDEO` is requested only when the user toggles `videos_enabled = true`

---

## Success Criteria

- [ ] On a fresh install, after credential entry, the user is asked whether to back up only new photos or the entire gallery history, and the chosen scope is honored
- [ ] Pre-Phase-4 users are not re-prompted; their stored state migrates safely (`first_backup_scope = today`, `videos_enabled = false`)
- [ ] Upload mode setting takes effect for new uploads:
  - `immediate` matches Phase 2 behavior
  - `scheduled` defers everything to the 02:00 nightly window
  - `hybrid` uploads immediately on Wi-Fi and defers on mobile data
- [ ] Toggling `videos_enabled` on starts capturing videos via a new `MediaStore.Video` observer; toggling off stops it without breaking the photo pipeline
- [ ] Videos under the duration threshold upload at original quality; longer videos are transcoded to the configured target resolution
- [ ] Disabling video uploads stops the observer and nightly scan from enqueuing new videos; existing video uploads in the queue continue
- [ ] Auto-delete strategies produce the expected behavior:
  - `never`: no local deletions occur automatically
  - `immediate`: the next-day prompt offers to delete everything uploaded since last cleanup
  - `after_days`: only rows with `uploaded_at <= now - N days` appear in the prompt
  - `after_count`: prompt only appears when at least N new uploads have occurred since the last cleanup
- [ ] Cost dashboard shows photo / video counts, total stored size, and an estimated monthly cost grounded in `B2Pricing`
- [ ] Share-link dialog produces a working presigned URL with the chosen TTL; the link plays in any standard browser until expiry
- [ ] Active share links screen lists current links and hides expired ones after 24 hours
- [ ] Settings screen surfaces every Phase 2–4 toggle in one place and persists changes through `PrefsStore`
- [ ] All Phase 2 and Phase 3 success criteria continue to pass

---

## Out of Scope (Post-Phase 4)

- Albums and album-level share collections
- Bandwidth throttling, upload pause schedules
- Multi-device or shared-bucket sync, conflict resolution
- Multiple B2 buckets per app instance
- In-app full-resolution photo viewer and video player (we keep delegating to system viewers)
- Trash / restore beyond Phase 3's 24h soft-delete window
- Client-side encryption
- RAW, HEIC originals at full fidelity beyond what MediaStore exposes
- Search, EXIF browsing, face detection, tagging
- Server-backed share pages, access analytics, watermarking
- Cross-provider migration (B2 ↔ S3 / Wasabi / Storj)
- Differential SQLite index sync (still a full-file upload from Phase 3)

---

## Open Questions

1. For `hybrid` mode, should "unmetered" alone be the trigger, or should we also require the device to be charging? (Battery concern at scale.)
2. For `after_count` deletion: should the count window be "since the last cleanup ran" (current proposal) or a rolling 30-day count? The current proposal can leave very long gaps if uploads slow down.
3. Cost dashboard: should we include cumulative egress estimation (instrumented via the thumbnail cache miss counter), or is that too noisy to bother surfacing? The current proposal hides it when the counter is empty, which means most installs never see it.
4. Share-link dialog: should we surface the full URL with a copy button in addition to (or instead of) the system share sheet, in case the user wants to paste it into a context the share sheet does not reach?
5. Should the "Active share links" screen track *which* contact / app the link was shared to? (Would require intercepting the share intent — niche utility, privacy cost.)
6. Local-delete notification at 19:00 — is that the right time, and should it be user-configurable in Phase 4 or deferred?
7. Video transcoding: confirm AndroidX Media3 `Transformer` is acceptable as a new dependency (it pulls in ExoPlayer-class artifacts and inflates the APK by a few MB). Fallback would be raw `MediaCodec`.
8. Should `first_backup_scope = all` enqueue videos too when `videos_enabled` is toggled on later, or only enqueue them from that point forward? Current proposal: only forward, because a full historical video upload is a much bigger commitment.
