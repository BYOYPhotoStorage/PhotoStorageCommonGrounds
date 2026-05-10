# Backblaze B2 S3-Compatible API Research

## Executive Summary

Backblaze B2 offers an S3-compatible API that allows existing S3-based tools and SDKs to work with minimal changes. For our Android photo backup app, using the S3-compatible API (via the AWS SDK for Kotlin) is recommended over the Native B2 API because:

1. **Familiarity:** S3 is the industry standard; more developers know it
2. **Future portability:** Easy to switch to Wasabi, Storj, MinIO, or other S3-compatible providers
3. **Rich SDK ecosystem:** AWS SDK for Kotlin handles signing, retries, and streaming
4. **Pre-signed URLs:** Built-in support for temporary share links

---

## Authentication

B2 S3-compatible API uses **standard AWS Signature Version 4 (SigV4)**.

| S3 Term | B2 Equivalent |
|---------|--------------|
| `Access Key ID` | B2 **Application Key ID** (e.g., `005...`) |
| `Secret Access Key` | B2 **Application Key** (the secret) |
| `Region` | Your bucket's region code (e.g., `us-west-004`) |

### Creating S3-Compatible Keys

In the B2 web console:
1. Go to **App Keys**
2. Create a new key with access to your bucket
3. The **Key ID** and **Key** are your S3 `Access Key ID` and `Secret Access Key`

> **Important:** Use an **application key** restricted to a single bucket, not your master account key. This limits blast radius if the key is compromised.

---

## Endpoints

The endpoint format is:

```
https://s3.<region>.backblazeb2.com
```

### Known Regions (as of 2026)

| Region | Endpoint |
|--------|----------|
| US West | `s3.us-west-000.backblazeb2.com` |
| US West | `s3.us-west-004.backblazeb2.com` |
| US East | `s3.us-east-005.backblazeb2.com` |
| EU Central | `s3.eu-central-003.backblazeb2.com` |

The exact region code for your bucket is visible in the B2 console. You must use the endpoint matching your bucket's region.

---

## Supported Operations

The following S3 operations are supported and sufficient for our MVP:

| Operation | S3 API | B2 Support | MVP Use |
|-----------|--------|------------|---------|
| List buckets | `ListBuckets` | ✅ Yes | Verify credentials on onboarding |
| Put object | `PutObject` | ✅ Yes | Upload photo and thumbnail |
| Get object | `GetObject` | ✅ Yes | Download photo for viewing |
| Delete object | `DeleteObject` | ✅ Yes | Delete from cloud |
| List objects | `ListObjectsV2` | ✅ Yes | Browse bucket contents |
| Head object | `HeadObject` | ✅ Yes | Check if file exists without downloading |
| Pre-signed GET | `Presigner` | ✅ Yes | Generate temporary view links |
| Pre-signed PUT | `Presigner` | ✅ Yes | Generate temporary upload links |

### Partially Supported

| Operation | Limitation |
|-----------|-----------|
| `PutBucketAcl` | Only `private` ↔ `public-read` toggle |
| `GetObjectAcl` | Returns bucket-level ACL (always `private` for private buckets) |
| `CopyObject` | Works, but `x-amz-tagging` headers are ignored |

### Not Supported

- Object-level ACLs
- IAM Roles
- Object tagging (`GetObjectTagging` returns empty tags)
- Website configuration
- Browser-based POST uploads to pre-signed URLs
- Object Lock
- Batch Operations
- Access Points

**Impact on our app:** None of the unsupported features are needed for a photo backup app.

---

## Pricing

| Component | Cost | Notes |
|-----------|------|-------|
| **Storage** | $6.95 / TB / month (~$0.00695 / GB / month) | First 10 GB free |
| **Upload** | Free | No ingress charges |
| **Download (Egress)** | Free up to 3× monthly storage | Then $0.01 / GB |
| **API Calls (Class A)** | Free | Uploads, list buckets, list files |
| **API Calls (Class B)** | Free | Downloads, file info |
| **API Calls (Class C)** | Free | Authorization |
| **API Calls (Class D)** | $0.004 / 10,000 calls | First 2,500/day free |
| **Minimum file size** | None | No penalties for small files |
| **Minimum storage duration** | None | Delete anytime |

