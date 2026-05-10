# Task 06 — B2 / S3 Uploader

> Read [`../mvp-implementation-overview.md`](../mvp-implementation-overview.md) first. Also read the [B2 S3-API research doc](../../research/backblaze-b2-s3-api-research.md) and the [ADR](../../decisions/2026-05-09-use-s3-compatible-api-over-native-b2-api.md).

## Goal

Wrap the AWS SDK for Kotlin so the app can authenticate to Backblaze B2 (S3-compatible API) and upload photos / thumbnails.

## Files you own

- `app/src/main/java/com/photobackup/app/b2/S3Config.kt`
- `app/src/main/java/com/photobackup/app/b2/S3ClientFactory.kt`
- `app/src/main/java/com/photobackup/app/b2/S3Uploader.kt`
- `app/src/main/java/com/photobackup/app/b2/S3KeyBuilder.kt`

## Public contract

```kotlin
package com.photobackup.app.b2

import aws.sdk.kotlin.services.s3.S3Client
import aws.smithy.kotlin.runtime.content.ByteStream
import com.photobackup.app.data.B2Credentials

data class S3Config(
    val region: String,        // e.g. "us-west-004"
    val endpoint: String,      // e.g. "https://s3.us-west-004.backblazeb2.com"
    val bucketName: String
) {
    companion object {
        // Derives region from endpoint, builds full Config from credentials' bucket.
        fun forBucket(
            bucketName: String,
            region: String = DEFAULT_REGION
        ): S3Config = S3Config(
            region = region,
            endpoint = "https://s3.$region.backblazeb2.com",
            bucketName = bucketName
        )

        const val DEFAULT_REGION = "us-west-004"
    }
}

object S3ClientFactory {
    fun create(credentials: B2Credentials, config: S3Config): S3Client
}

class S3Uploader(
    private val client: S3Client,
    private val bucket: String
) {
    suspend fun validateCredentials(): Result<Unit>
    suspend fun upload(
        key: String,
        contentType: String,
        contentLength: Long,
        body: ByteStream
    ): Result<Unit>
    fun close()  // calls client.close()
}

object S3KeyBuilder {
    fun photoKey(filename: String, dateTakenMs: Long): String
    fun thumbnailKey(filename: String, dateTakenMs: Long): String
}
```

### Method semantics

- `S3ClientFactory.create` constructs an `S3Client` configured with:
  - `region = config.region`
  - `endpointUrl = Url.parse(config.endpoint)`
  - `credentialsProvider = StaticCredentialsProvider { accessKeyId = credentials.keyId; secretAccessKey = credentials.applicationKey }`
  - HTTP client engine: OkHttp (`aws.smithy.kotlin:http-client-engine-okhttp`).
- `validateCredentials` calls `client.listBuckets()` and returns `Result.success(Unit)` on 2xx, `Result.failure(e)` on any throwable. The call's actual return value is ignored — we only care that auth succeeds.
- `upload` issues `PutObjectRequest`. **Must** set `checksumAlgorithm = null` to bypass the B2/AWS SDK incompatibility documented in the ADR.
- `S3KeyBuilder.photoKey("IMG_123.jpg", date)` returns `"photos/2026/05/09/IMG_123.jpg"`. Date components from `dateTakenMs` in **UTC**, zero-padded.
- `S3KeyBuilder.thumbnailKey("IMG_123.jpg", date)` returns `"thumbnails/2026/05/09/IMG_123.webp"` — same date prefix, basename swapped to `.webp`.

### Reference snippet for `upload`

```kotlin
suspend fun upload(
    key: String,
    contentType: String,
    contentLength: Long,
    body: ByteStream
): Result<Unit> = runCatching {
    client.putObject(PutObjectRequest {
        bucket = this@S3Uploader.bucket
        this.key = key
        this.body = body
        this.contentType = contentType
        this.contentLength = contentLength
        this.checksumAlgorithm = null   // CRITICAL — see ADR
    })
}
```

## Constraints

- **Always** set `checksumAlgorithm = null`. If you forget, B2 returns `400 Bad Request` and the app silently fails uploads.
- Do not log credentials, signed URLs, or `Authorization` headers.
- Caller is responsible for closing `body` (an `InputStream` wrapped in `ByteStream`); the AWS SDK consumes but does not always close.
- `S3Client` is expensive to create — caller (Task 09) creates one instance per credential set and reuses it. Do not create one per upload.

## Acceptance criteria

- [ ] `S3KeyBuilder.photoKey("a.jpg", 1746816000000L)` returns `"photos/2025/05/09/a.jpg"` (verify exact formatting in a unit test — UTC, zero-padded).
- [ ] `S3KeyBuilder.thumbnailKey("a.jpg", 1746816000000L)` returns `"thumbnails/2025/05/09/a.webp"`.
- [ ] Calling `validateCredentials` with intentionally invalid keys returns `Result.failure` (no crash).
- [ ] Calling `upload` against a real B2 bucket places the object at the expected key. Verified once during Task 09's smoke test.
- [ ] `PutObjectRequest` builder used in `upload` sets `checksumAlgorithm = null` — visible in code.

## Out of scope

- Multipart upload. Photos are well under 100 MB; simple `PutObject` suffices.
- Pre-signed URLs (will be added post-MVP for cloud browsing).
- Retries beyond what the SDK does by default.
- Delete / list operations.
