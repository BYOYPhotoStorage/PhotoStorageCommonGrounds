# Task 25 — Deletion Engine

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Implement the `DeletionEngine` that performs mode-aware deletes for single items and batches. The engine encapsulates the rules in PRD §2 (Context-Aware Deletion): Local removes only the local file, Cloud soft-deletes the cloud row and B2 objects, Merged does both with partial-failure handling.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/gallery/DeletionEngine.kt`

## Contract

```kotlin
class DeletionEngine(
    private val context: Context,
    private val uploadDao: UploadDao,
    private val s3Uploader: S3Uploader,
    private val galleryRepository: GalleryRepository
) {
    suspend fun delete(item: GalleryItem, mode: GalleryViewMode): DeletionResult
    suspend fun deleteBatch(items: List<GalleryItem>, mode: GalleryViewMode): BatchDeletionResult
}

sealed class DeletionResult {
    object Success : DeletionResult()
    data class Refused(val reason: String) : DeletionResult()
    data class Failed(val cause: Throwable, val partialSuccess: Boolean) : DeletionResult()
}

data class BatchDeletionResult(
    val succeeded: Int,
    val refused: Int,
    val failed: Int,
    val summary: String
)
```

## Behavior by mode

### Local mode

For `LocalOnly` or `Synced`:
1. If the item has a `queuedRecord` with status `uploading`, return `Refused("Cancel the upload first.")`
2. Delete from MediaStore:
   - Android 11+ (R+): use `MediaStore.createDeleteRequest(...)` and surface the `IntentSender` via a callback the activity wires to its `ActivityResultLauncher`
   - Android 10 (Q): use `ContentResolver.delete(uri, null, null)` directly
3. If a matching `UploadRecord` exists:
   - If `status = 'uploaded'`: set `local_present = false` (keeps the cloud row visible in Cloud view)
   - Else (in queue, never uploaded): `hardDelete(id)`
4. Call `galleryRepository.invalidate()`
5. Return `Success`

For `CloudOnly`: return `Refused("Photo is not on this device.")`

### Cloud mode

For `CloudOnly` or `Synced`:
1. If `status = 'uploading'`: return `Refused("Wait for upload to complete.")`
2. If `status = 'permanently_failed'`: return `Refused("Discard the failed upload first.")`
3. Issue two B2 deletes in parallel using `coroutineScope { async { } }` for `photoB2Path` and `thumbnailB2Path`
4. If both succeed (404 counts as success per Task 22): `uploadDao.softDeleteCloud(id, now)`
5. If either fails: do NOT mutate the SQLite row. Return `Failed(cause, partialSuccess = false)`. (The DELETE Object API is idempotent — the caller can retry.)
6. Call `galleryRepository.invalidate()`
7. Return `Success`

For `LocalOnly`: return `Refused("Photo is not in the cloud.")`

### Merged mode

For `Synced`:
1. Check both upload-state guards from above
2. Run the cloud delete (steps 3–4 of Cloud mode)
3. Run the local delete (step 2 of Local mode)
4. If both succeed: `hardDelete(id)` (the row no longer represents any reachable copy)
5. If only cloud succeeded: set `local_present = false` (row already has `status = 'cloud_deleted'`)
6. If only local succeeded: `setLocalPresent(id, false)` (cloud copy still exists, row keeps `status = 'uploaded'`)
7. If neither succeeded: return `Failed(combinedCause, partialSuccess = false)`
8. `galleryRepository.invalidate()`
9. Return `Success` if both succeeded, `Failed(.., partialSuccess = true)` if exactly one did

For `CloudOnly`: same as Cloud mode.
For `LocalOnly`: same as Local mode.

## `deleteBatch` semantics

```kotlin
suspend fun deleteBatch(items: List<GalleryItem>, mode: GalleryViewMode): BatchDeletionResult {
    var succeeded = 0
    var refused = 0
    var failed = 0
    val refusalReasons = mutableSetOf<String>()
    val failureMessages = mutableSetOf<String>()

    for (item in items) {
        when (val r = delete(item, mode)) {
            is DeletionResult.Success -> succeeded++
            is DeletionResult.Refused -> { refused++; refusalReasons += r.reason }
            is DeletionResult.Failed -> { failed++; failureMessages += (r.cause.message ?: "Unknown error") }
        }
    }

    galleryRepository.invalidate() // single trailing invalidate

    val parts = mutableListOf<String>()
    if (succeeded > 0) parts += "$succeeded deleted"
    if (refused > 0) parts += "$refused skipped (${refusalReasons.joinToString(", ")})"
    if (failed > 0) parts += "$failed failed"
    return BatchDeletionResult(succeeded, refused, failed, parts.joinToString(" · "))
}
```

Process items sequentially — concurrent batch deletes can spike memory and confuse B2 rate limiting. At 10K-photo scale, sequential is fast enough.

## MediaStore delete on Android 11+

`createDeleteRequest` returns an `IntentSender` that must be confirmed by the user. The engine cannot complete the delete synchronously — it needs an `ActivityResultLauncher` from the activity.

Expose a hook:

```kotlin
fun interface MediaStoreDeleteLauncher {
    suspend fun launch(uris: List<Uri>): Boolean // returns true if user approved
}