### Cost Example: Personal Photo Library

| Scenario | Estimate |
|----------|----------|
| 10,000 photos @ 5 MB avg | ~50 GB storage |
| Monthly storage cost | ~$0.35 |
| Upload 100 new photos/week | Free |
| Browse thumbnails (200 KB each), 500 views/month | ~100 MB egress = Free |
| **Total monthly cost** | **~$0.35** |

Compare to Google Cloud Storage Standard: ~$1.20/TB/month storage + $0.12/GB egress. B2 is dramatically cheaper for personal use.

---

## Known Compatibility Issues

### AWS SDK v3 Checksum Headers (2024+)

Newer AWS SDK versions automatically compute and send `x-amz-checksum-crc32` headers. **B2 does not support these headers** and returns a `400 Bad Request`.

**Workaround** (required for AWS SDK for Kotlin v1.3+):

```kotlin
val s3 = S3Client {
    region = "us-west-004"
    endpointUrl = Url.parse("https://s3.us-west-004.backblazeb2.com")
    credentialsProvider = StaticCredentialsProvider {
        accessKeyId = "YOUR_KEY_ID"
        secretAccessKey = "YOUR_KEY"
    }
    // Disable automatic checksum calculation
    httpClient {
        // Use OkHttp engine for Android
    }
}
```

For the AWS SDK for Kotlin, you may need to disable checksums at the request level:

```kotlin
s3.putObject {
    bucket = "my-bucket"
    key = "photos/2025/01/01/img.jpg"
    body = file.asByteStream()
    // Explicitly disable checksum
    checksumAlgorithm = null
}
```

> **Recommendation:** Test checksum behavior early in MVP implementation. This is the most common friction point when using modern AWS SDKs with B2.

### Path-Style vs Virtual-Hosted-Style URLs

B2 supports both. Use whichever your SDK defaults to. If you encounter issues, try forcing path-style:

```kotlin
S3Client {
    endpointUrl = Url.parse("https://s3.us-west-004.backblazeb2.com")
    // Path-style is the default for custom endpoints in most SDKs
}
```

---

## Android SDK Options

### Option 1: AWS SDK for Kotlin (Recommended)

```groovy
dependencies {
    implementation("aws.sdk.kotlin:s3:1.6.46")
    implementation("aws.smithy.kotlin:http-client-engine-okhttp:1.0.0")
}
```

**Pros:**
- First-class Kotlin coroutine support (`suspend` functions)
- Modern, actively maintained
- OkHttp engine works well on Android
- Built-in retry logic
- Streaming uploads (no need to load entire file into memory)

**Cons:**
- Larger dependency size (~3-5 MB)
- Checksum header issue with B2 (workaround required)

**Example:**

```kotlin
import aws.sdk.kotlin.services.s3.S3Client
import aws.sdk.kotlin.services.s3.model.PutObjectRequest
import aws.smithy.kotlin.runtime.content.asByteStream

class B2Uploader(private val keyId: String, private val appKey: String) {
    private val s3 = S3Client {
        region = "us-west-004"
        endpointUrl = Url.parse("https://s3.us-west-004.backblazeb2.com")
        credentialsProvider = StaticCredentialsProvider {
            accessKeyId = keyId
            secretAccessKey = appKey
        }
    }

    suspend fun uploadPhoto(bucket: String, key: String, file: File) {
        s3.putObject(PutObjectRequest {
            bucket = bucket
            this.key = key
            body = file.asByteStream()
        })
    }
}
```

### Option 2: AWS SDK for Java v2

```groovy
dependencies {
    implementation("software.amazon.awssdk:s3:2.25.0")
}
```

**Pros:**
- Mature, well-documented
- Smaller than Kotlin SDK in some configurations

**Cons:**
- Java futures/callbacks (not coroutine-native)
- Verbose API
- Same checksum header issues

### Option 3: MinIO Java Client

```groovy
dependencies {
    implementation("io.minio:minio:8.5.0")
}
```

**Pros:**
- Lightweight (~1 MB)
- S3-compatible by design
- No checksum header issues

**Cons:**
- Java API, requires Kotlin wrappers
- Less widely used than AWS SDK
- Smaller community

