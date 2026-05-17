# Task 21 — PrefsStore Extensions

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Add the three Phase 3 preferences: gallery view mode, last-synced index hash, and last index sync timestamp. Continue using `Encrypted SharedPreferences` (the same store as Phase 2).

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/data/PrefsStore.kt` (modify — add methods)

## Updated contract

Keep all existing Phase 2 methods unchanged. Add:

```kotlin
class PrefsStore(context: Context) {
    // ...existing methods...

    /** "local" | "cloud" | "merged". Default "merged". */
    fun getGalleryViewMode(): String
    fun setGalleryViewMode(mode: String)

    /** SHA-256 hex of the last successfully uploaded index file, or null if never synced. */
    fun getLastSyncedIndexHash(): String?
    fun setLastSyncedIndexHash(hash: String?)

    /** Unix millis of the last successful index sync, or null. */
    fun getLastIndexSyncAt(): Long?
    fun setLastIndexSyncAt(timestamp: Long?)
}
```

### Method semantics

- `getGalleryViewMode`: read `gallery_view_mode` key; if missing or empty, return `"merged"`.
- `setGalleryViewMode`: write the literal mode string (validate against `local|cloud|merged` and fall back to `merged` on unknown).
- `getLastSyncedIndexHash`: read `last_synced_index_hash` as a String (no encoding); return null if absent.
- `setLastSyncedIndexHash(null)`: removes the key.
- `getLastIndexSyncAt`: read `last_index_sync_at`; return null if not present.
- `setLastIndexSyncAt(null)`: removes the key.

## Implementation notes

- Reuse the existing `EncryptedSharedPreferences` instance — do not create a second one.
- Define the three new key constants at the top of `PrefsStore` for grep-ability:
  ```kotlin
  private const val KEY_GALLERY_VIEW_MODE = "gallery_view_mode"
  private const val KEY_LAST_SYNCED_INDEX_HASH = "last_synced_index_hash"
  private const val KEY_LAST_INDEX_SYNC_AT = "last_index_sync_at"
  ```
- `getLastIndexSyncAt`: SharedPreferences has no nullable Long. Treat `getLong(key, -1L)` and map `-1L` to `null`. Same for the setter: `setLastIndexSyncAt(null)` removes the key.
- Validation in `setGalleryViewMode`: if the value isn't one of the three known strings, silently normalize to `"merged"`. The UI is the only writer, so this is a defensive guardrail rather than a user-facing error.

## Constraints

- No schema or storage backend changes — same encrypted preferences file as Phase 2.
- Do not log preference values (consistent with credential-handling discipline from MVP).

## Dependencies (by interface)

- None — this task is self-contained.

## Acceptance criteria

- [ ] `getGalleryViewMode()` returns `"merged"` on a fresh install.
- [ ] `setGalleryViewMode("cloud")` followed by `getGalleryViewMode()` returns `"cloud"`.
- [ ] `setGalleryViewMode("bogus")` followed by `getGalleryViewMode()` returns `"merged"`.
- [ ] `getLastSyncedIndexHash()` returns null on a fresh install; setting and reading back works.
- [ ] `setLastSyncedIndexHash(null)` clears the value.
- [ ] `getLastIndexSyncAt()` returns null on a fresh install; setting and reading back returns the same Long.
- [ ] `setLastIndexSyncAt(null)` clears the value.
- [ ] All Phase 2 PrefsStore methods continue to behave identically.

## Out of scope

- A full settings screen — Task 24 only exposes a minimal "Back up index now" button alongside the existing toggles.
- Migration of preference keys — these are all new.
