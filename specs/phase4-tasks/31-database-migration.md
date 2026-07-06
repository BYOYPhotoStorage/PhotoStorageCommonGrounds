# Task 31 — Database Migration v3 → v4

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Extend the `uploads` table with four columns needed for Phase 4 — `media_type`, `original_path_b2`, `pending_local_delete`, `compressed` — and add a new `share_links` table for the Sharing feature. Expose new DAO methods for media-type queries, local-delete batching, and share-link persistence.

No production user data exists yet (Phases 2 and 3 have not been distributed), so the migration drops and recreates the `uploads` table. `share_links` is brand new.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/data/UploadRecord.kt` (modify — add fields)
- `app/src/main/java/com/hriyaan/photostorage/data/UploadDatabase.kt` (modify — bump version, recreate table, add `share_links`)
- `app/src/main/java/com/hriyaan/photostorage/data/UploadDao.kt` (modify — add methods + constants)
- `app/src/main/java/com/hriyaan/photostorage/data/ShareLinkRecord.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/data/ShareLinkDao.kt` (new)

## Schema migration (v3 → v4)

Increment `DB_VERSION` from `3` to `4`. `onUpgrade` drops `uploads` and re-runs `onCreate`. `share_links` is created in `onCreate` and the equivalent CREATE TABLE runs in `onUpgrade` so v3 → v4 installs gain it too.

```kotlin
override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
    db.execSQL("DROP TABLE IF EXISTS $TABLE_UPLOADS")
    onCreate(db)
}
```

Updated `onCreate`:

```sql
CREATE TABLE uploads (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    local_uri TEXT NOT NULL,
    filename TEXT NOT NULL,
    size INTEGER NOT NULL,
    date_taken INTEGER NOT NULL,
    photo_b2_path TEXT,
    thumbnail_b2_path TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    uploaded_at INTEGER,
    retry_count INTEGER NOT NULL DEFAULT 0,
    next_retry_at INTEGER,
    sha256 TEXT,
    created_at INTEGER NOT NULL DEFAULT 0,
    local_present INTEGER NOT NULL DEFAULT 1,
    cloud_deleted_at INTEGER,
    media_type TEXT NOT NULL DEFAULT 'photo',
    original_path_b2 TEXT,
    pending_local_delete INTEGER NOT NULL DEFAULT 0,
    compressed INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_status ON uploads(status);
CREATE INDEX idx_filename_size ON uploads(filename, size);
CREATE INDEX idx_sha256 ON uploads(sha256);
CREATE INDEX idx_cloud_deleted_at ON uploads(cloud_deleted_at);
CREATE INDEX idx_date_taken ON uploads(date_taken);
CREATE INDEX idx_media_type ON uploads(media_type);
CREATE INDEX idx_pending_local_delete ON uploads(pending_local_delete);

CREATE TABLE share_links (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    upload_id INTEGER NOT NULL,
    photo_b2_path TEXT NOT NULL,
    url TEXT NOT NULL,
    created_at INTEGER NOT NULL,
    expires_at INTEGER NOT NULL
);
CREATE INDEX idx_share_links_expires_at ON share_links(expires_at);
CREATE INDEX idx_share_links_upload_id ON share_links(upload_id);
```

> Booleans remain `INTEGER` 0/1, same convention as Phase 3.

## Updated data class

```kotlin
data class UploadRecord(
    val id: Long = 0L,
    val localUri: String,
    val filename: String,
    val size: Long,
    val dateTaken: Long,
    val photoB2Path: String?,
    val thumbnailB2Path: String?,
    val status: String,
    val uploadedAt: Long?,
    val retryCount: Int = 0,
    val nextRetryAt: Long? = null,
    val sha256: String? = null,
    val createdAt: Long = System.currentTimeMillis(),
    val localPresent: Boolean = true,
    val cloudDeletedAt: Long? = null,
    val mediaType: String = "photo",
    val originalPathB2: String? = null,
    val pendingLocalDelete: Boolean = false,
    val compressed: Boolean = false
)
```

Update `Cursor.toRecord()` to read the four new columns. Update `insert` to write them.

## New `ShareLinkRecord`

```kotlin
data class ShareLinkRecord(
    val id: Long = 0L,
    val uploadId: Long,
    val photoB2Path: String,
    val url: String,
    val createdAt: Long,
    val expiresAt: Long
)
```

## Updated `UploadDao` contract

Keep **all existing Phase 2 + Phase 3 methods** unchanged. Add:

```kotlin
companion object {
    // ...existing constants...
    const val MEDIA_TYPE_PHOTO = "photo"
    const val MEDIA_TYPE_VIDEO = "video"
}

/** All rows of the given media type. Ordered by dateTaken DESC. */
fun getByMediaType(mediaType: String): List<UploadRecord>

/** Count of rows in the table with the given media type and status='uploaded'. */
fun countByMediaType(mediaType: String): Int

/** SUM(size) for uploaded rows of the given media type. 0 if none. */
fun sumSizeByMediaType(mediaType: String): Long

/** Sets pending_local_delete=1 for each id. Returns rows affected. */
fun markPendingLocalDelete(ids: List<Long>): Int

/** Rows with pending_local_delete=1 AND status='uploaded'. */
fun getPendingLocalDelete(): List<UploadRecord>

/** Uploaded rows with uploaded_at <= cutoff AND pending_local_delete=0 AND local_present=1.
 *  Used by after_days strategy. Ordered by uploaded_at ASC (oldest first). */
fun getEligibleForLocalDelete(olderThanUploadedAt: Long): List<UploadRecord>

/** Oldest uploaded rows with local_present=1 AND pending_local_delete=0. Used by after_count. */
fun getOldestUploaded(limit: Int): List<UploadRecord>

/** Clears pending_local_delete for one row. Returns rows affected. */
fun clearPendingLocalDelete(id: Long): Int

/** Counts uploads with uploaded_at > timestamp. Used by after_count. */
fun countUploadsSince(timestamp: Long): Int
```

### Method semantics

- `getByMediaType`: `SELECT * FROM uploads WHERE media_type = ? ORDER BY date_taken DESC`
- `countByMediaType`: `SELECT COUNT(*) FROM uploads WHERE media_type = ? AND status = 'uploaded'`
- `sumSizeByMediaType`: `SELECT COALESCE(SUM(size), 0) FROM uploads WHERE media_type = ? AND status = 'uploaded'`
- `markPendingLocalDelete`: bulk `UPDATE uploads SET pending_local_delete = 1 WHERE id IN (...)`. Use parameter binding; chunk to 500 ids per statement if the list is huge.
- `getPendingLocalDelete`: `SELECT * FROM uploads WHERE pending_local_delete = 1 AND status = 'uploaded'`
- `getEligibleForLocalDelete`: `SELECT * FROM uploads WHERE status = 'uploaded' AND uploaded_at <= ? AND pending_local_delete = 0 AND local_present = 1 ORDER BY uploaded_at ASC`
- `getOldestUploaded`: `SELECT * FROM uploads WHERE status = 'uploaded' AND local_present = 1 AND pending_local_delete = 0 ORDER BY uploaded_at ASC LIMIT ?`
- `clearPendingLocalDelete`: `UPDATE uploads SET pending_local_delete = 0 WHERE id = ?`
- `countUploadsSince`: `SELECT COUNT(*) FROM uploads WHERE uploaded_at > ?`

## `ShareLinkDao`

```kotlin
class ShareLinkDao(private val helper: UploadDatabase) {
    fun insert(record: ShareLinkRecord): Long
    fun getActive(now: Long): List<ShareLinkRecord>
    fun getOlderThan(cutoff: Long): List<ShareLinkRecord>
    fun deleteOlderThan(cutoff: Long): Int
}
```

- `insert`: `INSERT INTO share_links (upload_id, photo_b2_path, url, created_at, expires_at) VALUES (?, ?, ?, ?, ?)`. Returns the auto-generated id.
- `getActive`: `SELECT * FROM share_links WHERE expires_at + 86400000 > ? ORDER BY created_at DESC` — keeps rows visible for 24h past expiry per PRD §6.
- `getOlderThan`: `SELECT * FROM share_links WHERE expires_at < ?`
- `deleteOlderThan`: `DELETE FROM share_links WHERE expires_at < ?` — used only by a future cleanup pass; Phase 4 does not call it automatically.

## Implementation notes

- All DAOs use the existing `UploadDatabase` `SQLiteOpenHelper` instance. Construct `ShareLinkDao` with the same helper.
- `markPendingLocalDelete` MUST run inside a single transaction when the id list is large (`db.beginTransaction()` ... `endTransaction()`).
- The `media_type` column has only two valid values today (`photo` / `video`) but is `TEXT` for forward compatibility (e.g., `live_photo`, `motion`).
- `pending_local_delete` is independent of `local_present`. A row is *eligible* for the delete prompt only when both `pending_local_delete = 1` and `local_present = 1` — if the user already removed the file via Local view, the row should be skipped.
- New column constants go in `UploadDatabase.Companion` alongside Phase 3's.

## Constraints

- Drop-and-recreate on v3 → v4 is intentional. Do not attempt to preserve rows.
- Do not change the behavior of existing Phase 2 / Phase 3 methods.
- `MEDIA_TYPE_PHOTO` / `MEDIA_TYPE_VIDEO` are the only acceptable values written to `media_type` by app code. Other values may exist in restored indices (forward compat) — read them through but never produce them.
- `share_links.url` is stored verbatim — never logged.

## Dependencies (by interface)

- None — this task is the Phase 4 foundation.

## Acceptance criteria

- [ ] A v3 database opens, `onUpgrade` drops `uploads` and creates the v4 schema with 19 columns including the four new ones.
- [ ] A v3 database that lacks `share_links` gets it created via `onUpgrade`.
- [ ] A fresh install (no prior DB) creates v4 schema directly via `onCreate`, including `share_links`.
- [ ] `getByMediaType("photo")` and `getByMediaType("video")` partition the table correctly.
- [ ] `markPendingLocalDelete(ids)` sets the flag for every id in one transaction.
- [ ] `getPendingLocalDelete()` returns only rows with the flag set AND `status='uploaded'`.
- [ ] `getEligibleForLocalDelete(cutoff)` excludes rows already marked, soft-deleted, or with `local_present = 0`.
- [ ] `ShareLinkDao.insert` followed by `getActive(now)` returns the new row.
- [ ] `ShareLinkDao.getActive` keeps a row visible for 24h after expiry, then drops it.
- [ ] Code compiles with no new dependencies.

## Out of scope

- Preserving rows across the v3 → v4 migration.
- Backfilling `media_type` for legacy rows — they all default to `photo`, which is correct for the current cohort (Phase 3 only handled photos).
- Schema changes for multi-bucket or multi-account support.
- A periodic cleanup task for old share-link rows (a `deleteOlderThan` call is exposed but not scheduled in Phase 4).
