# Task 15 — Duplicate Detection

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Implement two-level duplicate detection so the same photo is never uploaded twice. Used by the foreground service (Task 13) and the nightly scan (Task 16) before enqueuing a photo.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/dedup/DuplicateDetector.kt`

## Public contract

```kotlin
package com.hriyaan.photostorage.dedup

import android.content.Context
import android.net.Uri
import com.hriyaan.photostorage.data.MediaStorePhoto
import com.hriyaan.photostorage.data.UploadDao

class DuplicateDetector(
    private val context: Context,
    private val uploadDao: UploadDao
) {
    suspend fun isDuplicate(photo: MediaStorePhoto): DuplicateResult
    suspend fun computeSha256(uri: Uri): String
}

sealed class DuplicateResult {
    object NotDuplicate : DuplicateResult()
    data class Duplicate(val reason: String) : DuplicateResult()
}
```

## Two-level detection

### Level 1 (fast): filename + size + dateTaken

```kotlin
val existing = uploadDao.findByFilenameAndSize(photo.filename, photo.size)
if (existing != null && existing.dateTaken == photo.dateTakenMs) {
    return DuplicateResult.Duplicate("filename_size_date")
}
```

This catches the common case where the same photo file exists in both MediaStore and the DB.

### Level 2 (hash): SHA-256

If Level 1 does not find a match:

```kotlin
val hash = computeSha256(photo.uri)
val existingByHash = uploadDao.findBySha256(hash)
if (existingByHash != null) {
    return DuplicateResult.Duplicate("sha256")
}
```

This catches cases where the same photo was renamed, resized by another app, or restored from backup with different metadata.

### `computeSha256(uri)`

1. Open `context.contentResolver.openInputStream(uri)`.
2. Pipe bytes through `MessageDigest.getInstance("SHA-256")` in a buffered loop (8 KB buffer).
3. Return the hex-encoded digest (lowercase, no separators).

Example output: `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`

## Lazy hash computation

The SHA-256 is computed only when:
- Level 1 returns `NotDuplicate`, AND
- The caller is about to enqueue the photo for upload.

Do NOT compute SHA-256 during gallery browsing or general listing — it is too expensive for large photo sets.

## Constraints

- Run `computeSha256` in `Dispatchers.IO`.
- Handle `IOException` / `SecurityException` gracefully: if the URI cannot be read, treat as `NotDuplicate` (the upload worker will handle the failure).
- Use a fixed-size buffer (8 KB) to avoid loading the entire file into memory.
- The hex encoder must be deterministic — same bytes always produce the same string.

## Dependencies (by interface)

- `UploadDao` (Task 10) — `findByFilenameAndSize`, `findBySha256`
- `MediaStorePhoto` (Task 04 / MVP) — the input data

## Acceptance criteria

- [ ] A photo with matching `filename + size + dateTaken` in the DB returns `Duplicate("filename_size_date")`.
- [ ] A photo with different filename but same SHA-256 as a DB record returns `Duplicate("sha256")`.
- [ ] A photo with no match in the DB returns `NotDuplicate`.
- [ ] `computeSha256` produces the same hash for the same file across multiple calls.
- [ ] `computeSha256` on a 5 MB file completes in under 1 second on a mid-range device.
- [ ] No `OutOfMemoryError` when hashing a 50 MB file (uses streaming, not `readBytes()`).

## Out of scope

- Caching computed hashes in a separate file.
- Partial / block-level hashing.
- Detecting visually similar photos (different compression, slight edits).
