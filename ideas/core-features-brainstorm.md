# Core Features Brainstorm

## Problem Statement
Google Cloud Storage is expensive for personal photo backups. Backblaze B2 is significantly cheaper. We want an Android app that uploads photos directly from the phone gallery to Backblaze B2 — no middleman, no expensive storage layer.

---

## MVP Features

### 1. Backblaze B2 Onboarding
- First launch: prompt user to enter B2 Application Key ID and Application Key
- Optional: guide/wizard to help create a B2 bucket and key if they don't have one
- Validate credentials before saving
- Allow multiple bucket configurations (e.g., one for photos, one for videos)

### 2. Auto-Upload from Gallery
- Background service that monitors the device's photo gallery
- Automatically upload new photos/videos as they are taken
- Respects metered/unmetered networks (Wi-Fi only option)
- Handle duplicates: don't re-upload the same file

### 3. Upload Queue & Resilience
- Local queue for pending uploads
- Retry with exponential backoff on failure
- Resume interrupted uploads (chunked uploads)
- Handle app kills, network drops, device reboots

### 4. Basic Browse & Status
- Simple list/grid view of uploaded photos
- Upload status indicators: pending, uploading, uploaded, failed
- Option to retry failed uploads manually

### 5. Encryption
	- Client-side encryption before upload (the user holds the keys)
- B2 already encrypts at rest, but client-side encryption means even B2 can't read the files
- Option to set a passphrase / encryption key during onboarding

---

## Post-MVP Features

### Organization & Metadata
- Folder/album structure in B2 based on date taken or custom tags
- EXIF metadata preservation and indexing
- Face detection / tagging (on-device, privacy-preserving)
- Search by date, location, or tags

### Sync & Multi-Device
- Cross-device sync: same B2 bucket, multiple phones
- Conflict resolution if two devices upload the same photo
- Desktop companion app (Windows/Mac/Linux) for the same bucket

### Advanced Upload Controls
- Selective sync: choose which albums to back up
- Upload quality settings: original vs. compressed
- Video handling: separate toggle, max file size limits
- Scheduled uploads: nightly batch instead of immediate

### Cost & Bandwidth Controls
- Upload bandwidth limiting
- Monthly upload quota warnings
- Estimated cost dashboard (B2 pricing calculator)
- Storage usage breakdown

### Sharing
- Time-limited share links via B2's native capabilities
- Album sharing with family/friends
- Public gallery option

### Safety & Recovery
- Trash / soft-delete: keep deleted photos for 30 days
- Versioning support (B2 native versioning)
- Export/download entire backup for local recovery
- Checksum verification to detect bit-rot or corruption

---

## Open Questions

- Should the app support other S3-compatible providers (Wasabi, Storj, self-hosted MinIO)?
- How do we handle B2's free egress limits for viewing photos in-app?
- Do we want a server-side component at all (e.g., for indexing, sharing links), or stay 100% client-to-B2?
- What happens if a user loses their encryption key?
- Should we support RAW/DNG files, or just JPEG/HEIC/PNG?
