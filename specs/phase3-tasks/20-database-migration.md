# Task 20 — Database Migration v2 → v3

> Read [`../phase3-implementation-overview.md`](../phase3-implementation-overview.md) first.

## Goal

Extend the `uploads` table with two columns needed for Phase 3 — `local_present` and `cloud_deleted_at` — and expose new DAO methods for the Cloud view, soft-deletion, and index restore.

There is no production user data yet (Phase 2 has not been distributed beyond local development), so the migration drops and recreates the table — same pattern Phase 2 Task 10 used.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/data/UploadRecord.kt` (modify — add fields)
- `app/src/main/java/com/hriyaan/photostorage/data/UploadDatabase.kt` (modify — bump version, recreate table)
- `app/src/main/java/com/hriyaan/photostorage/data/UploadDao.kt` (modify — add methods + constant)

## Schema migration (v2 → v3)

Increment `DB_VERSION` from `2` to `3`. `onUpgrade` drops the table and re-runs `onCreate`:

```kotlin
override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
    db.execSQL("DROP TABLE IF EXISTS $TABLE_UPLOADS")
    onCreate(db)
}
```

This is the same shape as Phase 2's v1 → v2 step. Existing rows are dropped — testers will need to reinstall or accept that their queue resets.

Update `onCreate` to build the v3 schema:

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
    cloud_deleted_at INTEGER
);
CREATE INDEX idx_status ON uploads(status);
CREATE INDEX idx_filename_size ON uploads(filename, size);
CREATE INDEX idx_sha256 ON uploads(sha256);
CREATE INDEX idx_cloud_deleted_at ON uploads(cloud_deleted_at);
CREATE INDEX idx_date_taken ON uploads(date_taken);
```

> SQLite has no native boolean. `local_present` is `INTEGER` 0/1. The DAO reads it as `cursor.getInt(...) == 1`.

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
    val cloudDeletedAt: Long? = null
)
```

Update `Cursor.toRecord()` to read the two new columns. Update `insert` to write them.

## Updated DAO contract

Keep **all existing Phase 2 methods** unchanged. Add:

```kotlin
companion object {
    // ...existing constants...
    const val STATUS_CLOUD_DELETED = "cloud_deleted"
}

/** All uploaded rows that are not soft-deleted. Ordered by dateTaken DESC. */
fun getCloudView(): List<UploadRecord>

/** Cloud-view rows with dateTaken < given timestamp. For Merged view source. */
fun getCloudViewBefore(dateTaken: Long): List<UploadRecord>

/** Max(uploaded_at) over rows where status='uploaded'. Null if none. */
fun getLatestUploadedAt(): Long?

/** Updates local_present for a row. Returns rows affected. */
fun setLocalPresent(id: Long, present: Boolean): Int

/** Sets status='cloud_deleted' and cloud_deleted_at=now. Returns rows affected. */
fun softDeleteCloud(id: Long, now: Long): Int

/** Rows where status='cloud_deleted' and cloud_deleted_at <= cutoff. */
fun getSoftDeletedOlderThan(cutoff: Long): List<UploadRecord>

/** Hard-removes the row. Returns rows affected. */
fun hardDelete(id: Long): Int

/** Wipes the table and inserts all records in a single transaction. Used by index restore. */
fun replaceAll(records: List<UploadRecord>)
```

### Method semantics

- `getCloudView`: `SELECT * FROM uploads WHERE status = 'uploaded' AND cloud_deleted_at IS NULL ORDER BY date_taken DESC`
- `getCloudViewBefore`: `SELECT * FROM uploads WHERE status = 'uploaded' AND cloud_deleted_at IS NULL AND date_taken < ? ORDER BY date_taken DESC`
- `getLatestUploadedAt`: `SELECT MAX(uploaded_at) FROM uploads WHERE status = 'uploaded'`
- `setLocalPresent`: updates only `local_present`. Returns rows affected.
- `softDeleteCloud`: `UPDATE uploads SET status = 'cloud_deleted', cloud_deleted_at = ? WHERE id = ?`
- `getSoftDeletedOlderThan`: `SELECT * FROM uploads WHERE status = 'cloud_deleted' AND cloud_deleted_at <= ?`
- `hardDelete`: `DELETE FROM uploads WHERE id = ?`
- `replaceAll`: `BEGIN; DELETE FROM uploads; INSERT ...; COMMIT`. If the input list is empty, the table is left empty.

## Implementation notes

- `replaceAll` MUST use a single explicit transaction — `db.beginTransaction() ... db.setTransactionSuccessful() ... db.endTransaction()`. Failures inside should rethrow after `endTransaction`.
- New column constants go in `UploadDatabase.Companion`.
- The migration drops and recreates the table. Existing Phase 2 rows are lost — acceptable because there is no production user data yet.
- `getCloudView` excludes `cloud_deleted` rows — they exist only as a 24-hour soft-delete buffer for the cleanup worker (Task 26).

## Constraints

- Drop-and-recreate on v2 → v3 is intentional. Do not attempt to preserve rows.
- Do not change the behavior of existing Phase 2 methods.
- `STATUS_CLOUD_DELETED` is a new status string. Add it as `const val` on the companion.

## Acceptance criteria

- [ ] A v2 (or v1) database opens, `onUpgrade` drops the table, and the resulting v3 schema has 15 columns including `local_present` and `cloud_deleted_at`.
- [ ] A fresh install (no prior DB) creates the v3 schema directly via `onCreate`.
- [ ] `getCloudView` returns only rows with `status='uploaded'` and `cloud_deleted_at IS NULL`.
- [ ] `softDeleteCloud` correctly transitions a row and `getCloudView` excludes it on the next call.
- [ ] `replaceAll` wipes and repopulates the table in a single transaction; a failure mid-way leaves the prior state.
- [ ] `getLatestUploadedAt` returns null on an empty table and the correct max otherwise.
- [ ] Code compiles with no new dependencies.

## Out of scope

- Preserving rows across the v2 → v3 migration.
- Backfilling `local_present` from the actual MediaStore — defaults are sufficient; reconciliation is Task 28's job.
- Schema changes to support multi-device sync.
