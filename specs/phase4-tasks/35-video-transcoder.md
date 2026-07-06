# Task 35 — Video Transcoder

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Provide a `Transcoder` that takes a video URI and produces an H.264 + AAC MP4 at a configured target resolution, suitable for upload. Wraps AndroidX Media3 `Transformer` so the rest of the app does not have to learn its API. Used by Task 36 in the upload worker when compression is required by the active video quality mode.

The Gradle dependency is added in Task 42, not here. This task assumes the dependency is available (the `imports` will be unresolved until Task 42 wires it).

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/media/Transcoder.kt` (new)

## Contract

```kotlin
class Transcoder(private val context: Context) {
    /**
     * Transcodes an input video URI to MP4 H.264 + AAC at the given target resolution.
     * Aspect ratio is preserved (target is treated as the maximum height).
     *
     * progress is invoked from a worker thread with values in [0.0, 1.0].
     *
     * Cancellation: callers should run inside a coroutine and call cancel() on the job to abort.
     * The Transformer build is set up to honor coroutine cancellation; partial output files
     * are deleted before returning.
     */
    suspend fun transcode(
        input: Uri,
        output: File,
        targetResolution: String, // "720p" | "1080p"
        progress: (Float) -> Unit = {}
    ): Result<TranscodeResult>
}

data class TranscodeResult(
    val outputFile: File,
    val outputSizeBytes: Long,
    val durationMs: Long
)
```

### Method semantics

- `targetResolution = "720p"` → max height 720 px. `"1080p"` → 1080 px. Any other value returns `Result.failure(IllegalArgumentException)`.
- `output`: caller-owned path. The transcoder writes here; on success the file exists and `outputSizeBytes` is its length. On failure, the file is deleted before returning.
- Audio is encoded to AAC at 128 kbps mono/stereo (matches source channel count, capped at stereo).
- Video is encoded to H.264 at a target bitrate proportional to resolution:
  - 720p → 4 Mbps
  - 1080p → 8 Mbps
- Frame rate is preserved (capped at 30 fps if source is higher — `Transformer` handles this).
- `progress` invocations are throttled to ~250 ms intervals; callers do not need to debounce.

### Failure modes

| Cause | `Result` |
|-------|----------|
| Unsupported codec / corrupt input | `Result.failure(IOException(...))` with message including detail |
| Insufficient disk space in `output`'s parent | `Result.failure(IOException("No space available"))` |
| Invalid `targetResolution` | `Result.failure(IllegalArgumentException)` |
| Cancellation | `Result.failure(CancellationException)` (re-thrown) |
| Any `Transformer` error | `Result.failure(IOException)` wrapping the SDK error |

In every failure case, attempt `output.delete()` before returning so callers do not have to clean up.

## Implementation sketch

```kotlin
class Transcoder(private val context: Context) {

