# Task 10 — Database Migration & Queue Schema

> Read [`../phase2-implementation-overview.md`](../phase2-implementation-overview.md) first.

## Goal

Migrate the MVP `uploads` table from schema v1 to v2, adding columns needed for queue tracking, retry logic, and SHA-256 duplicate detection. Update the data class and DAO to expose queue-oriented queries.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/data/UploadRecord.kt` (modify — add fields)
- `app/src/main/java/com/hriyaan/photostorage/data/UploadDatabase.kt` (modify — add migration)
- `app/src/main/java/com/hriyaan/photostorage/data/UploadDao.kt` (modify — add queue methods)

## Schema migration (v1 → v2)

Increment `DB_VERSION` from `1` to `2`.

There is no production user data yet (internal testing only), so the migration can simply drop and recreate the table:

```sql
DROP TABLE IF EXISTS uploads;
```

Then re-run `onCreate` to build the v2 schema with all columns.

In `onUpgrade(db, 1, 2)`:

```kotlin
override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
    db.execSQL("DROP TABLE IF EXISTS $TABLE_UPLOADS")
    onCreate(db)
}
```

New `onCreate` schema:

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
    created_at INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_status ON uploads(status);
CREATE INDEX idx_filename_size ON uploads(filename, size);
CREATE INDEX idx_sha256 ON uploads(sha256);
```

> `created_at` defaulting to `0` is fine for the migration path. New inserts should set it to `System.currentTimeMillis()`.

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
    val createdAt: Long = System.currentTimeMillis()
)
```

## Updated DAO contract

Keep **all existing MVP methods** exactly as they are. Add the following:

```kotlin
class UploadDao internal constructor(private val helper: SQLiteOpenHelper) {
    // ... existing MVP methods (insert, updateStatus, setUploadedPaths, findByFilenameAndSize, getAll) ...

    companion object {
        // ... existing status constants ...
        const val STATUS_PERMANENTLY_FAILED = "permanently_failed"
    }

    /** Returns pending items, ordered oldest first. */
    fun getPendingQueue(): List<UploadRecord>

    /** Returns failed items whose retry time has come and haven't exceeded max retries. */
    fun getFailedRetryable(now: Long = System.currentTimeMillis()): List<UploadRecord>

    /** Atomically updates retry_count and next_retry_at. */
    fun updateRetry(id: Long, retryCount: Int, nextRetryAt: Long?): Int

    /** Finds a record by SHA-256 hash. Returns null if not found. */
    fun findBySha256(sha256: String): UploadRecord?

    /** Stores the computed SHA-256 for a record. */
    fun updateSha256(id: Long, sha256: String): Int
}
```

### Method semantics

- `getPendingQueue`: `SELECT * FROM uploads WHERE status = 'pending' ORDER BY created_at ASC`
- `getFailedRetryable`: `SELECT * FROM uploads WHERE status = 'failed' AND retry_count < 5 AND next_retry_at <= ? ORDER BY next_retry_at ASC`
- `updateRetry`: updates only `retry_count` and `next_retry_at`, returns rows affected
- `findBySha256`: exact match on `sha256` column. Returns most recently inserted match if duplicates exist (use `ORDER BY id DESC LIMIT 1`).
- `updateSha256`: sets `sha256` for the given row id

## Implementation notes

- `UploadDatabase.Helper.onUpgrade` drops the old table and recreates it with the v2 schema.
- New column constants must be added to `UploadDatabase.Companion`.
- `Cursor.toRecord()` must read the new columns. Since the table is recreated, all columns are always present; `getColumnIndexOrThrow` is safe.

## Constraints

- The table is dropped and recreated on upgrade — acceptable because there is no production user data yet.
- Do not change the behavior of existing MVP methods.
- `STATUS_PERMANENTLY_FAILED` is a new status string. Ensure it is a `const val` on the DAO companion.

## Acceptance criteria

- [ ] Opening a v1 database triggers `onUpgrade`, drops the old table, and creates the v2 schema with all columns.
- [ ] `insert` with the new `UploadRecord` defaults populates all columns correctly.
- [ ] `getPendingQueue` returns only records with `status = 'pending'`, ordered by `created_at ASC`.
- [ ] `getFailedRetryable(now)` returns records with `status = 'failed'`, `retry_count < 5`, and `next_retry_at <= now`.
- [ ] `findBySha256` returns the matching record, or null when no match.
- [ ] `updateRetry` and `updateSha256` return 1 when the row exists, 0 otherwise.
- [ ] Code compiles with no new dependencies.

## Out of scope

- No data backfill for `created_at` on migrated rows (0 is acceptable).
- No removal of old indices.
