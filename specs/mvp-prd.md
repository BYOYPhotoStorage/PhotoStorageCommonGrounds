# MVP PRD — Photo Backup App

## Overview

A minimal Android app that reads photos from the device gallery and uploads them to Backblaze B2 on user tap. The goal of this MVP is to validate the core pipeline — B2 connectivity, photo reading, upload, and local tracking — before adding background automation.

---

## Goals

1. Prove the app can read photos from the Android gallery
2. Prove the app can authenticate and upload to Backblaze B2
3. Prove the app can generate and upload thumbnails
4. Prove the app can track upload state locally with SQLite
5. Provide a foundation to build background auto-upload on top of

---

## Non-Goals

- Background auto-upload
- Automatic gallery scanning
- Video support
- Cloud browsing / merged gallery
- Settings screen
- Nightly SQLite backup to B2
- Deletion from cloud
- Notifications
- Duplicate detection (beyond basic filename check)
- Encryption

---

## User Flow

### 1. Onboarding

Screen: Simple form with 3 fields
- B2 Application Key ID (text input)
- B2 Application Key (text input, masked)
- B2 Bucket Name (text input)
- "Connect" button

On tap:
- Validate credentials by listing the bucket contents via B2 API
- If valid, save credentials to Encrypted SharedPreferences
- If invalid, show error toast

### 2. Gallery Screen (Main Screen)

- Displays a grid of **local photos only** (pulled from MediaStore)
- Each photo cell shows:
  - Thumbnail
  - Upload status indicator (not uploaded / uploading / uploaded)
- Tap a photo to trigger upload
- Long-press a photo to see raw file info (filename, size, date)

### 3. Upload Flow

On user tap:
1. Generate a WebP thumbnail from the photo
2. Upload the original to `photos/YYYY/MM/DD/filename.jpg`
3. Upload the thumbnail to `thumbnails/YYYY/MM/DD/filename.webp`
4. Insert a record into local SQLite with:
   - `local_uri`
   - `filename`
   - `size`
   - `uploaded_at`
   - `photo_b2_path`
   - `thumbnail_b2_path`
   - `status`
5. Update the gallery cell to show "uploaded" state

### 4. App Restart Behavior

- On launch, read SQLite to know which local photos are already uploaded
- Re-query MediaStore to refresh the local gallery grid
- Match local photos to SQLite records by filename + size
- Show correct upload status for each photo

---

## Technical Requirements

### B2 Integration
- Use Backblaze B2 Native API (not S3-compatible API)
- Implement: authorize, upload file, list file names
- Handle token expiry and re-authorization

### Photo Access
- Use Android MediaStore API to query images
- Target API 33+ with `READ_MEDIA_IMAGES` permission
- Request permission on first launch if not granted

### Thumbnails
- Generate 200x200 WebP thumbnail using Android's `ThumbnailUtils` or `BitmapFactory`
- Compress to ~20-40 KB

### Local Database (SQLite)
Single table: `uploads`

| Column | Type | Notes |
|--------|------|-------|
| id | INTEGER PRIMARY KEY | Auto-increment |
| local_uri | TEXT | MediaStore content URI |
| filename | TEXT | Original filename |
| size | INTEGER | File size in bytes |
| date_taken | INTEGER | Unix timestamp |
| photo_b2_path | TEXT | Path in B2 bucket |
| thumbnail_b2_path | TEXT | Path in B2 bucket |
| status | TEXT | `pending` / `uploading` / `uploaded` / `failed` |
| uploaded_at | INTEGER | Unix timestamp, nullable |

### Error Handling
- Network failure: mark status as `failed`, allow retry on next tap
- B2 auth failure: show toast, do not retry automatically
- Disk full: show toast

---

## Success Criteria

- [ ] User can enter B2 credentials and connect successfully
- [ ] App displays local gallery photos in a grid
- [ ] Tapping a photo uploads it to B2 within 10 seconds (on Wi-Fi, 5MB photo)
- [ ] Thumbnail is generated and uploaded alongside the original
- [ ] Upload status persists across app restarts
- [ ] Upload status is visually indicated in the gallery grid
- [ ] App handles B2 token expiry gracefully during upload

---

## Out of Scope (Post-MVP)

- Background auto-upload service
- ContentObserver for real-time detection
- WorkManager for scheduled tasks
- Video support
- Multiple view modes (cloud, merged)
- Settings / configuration screen
- Deletion from B2
- Notifications
- Duplicate detection via SHA-256
- Nightly SQLite backup to B2
- Reinstall recovery flow
- Upload queue / batching
- Wi-Fi-only checks
- Compression options

---

## Open Questions

1. Should the MVP support B2 Application Key with bucket restrictions, or require a master key?
2. Should failed uploads show a retry button, or require re-tapping the photo?
3. What happens if the user taps an already-uploaded photo — re-upload or ignore?