    suspend fun transcode(
        input: Uri,
        output: File,
        targetResolution: String,
        progress: (Float) -> Unit = {}
    ): Result<TranscodeResult> = suspendCancellableCoroutine { cont ->
        val targetHeight = when (targetResolution) {
            "720p" -> 720
            "1080p" -> 1080
            else -> {
                cont.resume(Result.failure(IllegalArgumentException("Unknown targetResolution: $targetResolution"))) {}
                return@suspendCancellableCoroutine
            }
        }
        val bitrate = if (targetHeight == 720) 4_000_000 else 8_000_000

        val mediaItem = MediaItem.fromUri(input)
        val editedMediaItem = EditedMediaItem.Builder(mediaItem)
            .setEffects(Effects(emptyList(), listOf(ScaleAndRotateTransformation.Builder()
                .setRotationDegrees(0f)
                .build())))
            .build()

        val transformer = Transformer.Builder(context)
            .setVideoMimeType(MimeTypes.VIDEO_H264)
            .setAudioMimeType(MimeTypes.AUDIO_AAC)
            .setEncoderFactory(
                DefaultEncoderFactory.Builder(context)
                    .setRequestedVideoEncoderSettings(
                        VideoEncoderSettings.Builder()
                            .setBitrate(bitrate)
                            .setEncoderPerformanceParameters(VideoEncoderSettings.NO_VALUE, VideoEncoderSettings.NO_VALUE)
                            .build()
                    )
                    .build()
            )
            .addListener(object : Transformer.Listener {
                override fun onCompleted(composition: Composition, exportResult: ExportResult) {
                    val result = TranscodeResult(output, output.length(), exportResult.durationMs)
                    cont.resume(Result.success(result)) {}
                }
                override fun onError(composition: Composition, exportResult: ExportResult, exportException: ExportException) {
                    output.delete()
                    cont.resume(Result.failure(IOException(exportException))) {}
                }
            })
            .build()

        // Wire progress polling via Handler — Transformer's ProgressHolder
        val handler = Handler(Looper.getMainLooper())
        val progressHolder = ProgressHolder()
        val pollProgress = object : Runnable {
            override fun run() {
                val state = transformer.getProgress(progressHolder)
                if (state != Transformer.PROGRESS_STATE_NOT_STARTED && state != Transformer.PROGRESS_STATE_UNAVAILABLE) {
                    progress(progressHolder.progress / 100f)
                }
                if (cont.isActive) handler.postDelayed(this, 250)
            }
        }
        handler.post(pollProgress)

        cont.invokeOnCancellation {
            handler.removeCallbacks(pollProgress)
            transformer.cancel()
            output.delete()
        }

        transformer.start(editedMediaItem, output.absolutePath)
    }
}
```

> Exact class names depend on the Media3 version added in Task 42 (`media3-transformer:1.x`). If the API surface differs in the chosen version (e.g., `Composition` is replaced by `EditedMediaItemSequence`), adapt structurally but keep the public contract above identical.

## Aspect ratio and scaling

`Transformer` accepts a `Presentation` effect that scales to a target height while preserving aspect ratio. Use:

```kotlin
.setEffects(Effects(emptyList(), listOf(Presentation.createForHeight(targetHeight))))
```

instead of the inline `ScaleAndRotateTransformation` placeholder above — that line is a sketch placeholder. Verify against the Media3 docs for the chosen version.

## Free-disk-space pre-check

Before invoking `Transformer.start`, verify `output.parentFile?.usableSpace` is at least `inputSize * 1.2` (rough safety margin for VBR encoding). If not, return `Result.failure(IOException("No space available"))` without starting.

```kotlin
val inputSize = runCatching {
    context.contentResolver.openAssetFileDescriptor(input, "r")?.use { it.length }
}.getOrNull() ?: 0L
val available = output.parentFile?.usableSpace ?: 0L
if (inputSize > 0 && available < (inputSize * 12 / 10)) {
    return@suspendCancellableCoroutine cont.resume(Result.failure(IOException("No space available"))) {}
}
```

## Implementation notes

- Runs on `Dispatchers.IO` from the caller's perspective. Internally `Transformer` may use its own threads; the suspending wrapper bridges them.
- Output format is always MP4. Do not try to match the source container.
- Hardware-accelerated encoding is preferred. `DefaultEncoderFactory` picks hardware codecs by default — do not force software.
- Do not call `MediaMetadataRetriever` here — that is Task 36's job for thumbnail generation.
- The transcoder does not know about B2 paths or upload semantics. It only produces an MP4 on disk.

## Constraints

- Hardcoded codec choices (H.264 + AAC) — broad device support. Do not switch to HEVC despite better compression: playback compatibility for share links would suffer.
- Bitrates are hardcoded (4/8 Mbps). Do not expose them in `PrefsStore` in Phase 4.
- No two-pass encoding (battery + time cost). Single-pass VBR is acceptable.
- Mute is never produced — if the source has no audio, the output has no audio track too (do not silently inject one).
- The transcoder MUST honor coroutine cancellation. A killed worker should not leak a `Transformer` instance or a partial file.

## Dependencies (by interface)

- AndroidX Media3 (`androidx.media3:media3-transformer`, `androidx.media3:media3-effect`, `androidx.media3:media3-common`) — added in Task 42.
- No application-layer dependencies. The transcoder is pure media + Context.

## Acceptance criteria

- [ ] Transcoding a 10-second 1080p test video at `targetResolution="720p"` produces a smaller MP4 with playable H.264 + AAC tracks (verify with `ffprobe` or in-app playback).
- [ ] Output dimensions: height = 720 (or 1080), width scales proportionally, even-numbered (the encoder pads if needed).
- [ ] `progress` callback fires at least once during a multi-second transcode.
- [ ] Cancelling the coroutine mid-transcode terminates the `Transformer` and deletes the partial output file.
- [ ] Invalid resolution returns `Result.failure(IllegalArgumentException)` without creating output.
- [ ] Pre-check returns `Result.failure(IOException("No space available"))` when usable space is below the safety margin.
- [ ] Failed transcode (e.g., corrupted input) returns `Result.failure(IOException)` and deletes any partial output.

## Out of scope

- HEVC / VP9 / AV1 output.
- Hardware-vs-software codec selection.
- Multi-resolution ladders (HLS/DASH).
- Trim / crop / overlay effects.
- Live-photo / motion-photo handling.
- Audio normalization.
- Two-pass encoding.
- Adaptive bitrate.
