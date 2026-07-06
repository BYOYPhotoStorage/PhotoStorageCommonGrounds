# Task 32 — PrefsStore Extensions

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Add every Phase 4 preference: first-backup scope and completion flag, video settings, upload mode, local-delete strategy + companion knobs, and the rolling monthly egress counter for the cost dashboard. Continue using `EncryptedSharedPreferences` (same store as Phases 2–3).

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/data/PrefsStore.kt` (modify — add methods)

## Updated contract

Keep all existing Phase 2 + Phase 3 methods unchanged. Add the following methods. Defaults match the PRD; all setters that take a nullable timestamp accept `null` to remove the key.

```kotlin
class PrefsStore(context: Context) {
    // ...existing...

    // ─── First-backup flow ────────────────────────────────────────
    /** "today" | "all". Default "today". */
    fun getFirstBackupScope(): String
    fun setFirstBackupScope(scope: String)

    /** Set true once FirstBackupActivity completes. Read by MainActivity routing. */
    fun hasCompletedFirstBackupFlow(): Boolean
    fun setFirstBackupFlowCompleted(completed: Boolean)

    // ─── Videos ───────────────────────────────────────────────────
    /** Default false. */
    fun getVideosEnabled(): Boolean
    fun setVideosEnabled(enabled: Boolean)

    /** "original" | "compressed" | "duration_based". Default "duration_based". */
    fun getVideoQualityMode(): String
    fun setVideoQualityMode(mode: String)

    /** For "duration_based" mode. Default 2. */
    fun getVideoDurationThresholdMinutes(): Int
    fun setVideoDurationThresholdMinutes(minutes: Int)

    /** "720p" | "1080p". Default "720p". */
    fun getVideoTargetResolution(): String
    fun setVideoTargetResolution(resolution: String)

    // ─── Upload mode ──────────────────────────────────────────────
    /** "immediate" | "scheduled" | "hybrid". Default "immediate". */
    fun getUploadMode(): String
    fun setUploadMode(mode: String)

    // ─── Local deletion ───────────────────────────────────────────
    /** "never" | "immediate" | "after_days" | "after_count". Default "never". */
    fun getLocalDeleteStrategy(): String
    fun setLocalDeleteStrategy(strategy: String)

    /** Default 30. */
    fun getLocalDeleteDays(): Int
    fun setLocalDeleteDays(days: Int)

    /** Default 100. */
    fun getLocalDeleteCount(): Int
    fun setLocalDeleteCount(count: Int)

    /** When set, no daily prompt fires until after this timestamp. */
    fun getLocalDeleteSuppressUntil(): Long?
    fun setLocalDeleteSuppressUntil(timestamp: Long?)

    /** Last time the LocalDeleteWorker ran (success or no-op). */
    fun getLastLocalDeleteRunAt(): Long?
    fun setLastLocalDeleteRunAt(timestamp: Long?)

    /** Consecutive days the user has dismissed the delete notification. */
    fun getLocalDeleteDismissStreak(): Int
    fun setLocalDeleteDismissStreak(count: Int)

    // ─── Cost dashboard ───────────────────────────────────────────
    /** Rolling counter, reset to 0 when the month anchor flips. */
    fun getEgressBytesMonth(): Long
    fun setEgressBytesMonth(bytes: Long)