class DeletionEngine(
    ...,
    private val mediaStoreDeleteLauncher: MediaStoreDeleteLauncher
)
```

The activity (Task 24) supplies a launcher that wraps `ActivityResultLauncher<IntentSenderRequest>` and a `CompletableDeferred<Boolean>` to bridge the callback to a suspending call.

For batch local-mode deletes, collect URIs first and call `launch(uris)` once — `createDeleteRequest` accepts a list, which gives the user a single system prompt.

## Implementation notes

- All work on `Dispatchers.IO` (the engine functions are `suspend` and assume the IO dispatcher; do not switch context internally).
- `softDeleteCloud(id, now)` uses `System.currentTimeMillis()`.
- B2 deletes for thumbnails do not block on the photo result — issue both with `async` and `awaitAll`.
- A `Refused` result never mutates state. A `Failed` result may leave partial state (only when explicitly noted as `partialSuccess = true`).
- The `summary` string is what the UI displays in a toast.

## Constraints

- Idempotent: re-running `delete()` on the same item after success must be safe (returns `Refused` because the row is gone or the file is missing).
- Never auto-confirm Merged-view deletes. The destructive confirmation is the UI's responsibility (Task 24); the engine just executes.
- No retries inside the engine. If a B2 DELETE returns 5xx, return `Failed` — the user can re-issue from the UI.
- Selection that crosses modes shouldn't happen by construction (the active mode determines the dialog and the engine call), but the engine still validates each item's variant and rejects mismatches with `Refused`.

## Dependencies (by interface)

- `UploadDao` (Task 20) — `softDeleteCloud`, `setLocalPresent`, `hardDelete`
- `S3Uploader` (Task 22) — `deleteObject`
- `GalleryRepository` (Task 23) — `invalidate`
- MediaStore via `ContentResolver` + `createDeleteRequest` on Android 11+

## Acceptance criteria

- [ ] Local-mode delete on a `Synced` item removes the local file and sets `local_present = false`.
- [ ] Cloud-mode delete on a `Synced` item issues both B2 DELETEs and soft-deletes the row.
- [ ] Cloud-mode delete failure (simulate 5xx) leaves the SQLite row unchanged.
- [ ] Merged-mode delete on a `Synced` item, with both operations succeeding, ends with the row hard-deleted from `uploads` and the file removed from MediaStore.
- [ ] Merged-mode partial failure (cloud OK, local denied by user) sets `local_present = false` but leaves `cloud_deleted_at` set — return `Failed(.., partialSuccess = true)`.
- [ ] Deleting an `uploading` photo via Cloud/Merged returns `Refused` and does not mutate state.
- [ ] `deleteBatch` returns counts that sum to the input size and a non-empty `summary`.
- [ ] Code compiles with no new dependencies beyond Phase 2 + Tasks 20–23.

## Out of scope

- Undo affordance for cloud deletes within the 24h soft-delete window (the soft-delete window exists for recovery in the SQLite layer, but Phase 3 has no "trash" UI — that's deliberately deferred).
- Background retry of failed B2 deletes — the user retries manually.
- Multi-bucket support.
