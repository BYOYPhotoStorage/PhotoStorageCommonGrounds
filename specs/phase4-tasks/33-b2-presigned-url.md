# Task 33 — B2 Presigned URL Extension

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Extend the existing S3-compatible B2 client with a single new operation: `presignGetUrl`. The Sharing feature (Task 39) uses it to generate time-limited public URLs for individual photos and videos. The PUT / DELETE / HEAD / GET paths added in earlier phases are unchanged.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/upload/S3Uploader.kt` (modify — add one method)

## Updated contract

Keep all existing MVP / Phase 2 / Phase 3 methods unchanged. Add:

```kotlin
class S3Uploader(/* existing constructor */) {
    // ...existing methods...

    /**
     * Generates a presigned S3 GET URL for the given object path. The URL is
     * publicly fetchable (no auth required) until the TTL elapses. TTL is
     * enforced by B2; this client does no expiry tracking.
     *
     * Returns Result.success with the URL string on success.
     * Returns Result.failure on signing errors (e.g., missing credentials).
     */
    suspend fun presignGetUrl(path: String, ttlSeconds: Long): Result<String>
}
```

### Method semantics

- Accepts a *key/path* (e.g., `photos/2026/05/16/IMG_0001.jpg`), not a full URL — same convention as the rest of the client.
- TTL is in seconds. Acceptable range is `[60, 7 * 86_400]` (1 minute to 7 days). Values outside this range return `Result.failure(IllegalArgumentException("ttl out of range"))`.
- The returned URL is an HTTPS URL pointing at the configured B2 endpoint.
- Does **not** verify that the object exists. Sharing an URL to a missing object returns 404 to the recipient — that is acceptable behavior (the user will see the recipient cannot find it and can re-issue).
- Idempotent within the TTL: calling twice with the same path and TTL within a short window produces equivalent (usable) URLs; they will differ byte-for-byte due to the signature timestamp, but both work.

## Implementation notes

- Use the existing AWS SDK V4 presigner with the configured B2 endpoint, region, and credentials from `S3Client`'s configuration.
- Run on `Dispatchers.IO`. Signing is CPU-bound but fast; still keep it off the main thread for consistency.
- Do not log the returned URL. Path may be logged at debug level. Treat the URL like a credential.
- No network call is required for presigning — it is local cryptographic work. Therefore failure modes are limited to:
  - Missing or invalid B2 credentials (returns `Result.failure`)
  - TTL out of range (returns `Result.failure`)
- The B2 endpoint requires path-style addressing (set by MVP). The presigner inherits the existing endpoint configuration; do not override it.

```kotlin
suspend fun presignGetUrl(path: String, ttlSeconds: Long): Result<String> = withContext(Dispatchers.IO) {
    if (ttlSeconds !in 60..(7 * 86_400)) {
        return@withContext Result.failure(IllegalArgumentException("ttl out of range"))
    }
    runCatching {
        val presigner = S3Presigner.builder()
            .endpointOverride(endpoint)
            .region(region)
            .credentialsProvider(credentialsProvider)
            .serviceConfiguration(S3Configuration.builder().pathStyleAccessEnabled(true).build())
            .build()
        presigner.use { p ->
            val req = GetObjectRequest.builder().bucket(bucket).key(path).build()
            val presigned = p.presignGetObject {
                it.signatureDuration(Duration.ofSeconds(ttlSeconds))
                  .getObjectRequest(req)
            }
            presigned.url().toString()
        }
    }
}
```

> The exact SDK class names depend on the artifact version already wired up in MVP. If MVP uses a different S3 client family (e.g., MinIO SDK), use that family's presigner equivalent. The contract — input, output, TTL bounds — is fixed.

## Constraints

- No retries inside this method — it does no network I/O.
- No conditional behavior based on `wifi_only_uploads` — presigning is local.
- Do not extend the URL with additional query parameters (e.g., `?response-content-disposition=…`). Phase 4 issues vanilla GET URLs.
- The `B2 checksum quirk` does not apply (presigned GETs do not carry checksums).

## Dependencies (by interface)

- The existing S3-compatible client wired up in MVP / Phase 2 / Phase 3. No new external dependencies.

## Acceptance criteria

- [ ] `presignGetUrl("photos/2026/05/17/IMG_001.jpg", 3_600)` returns `Result.success` with a URL string.
- [ ] Fetching the returned URL with `curl` retrieves the object content for the next hour and fails after expiry.
- [ ] `presignGetUrl(path, 30)` returns `Result.failure(IllegalArgumentException)`.
- [ ] `presignGetUrl(path, 8 * 86_400)` returns `Result.failure(IllegalArgumentException)`.
- [ ] The returned URL is not logged at any log level.
- [ ] Code compiles with no new dependencies beyond what the MVP already uses.

## Out of scope

- Presigned PUT URLs (uploads continue through the existing authenticated client).
- POST policies, signed cookies, or alternative presigning schemes.
- Tracking issued URLs — that is Task 39's job (`ShareLinkDao`).
- Revoking issued URLs — B2 presigned URLs are non-revocable, surface this in the UI per PRD §6.
