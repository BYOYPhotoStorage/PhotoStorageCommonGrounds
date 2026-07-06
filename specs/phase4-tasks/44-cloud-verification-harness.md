# Task 44 — Cloud Verification Harness

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md), [`../../decisions/2026-07-03-mobile-testing-strategy-maestro-companion-app.md`](../../decisions/2026-07-03-mobile-testing-strategy-maestro-companion-app.md), and [`43-companion-app.md`](43-companion-app.md) first.

## Goal

Build a host-side Python CLI that independently verifies that media injected by the companion app has been uploaded correctly to the test B2 bucket. It must:

1. Read the companion app's injection report.
2. List objects in the target upload bucket (`test0307`).
3. Match injected files to bucket objects by SHA256 and size when possible.
4. Fall back to a "presence" match by injected display name when the main app has compressed or transcoded the file.
5. Report missing, mismatched, or extra objects.
6. Clean up test objects after a run.

Keeping verification on the host prevents the device from "passing" its own tests through a bug in its own upload logic, and it lets us use mature AWS/B2 tooling.

## Scope

- Python 3.11+ CLI.
- `boto3` for S3/B2 access.
- One verifier script and one cleanup script.
- No dependency on the main app's code.

## Files you own

New files under `photoStorageApp/e2e-runner/`:

- `verify_bucket.py`
- `cleanup_bucket.py`
- `requirements.txt`
- `.env.example`

Modified files: none in the main app.

## Dependencies

`photoStorageApp/e2e-runner/requirements.txt`:

```text
boto3>=1.34
python-dotenv>=1.0
```

`python-dotenv` is optional; the orchestration runner can also export environment variables directly.

## Configuration

The harness reads B2 credentials from environment variables (never hard-coded):

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `B2_KEY_ID` | yes | — | S3-compatible key ID |
| `B2_APPLICATION_KEY` | yes | — | S3-compatible application key |
| `B2_UPLOAD_BUCKET` | no | `test0307` | Bucket the main app uploads into |
| `B2_REGION` | no | `us-west-004` | B2 region |
| `B2_ENDPOINT_URL` | no | `https://s3.{region}.backblazeb2.com` | S3 endpoint |
| `E2E_REPORT_DIR` | no | `./e2e-reports` | Where to write result JSON |

The upload bucket is separate from the reference bucket (`refImages0307`) used by the companion app.

## CLI

### `verify_bucket.py`

```bash
python verify_bucket.py \
  --tag a1b2c3 \
  --report ./e2e-reports/a1b2c3.json \
  [--bucket test0307] \
  [--output ./e2e-reports] \
  [--require-exact]
```

Inputs:

- `--tag`: the run identifier shared with the companion app.
- `--report`: path to the pulled injection report (`/sdcard/.../reports/<tag>.json`).
- `--bucket`: upload B2 bucket to inspect. Defaults to `test0307`.
- `--output`: directory for the verification report.
- `--require-exact`: if set, a "presence" match is treated as a failure. Useful for deduplication or no-compression scenarios.

Behavior:

1. Load the injection report and build an expected set keyed by `displayName`.
2. Create an S3 client pointing at B2.
3. List all objects under the prefixes:
   - `photos/`
   - `videos/`
   - `thumbnails/`
4. For each expected item, search the bucket listing for a candidate whose key contains the injected `displayName`.
5. Try to confirm an **exact** match:
   - Candidate `Size` equals `sizeBytes` **and**
   - Candidate SHA256 equals `sha256` (computed by downloading the candidate if not already known).
6. If exact match fails but a candidate with the right display name exists, record a **presence** match. This is the default acceptance path for images the main app may have compressed, or videos it may have transcoded.
7. Classify results:
   - **found (exact)**: object exists with matching hash and size.
   - **found (presence)**: object exists with the right display name but hash/size differ.
   - **sizeMismatch**: object exists with the right display name but size is outside tolerance and `--require-exact` is set.
   - **hashMismatch**: object exists with the right display name and matching size but wrong hash and `--require-exact` is set.
   - **missing**: no candidate found.
