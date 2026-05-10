# ADR: Use S3-Compatible API Over Native B2 API

## Status

Accepted

## Context

Backblaze B2 offers two APIs for interacting with object storage:

1. **Native B2 API** — B2-specific endpoints, custom authentication tokens, and a bespoke request/response format.
2. **S3-Compatible API** — Standard AWS S3 semantics, AWS Signature Version 4 authentication, and compatibility with existing S3 tools and SDKs.

The MVP requires an Android app that uploads photos directly from the phone to B2. We needed to choose which API to build against.

## Decision

Use the **S3-Compatible API** via the **AWS SDK for Kotlin**.

## Consequences

### Positive

- **SDK maturity.** The AWS SDK for Kotlin provides coroutine-native APIs, built-in retry logic, and streaming uploads — none of which exist for the Native B2 API on Android.
- **Future portability.** If we later want to support Wasabi, Storj, MinIO, or AWS S3 itself, zero upload logic changes are required. Only the endpoint URL and credentials change.
- **Familiarity.** S3 is the industry standard. Developers, documentation, and Stack Overflow answers are abundant.
- **Pre-signed URLs.** Built-in support for generating temporary download URLs, which simplifies thumbnail loading in the gallery (Glide can fetch directly from the URL).
- **No custom HTTP layer.** We avoid hand-rolling OkHttp calls, token refresh logic, and B2-specific error handling.

### Negative

- **Checksum header incompatibility.** AWS SDK for Kotlin v1.3+ automatically sends `x-amz-checksum-crc32` headers. B2 rejects these. We must explicitly set `checksumAlgorithm = null` on every `PutObject` and `UploadPart` request.
- **Larger dependency size.** The AWS SDK for Kotlin adds ~3–5 MB to the APK. The Native B2 API could have been implemented with just OkHttp (~500 KB).
- **S3 feature gaps on B2.** B2 does not support object tagging, IAM roles, or website configuration through the S3-compatible API. None of these are needed for a photo backup app, but it limits future S3 feature exploration.

## Alternatives Considered

### Native B2 API

- **Pros:** Smaller dependency footprint; direct access to all B2 features (e.g., lifecycle rules, native file info).
- **Cons:** No first-party Android SDK; would require hand-rolling all HTTP calls, auth token management, and upload URL caching. Higher implementation cost and more bugs.
- **Verdict:** Rejected. The implementation overhead outweighs the dependency size savings.

### MinIO Java Client

- **Pros:** Lightweight (~1 MB), S3-compatible, no checksum header issues with B2.
- **Cons:** Java API (not coroutine-native), smaller community, less documentation for Android.
- **Verdict:** Identified as a fallback if the AWS SDK checksum issue becomes unworkable during MVP implementation. Not the primary choice.

## Mitigations

- The checksum issue is documented and tested in the first week of MVP development. If it proves fragile, we will switch to the MinIO client without changing the rest of the architecture.
- The dependency size is acceptable for an MVP focused on proving the pipeline. If APK size becomes critical, we can evaluate the MinIO client or ProGuard/R8 shrinking.