### Recommendation for MVP

**Use AWS SDK for Kotlin.** Despite the checksum quirk, it's the best long-term choice:
- Coroutine-native = clean Android code
- If we ever switch to AWS S3, Wasabi, or Storj, zero code changes
- Active development and Android support

If the checksum issue proves problematic during MVP, fall back to **MinIO client** as a lightweight alternative.

---

## Upload Strategy for Photos

### Small Files (< 100 MB)

Use simple `PutObject`:

```kotlin
s3.putObject {
    bucket = bucketName
    key = "photos/2025/01/01/IMG_1234.jpg"
    body = file.asByteStream()
    contentType = "image/jpeg"
}
```

### Large Files (> 100 MB, e.g., videos)

Use **multipart upload**:

```kotlin
// 1. Initiate
val createResp = s3.createMultipartUpload { ... }
val uploadId = createResp.uploadId

// 2. Upload parts (5 MB minimum per part)
// 3. Complete
s3.completeMultipartUpload { ... }
```

For MVP (photos only, no videos), simple `PutObject` is sufficient.

---

## Download Strategy for Thumbnails

For the gallery's cloud view, fetch thumbnails on demand:

```kotlin
suspend fun downloadThumbnail(key: String): ByteArray {
    s3.getObject(GetObjectRequest {
        bucket = bucketName
        this.key = key
    }) { response ->
        return response.body?.readAllBytes() ?: byteArrayOf()
    }
}
```

Cache thumbnails locally using Android's `CacheManager` or Glide:

```kotlin
Glide.with(context)
    .load(generatePresignedUrl(key))
    .into(imageView)
```

### Pre-signed URLs for Gallery View

Instead of downloading through the SDK, generate a temporary URL and let Glide handle it:

```kotlin
suspend fun getThumbnailUrl(key: String, expireHours: Int = 1): String {
    val request = GetObjectRequest { bucket = bucketName; this.key = key }
    val presigned = s3.presignGetObject(request, expireHours.hours)
    return presigned.url.toString()
}
```

This offloads download handling to Glide and reduces memory pressure.

---

## Comparison: S3-Compatible API vs Native B2 API

| Aspect | S3-Compatible API | Native B2 API |
|--------|------------------|---------------|
| **Authentication** | AWS SigV4 | B2 custom auth tokens |
| **SDK availability** | Excellent (AWS SDK) | Limited (community libraries) |
| **Upload large files** | Multipart upload | `b2_upload_part` (similar) |
| **Pre-signed URLs** | Built-in | Not available |
| **Android support** | First-class | Weak |
| **Learning curve** | Low (S3 is standard) | Medium (B2-specific) |
| **Portability** | High (any S3 provider) | Low (B2 only) |
| **Feature richness** | S3-standard | B2-specific features (e.g., lifecycle rules) |

**Verdict:** S3-compatible API is the better choice for an Android app.

---

## Security Best Practices

1. **Use bucket-restricted keys:** Never ship the master account key in the app
2. **Rotate keys periodically:** B2 allows creating new keys and revoking old ones
3. **HTTPS only:** Always use `https://` endpoints
4. **Don't log credentials:** Ensure `accessKeyId` and `secretAccessKey` never appear in logs
5. **Pre-signed URL expiry:** Keep short (1 hour max) for share links

---

## Sources

- [Backblaze B2 S3-Compatible API Docs](https://www.backblaze.com/b2/docs/s3_compatible_api.html)
- [Backblaze B2 Pricing](https://www.backblaze.com/cloud-storage/pricing)
- [AWS SDK for Kotlin S3 Examples](https://docs.aws.amazon.com/sdk-for-kotlin/latest/developer-guide/kotlin_s3_code_examples.html)
- [AWS SDK for Kotlin Configuration](https://docs.aws.amazon.com/sdk-for-kotlin/latest/developer-guide/configuration.html)
- [AWS SDK for Kotlin HTTP Client](https://docs.aws.amazon.com/sdk-for-kotlin/latest/developer-guide/http-client-config.html)
- [Maven Central - AWS SDK Kotlin S3](https://central.sonatype.com/artifact/aws.sdk.kotlin/s3)