8. Look for unexpected objects whose key contains `ps_test_<tag>` but which are not in the report (e.g., duplicate uploads from a previous run that were not cleaned up).
9. Write `verified-<tag>.json` and exit `0` only if every expected item is found (exact or presence) and there are no unexpected objects, unless `--require-exact` changes the rules.

Output `verified-<tag>.json`:

```json
{
  "tag": "a1b2c3",
  "bucket": "test0307",
  "verifiedAt": "2026-07-04T12:40:00Z",
  "allFound": true,
  "unexpectedCount": 0,
  "requireExact": false,
  "items": [
    {
      "displayName": "ps_test_a1b2c3_001_DSC_1234.jpg",
      "kind": "photo",
      "expectedSha256": " salted-sha256...",
      "expectedSizeBytes": 205100,
      "found": true,
      "matchType": "presence",
      "bucketKey": "photos/2026/07/04/ps_test_a1b2c3_001_DSC_1234.jpg",
      "bucketSize": 198000,
      "bucketETag": "\"abc123...\""
    }
  ],
  "missing": [],
  "mismatched": [],
  "unexpected": []
}
```

### `cleanup_bucket.py`

```bash
python cleanup_bucket.py --tag a1b2c3 [--bucket test0307]
python cleanup_bucket.py --all-test-media [--bucket test0307]
```

Modes:

- `--tag <tag>`: delete every object whose key contains `ps_test_<tag>`.
- `--all-test-media`: delete every object whose key contains `ps_test_` (use with caution).

The orchestration runner should call the tag-scoped cleanup in a `trap` so it runs even if the test fails.

## Matching strategy

The main app uses the following key layout (see `S3KeyBuilder`):

- Photos: `photos/yyyy/MM/dd/<displayName>`
- Thumbnails: `thumbnails/yyyy/MM/dd/<basename>.webp`
- Videos: `videos/yyyy/MM/dd/<displayName>` (or `.compressed.mp4` when transcoded)

Because the harness does not know exactly when the upload worker ran, matching by date prefix is fragile. The canonical steps are:

1. Build a candidate list from `photos/`, `videos/`, and `thumbnails/` prefixes.
2. For each expected item, find candidates whose key contains the injected `displayName`.
3. Try exact SHA256 + size match first.
4. Fall back to presence match if exact is not required.

For thumbnails, the expected file is a WebP generated by the main app. The companion app does not know the thumbnail's final bytes or hash, so the harness should **not** expect thumbnails for injected items unless the main app is configured to generate them and the orchestration runner waits for thumbnail upload. The default verification scope is the original photo/video only. A future flag `--verify-thumbnails` can extend this.

## Notes on compression and transcoding

The main app may:

- Compress images before upload.
- Transcode videos before upload.

The harness handles both by default:

- If the uploaded object has the same SHA256 and size as the injected file, record `matchType: exact`.
- If the uploaded object has the same `displayName` but different bytes, record `matchType: presence` and still count the item as found.

This lets tests prove that the pipeline moved the file from the device to the cloud even when the bytes changed. Use `--require-exact` for stricter checks when compression/transcoding is disabled.

## Testing the harness in isolation

1. Push a known file to the test bucket manually:
   ```bash
   aws s3 cp ./DSC_1234.jpg s3://test0307/photos/2026/07/04/ps_test_manual_001_DSC_1234.jpg \
     --endpoint-url=https://s3.us-west-004.backblazeb2.com
   ```
2. Create a hand-written report with a matching `displayName` and a different `sha256` to exercise the presence-match path.
3. Run `verify_bucket.py` against it and confirm `allFound=true`.
4. Run `cleanup_bucket.py --tag manual` and confirm the object is gone.

## Security

- Never commit credentials.
- Use a bucket-scoped application key for CI.
- Run cleanup even on failure so test data does not accumulate.
- The harness only reads/writes the upload bucket; it never touches production buckets or the reference bucket.

## Open decisions handled here

From the mobile-testing ADR:

- **Cloud harness language:** Python, using `boto3`. This is faster to script than Kotlin and has better AWS/B2 examples.
- **Bucket verification responsibility:** Host-side, not in the companion app. The companion app supplies the report; the harness supplies the verdict.
- **Compression/transcoding handling:** Verified by presence match by default, with an optional strict exact-match mode.
