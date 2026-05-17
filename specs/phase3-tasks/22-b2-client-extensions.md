# Task 22 — B2 Client DELETE + GET Extensions

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Extend the existing S3-compatible B2 client with three operations Phase 3 needs: `DELETE Object` (for cloud deletes), `HEAD Object` (for the recovery flow's existence check), and `GET Object → file` (for downloading the SQLite index and B2-stored thumbnails). The PUT path is unchanged.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/upload/S3Uploader.kt` (modify — add three methods)

## Updated contract

Keep all existing MVP/Phase 2 upload methods unchanged. Add:

```kotlin
class S3Uploader(/* existing constructor */) {
    // ...existing upload methods...

    /**
     * Deletes a single object from the bucket. 404 is treated as success
     * (idempotent semantics — already gone counts as deleted).
     */
    suspend fun deleteObject(path: String): Result<Unit>

    /**
     * Checks whether an object exists. Returns Result.success(true) if it does,
     * Result.success(false) on 404, and Result.failure on any other error.
     */
    suspend fun headObject(path: String): Result<Boolean>

    /**
     * Streams the object to a local File. Caller owns the dest path.
     * On failure the dest file is deleted before returning.
     */
    suspend fun downloadObject(path: String, dest: File): Result<Unit>
}
```

### Method semantics

- `deleteObject`:
  - Issues `DeleteObjectRequest` against the configured bucket
  - 200/204 → `Result.success`
  - 404 → `Result.success` (treat already-deleted as success)
  - 4xx other than 404 → `Result.failure` with a wrapping exception that includes the status code
  - 5xx, network errors, timeouts → `Result.failure` (caller decides retry)
- `headObject`:
  - Issues `HeadObjectRequest`
  - 200 → `Result.success(true)`
  - 404 → `Result.success(false)`
  - Other errors → `Result.failure`
- `downloadObject`:
  - Issues `GetObjectRequest`
  - Streams response body to `dest` via `FileOutputStream` (buffered, 64 KB chunks)
  - On any failure, attempts `dest.delete()` before returning `Result.failure`
  - Does NOT create parent directories — caller is responsible

## Implementation notes

- Reuse the existing `S3Client` instance — do not create a second one.
- All three methods run on `Dispatchers.IO`.
- All three methods accept the *key/path* (e.g., `photos/2026/05/16/IMG_0001.jpg`), not a full URL. The client already knows the bucket.
- The `B2 checksum quirk` only matters for PUT — DELETE/HEAD/GET are not affected and require no special configuration.
- For `downloadObject`, prefer `responseBody.byteStream()` (or the equivalent SDK call) over loading the full body into memory; the SQLite index and thumbnails are small but the same path is used for both, and we want it to work for any future use.

## Constraints

- Idempotent DELETE: 404 = success. The caller (Task 25) relies on this.
- No retries inside these methods — the caller decides retry policy. (Phase 2's retry/backoff logic in `UploadWorker` is for PUT only; Phase 3 callers handle their own retries.)
- Do not log the bucket key as a credential — paths are fine to log at debug level, but do not include the signed URL.

## Dependencies (by interface)

- The existing S3-compatible client wired up in MVP. No new external dependencies.

## Acceptance criteria

- [ ] `deleteObject("nonexistent/path.jpg")` returns `Result.success` (404 path).
- [ ] `deleteObject(validPath)` succeeds and a subsequent `headObject(validPath)` returns `Result.success(false)`.
- [ ] `headObject` returns true/false for present/absent objects without throwing.
- [ ] `downloadObject` streams a known-existing file to disk and the resulting file's SHA-256 matches the expected value.
- [ ] On network failure, `downloadObject` cleans up the partially-written destination file.
- [ ] Code compiles with no new dependencies beyond what the MVP already uses.

## Out of scope

- Multipart uploads / range GETs.
- Object listing (`ListObjects`) — Phase 3 does not need it (the index file at a known path is the only listing-like operation, handled via HEAD).
- Bucket-level operations.
- Conditional requests (`If-Match`, `If-None-Match`) — Task 27 uses local hash comparison instead.
