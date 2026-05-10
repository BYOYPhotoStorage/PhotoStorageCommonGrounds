# Project Clarification Q&A

This document captures the clarifying questions and answers that shaped the architecture and scope of the photo backup app.

---

## Q1: What platforms should the app target?

**Answer:** Android only. iOS is not considered because it does not support background tasks reliably enough for automatic photo uploads.

---

## Q2: Should the app have a server component, or be pure client-side?

**Answer:** Pure client-side. No backend server at all. The app talks directly from the phone to Backblaze B2.

A local SQLite database is used for indexing and fast local searching. The SQLite file itself is periodically backed up to B2 so it can be restored on reinstall or new devices.

---

## Q3: How should photos be organized in B2, and should there be client-side encryption?

**Answer:**
- **Organization:** Date-based folder structure:
  - `photos/YYYY/MM/DD/filename.jpg`
  - `thumbnails/YYYY/MM/DD/filename.webp`
- **Encryption:** No client-side encryption. B2's at-rest encryption is sufficient for this use case.

---

## Q4: Should the app handle videos, and what happens to originals on the phone after upload?

**Answer:**
- **Videos:** Yes, videos are supported. Thumbnails are generated from the first frame. Video compression options are provided (similar to WhatsApp).
- **Settings for videos:**
  - Toggle to upload full-size or compressed
  - Duration-based rules (e.g., videos under 2 minutes upload at full size, longer videos use reduced size)
  - Option to disable video uploads entirely
- **Deletion strategy (configurable in settings):**
  - Never auto-delete
  - Delete immediately after successful upload
  - Delete after X days of uploading
  - Delete after N photos have been taken
  - Only delete when explicitly deleted from the app

---

## Q5: How should duplicate detection work?

**Answer:** Two-level filtering approach:
1. **Level 1:** Compare filename + file size + modified date. If all match, treat as duplicate.
2. **Level 2:** If Level 1 does not match, compute SHA-256 of the original file and compare against stored hashes in the SQLite index.

The SHA-256 hash is stored in the SQLite database so duplicate detection works even after app reinstall.

---

## Q6: What triggers uploads, and how does the SQLite database sync to B2?

**Answer:**
- **Upload trigger:** Configurable in app settings:
  - Immediate upload as photos are taken
  - Scheduled/batch upload (e.g., nightly)
  - Hybrid mode
  - User chooses their preferred mode
- **SQLite backup:**
  - Automatic nightly sync to B2 (only if new photos were added since last sync)
  - Manual on-demand sync option in settings
  - Users can manually trigger a sync before reinstalling or switching phones

---

## Q7: How should the app discover photos to upload, and how should reinstalls recover state?

**Answer:**
- **Discovery mechanism:**
  - Primary: Android MediaStore ContentObserver for real-time notifications when new photos are taken
  - Fallback: Nightly WorkManager job that performs a full gallery scan to catch anything missed (photos added while app was killed, restored from backup, etc.)
  - WorkManager constraints can include "only on Wi-Fi" and "only when charging"
- **Reinstall / new phone recovery:**
  1. Download the SQLite index from B2 on sign-in
  2. Compare the timestamp of the last uploaded photo with the local gallery
  3. Ask the user: "You are caught up until [date]. Upload only new photos from that point, or do a thorough full scan?"

---

## Q8: What scale are we designing for?

**Answer:**
- ~10,000 photos initially
- ~100 new photos per week
- A single SQLite database is sufficient for this scale (1–3 MB at 10K photos)
- The schema is designed so that partitioning by year remains possible in the future if needed

---

## Q9: What should the onboarding flow look like?

**Answer:**
- For MVP: Simple form-based onboarding. User provides B2 Application Key ID and Application Key.
- Future: Chat-like, agentic onboarding where the app asks questions one by one in a conversational interface, with an option to explain how to obtain B2 credentials.

---

## Q10: How should the app communicate status and errors to the user?

**Answer:** Notifications should be used wherever possible:
- Persistent notification while uploads are running (required by Android for foreground services)
- Completion summaries (e.g., "47 photos backed up")
- Error notifications for repeated failures
- Confirmation notifications for destructive actions (e.g., deleting from both local and cloud)

---

## Q11: What is the day-to-day experience of the app?

**Answer:** Option C — **Hybrid gallery-first UI**:
- Main screen is a photo gallery (similar to Google Photos)
- A small status bar at the top shows backup health (last backup time, pending count, etc.)
- Backup happens silently in the background

---

## Q12: What photos should the gallery display?

**Answer:** The gallery supports three view modes:
1. **Local Viewing:** Shows only photos currently on the phone
2. **Cloud Viewing:** Shows only photos backed up in B2
3. **Merged Viewing:** Default mode. Shows:
   - All remote (backed-up) photos up to the last sync timestamp
   - All local photos with timestamps newer than the last sync timestamp

This avoids complex merge conflicts while giving the user a continuous timeline.

---

## Q13: What happens when a user deletes a photo from the gallery?

**Answer:** Deletion behavior depends on the current view mode:
- **Local Viewing:** Delete locally only
- **Cloud Viewing:** Delete from cloud only
- **Merged Viewing:** Delete from both local and cloud, with a notification confirming: "This will be deleted from both your phone and cloud storage."

---

## Q14: Should the app support multiple B2 buckets?

**Answer:** One bucket per app instance. However, multiple family members could use the same B2 account with separate buckets for each person's app instance.

---

## Q15: After onboarding, should the app auto-start backup or wait for the user?

**Answer:** Wait for the user to manually start the first backup. The app should also ask the user for a starting point:
- **Upload from today only:** Only new photos taken from now on will be backed up
- **Upload all existing photos:** Back up the entire gallery history

---

## Q16: How should B2 credentials be stored on the device?

**Answer:** Encrypted SharedPreferences / DataStore — Android's standard approach. Credentials are encrypted at rest using the device's hardware keystore. Secure against casual access, though recoverable on a rooted device.

---

## Summary of Key Architectural Decisions

| Decision | Choice |
|----------|--------|
| Platform | Android only |
| Architecture | Pure client-side, no server |
| Cloud Provider | Backblaze B2 |
| Local Database | SQLite, nightly sync to B2 |
| Encryption | None (B2 at-rest only) |
| Media Types | Photos + Videos |
| Upload Trigger | User-configurable (immediate / scheduled / hybrid) |
| Discovery | MediaStore ContentObserver + nightly WorkManager scan |
| Duplicate Detection | Filename+size+date, then SHA-256 |
| Gallery Views | Local, Cloud, Merged |
| Deletion | Context-aware based on current view mode |
| Buckets | One per app instance |
| First Backup | Manual start with "from today" or "all" option |
| Credential Storage | Encrypted SharedPreferences / DataStore |
| Notifications | Used everywhere applicable |
| Scale Target | 10K photos, ~100/week growth |
