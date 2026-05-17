# Task 02 — Data Layer (SQLite)

> Read [`../mvp-implementation-overview.md`](../mvp-implementation-overview.md) first.

## Goal

Provide a tiny SQLite layer for tracking uploaded photos. Single table, raw `SQLiteOpenHelper` (no Room).

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/data/UploadRecord.kt`
- `app/src/main/java/com/hriyaan/photostorage/data/UploadDatabase.kt`
- `app/src/main/java/com/hriyaan/photostorage/data/UploadDao.kt`
- `app/src/test/java/com/hriyaan/photostorage/data/UploadDaoTest.kt` (Robolectric or pure JVM with in-memory SQLite — optional but recommended)

## Schema

Single database file `uploads.db`, version `1`:

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
    uploaded_at INTEGER
);

CREATE INDEX idx_status ON uploads(status);
CREATE INDEX idx_filename_size ON uploads(filename, size);
```

Status values are exactly: `pending`, `uploading`, `uploaded`, `failed`. Define them as constants on the DAO companion.

## Public contract

```kotlin
package com.hriyaan.photostorage.data

data class UploadRecord(
    val id: Long = 0L,
    val localUri: String,
    val filename: String,
    val size: Long,
    val dateTaken: Long,
    val photoB2Path: String?,
    val thumbnailB2Path: String?,
    val status: String,
    val uploadedAt: Long?
)

class UploadDatabase(context: Context) {
    val dao: UploadDao
    fun close()
}

class UploadDao(/* package-internal ctor — exposed via UploadDatabase.dao */) {
    companion object {
        const val STATUS_PENDING = "pending"
        const val STATUS_UPLOADING = "uploading"
        const val STATUS_UPLOADED = "uploaded"
        const val STATUS_FAILED = "failed"
    }

    fun insert(record: UploadRecord): Long
    fun updateStatus(id: Long, status: String, uploadedAt: Long? = null): Int
    fun setUploadedPaths(
        id: Long,
        photoPath: String,
        thumbnailPath: String,
        uploadedAt: Long
    ): Int
    fun findByFilenameAndSize(filename: String, size: Long): UploadRecord?
    fun getAll(): List<UploadRecord>
}
```

### Method semantics

- `insert(record)` returns the new row id. Always sets `status` from `record.status` (caller decides; usually `STATUS_PENDING` or `STATUS_UPLOADING`).
- `updateStatus` updates only `status` and (if non-null) `uploaded_at`. Returns rows affected.
- `setUploadedPaths` sets `photo_b2_path`, `thumbnail_b2_path`, `uploaded_at`, and `status = 'uploaded'` atomically.
- `findByFilenameAndSize` is the duplicate check used by the gallery. Returns the most recently inserted match, or null.
- `getAll` returns rows ordered by `date_taken DESC`.

## Implementation notes

- Subclass `SQLiteOpenHelper`; expose `dao` lazily.
- All DAO methods are blocking — callers wrap them in `withContext(Dispatchers.IO)`.
- Use `ContentValues` for inserts/updates. No raw string concatenation in `WHERE` clauses — bind args.
- Do **not** add a singleton or `getInstance()` — the Application class (Task 09) will hold the single instance.

## Acceptance criteria

- [ ] `UploadDatabase(context).dao.insert(...)` round-trips a record via `getAll()`.
- [ ] `findByFilenameAndSize` returns null when no match exists, returns the row when one does.
- [ ] `setUploadedPaths` atomically updates all four fields (paths + timestamp + status). Verify with `getAll`.
- [ ] Schema creates both indices (`idx_status`, `idx_filename_size`) — verify via `sqlite_master`.
- [ ] Code compiles against Task 01's `app/build.gradle.kts` with no extra dependencies.

## Out of scope

- No migrations (this is v1, MVP).
- No Flow / LiveData. Plain blocking calls.
- No SHA-256 column (post-MVP).