    /** Timestamp anchoring the current rolling-month window. */
    fun getEgressMonthAnchor(): Long
    fun setEgressMonthAnchor(timestamp: Long)
}
```

### Key constants

Define at the top of `PrefsStore` for grep-ability:

```kotlin
private const val KEY_FIRST_BACKUP_SCOPE = "first_backup_scope"
private const val KEY_FIRST_BACKUP_FLOW_COMPLETED = "first_backup_flow_completed"
private const val KEY_VIDEOS_ENABLED = "videos_enabled"
private const val KEY_VIDEO_QUALITY_MODE = "video_quality_mode"
private const val KEY_VIDEO_DURATION_THRESHOLD_MINUTES = "video_duration_threshold_minutes"
private const val KEY_VIDEO_TARGET_RESOLUTION = "video_target_resolution"
private const val KEY_UPLOAD_MODE = "upload_mode"
private const val KEY_LOCAL_DELETE_STRATEGY = "local_delete_strategy"
private const val KEY_LOCAL_DELETE_DAYS = "local_delete_days"
private const val KEY_LOCAL_DELETE_COUNT = "local_delete_count"
private const val KEY_LOCAL_DELETE_SUPPRESS_UNTIL = "local_delete_suppress_until"
private const val KEY_LAST_LOCAL_DELETE_RUN_AT = "last_local_delete_run_at"
private const val KEY_LOCAL_DELETE_DISMISS_STREAK = "local_delete_dismiss_streak"
private const val KEY_EGRESS_BYTES_MONTH = "cost_dashboard_egress_bytes_month"
private const val KEY_EGRESS_MONTH_ANCHOR = "cost_dashboard_egress_month_anchor"
```

### Validation

Every setter that accepts an enum-like string MUST validate. Invalid values silently fall back to the default — the UI is the only writer, so this is a defensive guardrail.

```kotlin
private val VALID_FIRST_BACKUP_SCOPES = setOf("today", "all")
private val VALID_VIDEO_QUALITY_MODES = setOf("original", "compressed", "duration_based")
private val VALID_VIDEO_TARGET_RESOLUTIONS = setOf("720p", "1080p")
private val VALID_UPLOAD_MODES = setOf("immediate", "scheduled", "hybrid")
private val VALID_LOCAL_DELETE_STRATEGIES = setOf("never", "immediate", "after_days", "after_count")
```

If the persisted value is missing OR invalid, return the documented default. If a setter receives an invalid value, normalize to the default before writing.

### Numeric range clamping

- `setVideoDurationThresholdMinutes`: clamp to `[1, 60]`.
- `setLocalDeleteDays`: clamp to `[1, 365]`.
- `setLocalDeleteCount`: clamp to `[1, 10000]`.
- `setLocalDeleteDismissStreak`: clamp to `[0, 999]`.
- `setEgressBytesMonth`: clamp negative input to `0`.

Reads never need clamping (writes guarantee valid state).

### Nullable-timestamp pattern

`SharedPreferences` has no nullable Long. Use the same sentinel pattern as Phase 3's `last_index_sync_at`: `getLong(key, -1L)` and map `-1L` to `null`. Setters that receive `null` call `edit().remove(key).apply()`.

## Implementation notes

- Reuse the existing `EncryptedSharedPreferences` instance — do not create a second one.
- Do not log preference values (consistent with credential-handling discipline from MVP).
- The egress counter (`EgressBytesMonth`) is updated by the thumbnail cache fetcher (Phase 3 Task 29) and read by Task 38. Phase 4 does NOT modify Task 29's code — it only reads the counter here. The hook for *writing* the counter is added by Task 38 when the cost dashboard service initializes (it wraps the existing cache fetcher with an instrumentation shim). This task only provides the storage.
- The month anchor flip is handled inside `setEgressBytesMonth`: if `now`'s month is different from the stored anchor's month, reset the counter to the incoming value and update the anchor to `now`. Callers do not have to think about it.

```kotlin
fun setEgressBytesMonth(bytes: Long) {
    val clamped = bytes.coerceAtLeast(0L)
    val now = System.currentTimeMillis()
    val anchor = getEgressMonthAnchor()
    if (sameMonth(anchor, now)) {
        prefs.edit().putLong(KEY_EGRESS_BYTES_MONTH, clamped).apply()
    } else {
        prefs.edit()
            .putLong(KEY_EGRESS_BYTES_MONTH, clamped)
            .putLong(KEY_EGRESS_MONTH_ANCHOR, now)
            .apply()
    }
}

private fun sameMonth(a: Long, b: Long): Boolean { /* Calendar comparison in default tz */ }
```

## Constraints

- No schema or storage backend changes — same encrypted preferences file as Phases 2–3.
- Do not introduce a separate prefs file.
- Do not surface a "reset all settings" affordance here — that would belong in Task 41.

## Dependencies (by interface)

- Phase 3 `PrefsStore` (Task 21). This task only extends it.

## Acceptance criteria

- [ ] On a fresh install, every getter returns its documented default (`today`, `false`, `duration_based`, `2`, `720p`, `immediate`, `never`, `30`, `100`, `null` for timestamps, `0` for the dismiss streak, `0` for the egress counter).
- [ ] `setFirstBackupScope("bogus")` followed by `getFirstBackupScope()` returns `"today"`.
- [ ] `setUploadMode("scheduled")` followed by `getUploadMode()` returns `"scheduled"`; setting `"bogus"` falls back to `"immediate"`.
- [ ] `setLocalDeleteSuppressUntil(null)` clears the value.
- [ ] `setVideoDurationThresholdMinutes(999)` followed by `getVideoDurationThresholdMinutes()` returns `60` (clamped).
- [ ] Setting `EgressBytesMonth` after the month anchor's calendar month has elapsed resets the counter and updates the anchor.
- [ ] All Phase 2 and Phase 3 `PrefsStore` methods continue to behave identically.

## Out of scope

- A migration step for legacy keys — Phase 4 introduces all-new keys; readers handle missing values via defaults.
- Settings UI (Task 41).
- The thumbnail fetcher instrumentation shim (Task 38).
- A configurable "reset to defaults" — not in Phase 4.
