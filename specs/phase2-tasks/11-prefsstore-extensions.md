# Task 11 — PrefsStore Phase 2 Extensions

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Extend the MVP `PrefsStore` with three new settings keys needed by Phase 2: auto-upload toggle, Wi-Fi-only mode, and the last MediaStore scan timestamp.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/data/PrefsStore.kt` (modify — append methods and constants)

## Public contract

Add these methods to the existing `PrefsStore` class:

```kotlin
class PrefsStore(context: Context) {
    // ... existing MVP methods ...

    fun isAutoUploadEnabled(): Boolean
    fun setAutoUploadEnabled(enabled: Boolean)

    fun isWifiOnly(): Boolean
    fun setWifiOnly(enabled: Boolean)

    fun getLastScanTimestamp(): Long
    fun setLastScanTimestamp(timestamp: Long)
}
```

### Method semantics

| Method | Key | Default | Notes |
|--------|-----|---------|-------|
| `isAutoUploadEnabled()` | `"auto_upload_enabled"` | `false` | Whether the foreground service should run. |
| `setAutoUploadEnabled(enabled)` | `"auto_upload_enabled"` | — | Writes boolean with `apply()`. |
| `isWifiOnly()` | `"wifi_only_uploads"` | `false` | When true, uploads only on unmetered networks. |
| `setWifiOnly(enabled)` | `"wifi_only_uploads"` | — | Writes boolean with `apply()`. |
| `getLastScanTimestamp()` | `"last_scan_timestamp"` | `0L` | Unix millis of the last successful MediaStore scan. Used by ContentObserver to query only newer photos. |
| `setLastScanTimestamp(timestamp)` | `"last_scan_timestamp"` | — | Writes long with `apply()`. |

## Implementation notes

Store the new keys in the **same** `EncryptedSharedPreferences` file (`"b2_credentials"`) that the MVP created. No need for a separate file — the encryption overhead is the same and it keeps all app settings together.

Add companion constants:

```kotlin
companion object {
    // ... existing constants ...
    private const val AUTO_UPLOAD_ENABLED = "auto_upload_enabled"
    private const val WIFI_ONLY = "wifi_only_uploads"
    private const val LAST_SCAN_TIMESTAMP = "last_scan_timestamp"
}
```

## Constraints

- Use the existing `EncryptedSharedPreferences` instance — do not create a second file.
- All writes use `apply()` (async).
- Booleans default to `false`, the timestamp defaults to `0L`.

## Acceptance criteria

- [ ] `setAutoUploadEnabled(true)` followed by a new `PrefsStore` instance returns `true` from `isAutoUploadEnabled()`.
- [ ] `setWifiOnly(true)` persists and is readable across instances.
- [ ] `setLastScanTimestamp(12345L)` persists and is readable across instances.
- [ ] All new keys are stored in the same encrypted prefs file as credentials.
- [ ] No change to existing credential methods.

## Out of scope

- Settings UI (Task 18 exposes these via toggle; the UI is not in this task).
- Default value migration or user prompting.
