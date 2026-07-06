# Task 38 — Cost Dashboard

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

Surface a read-only screen that tells the user what their backup is costing on B2: total stored bytes, photo and video counts, estimated monthly storage cost, and an optional egress estimate. Pull aggregates from the `uploads` table; do not call B2 APIs for pricing. All math happens in `B2Pricing` and `CostDashboardService`.

The dashboard is reachable from Task 41's Settings screen and from the gallery's status header (a small "View costs" link). No write actions — this task is purely informational.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/cost/B2Pricing.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/cost/CostDashboardService.kt` (new)
- `app/src/main/java/com/hriyaan/photostorage/ui/CostDashboardActivity.kt` (new)
- `app/src/main/res/layout/activity_cost_dashboard.xml` (new)
- `app/src/main/res/values/strings.xml` (modify — add this task's keys only)

## `B2Pricing`

A single Kotlin object holding the constants. Changing a price is a one-diff change.

```kotlin
object B2Pricing {
    /** USD per gigabyte stored per month. */
    const val STORAGE_PER_GB_PER_MONTH_USD: Double = 0.006

    /** USD per gigabyte of egress (B2 download). */
    const val EGRESS_PER_GB_USD: Double = 0.010

    /** Date of last manual price review. Surfaced in the UI footnote. */
    const val LAST_UPDATED: String = "2026-05-17"

    fun monthlyStorageCostUsd(bytes: Long): Double {
        val gb = bytes / 1_000_000_000.0
        return gb * STORAGE_PER_GB_PER_MONTH_USD
    }

    fun egressCostUsd(bytes: Long): Double {
        val gb = bytes / 1_000_000_000.0
        return gb * EGRESS_PER_GB_USD
    }
}
```

> The constants follow B2's published per-GB-per-month / per-GB rates as of `LAST_UPDATED`. Keep them as Doubles for explicit readability — money formatting happens at display time.

## `CostDashboardService`

```kotlin
class CostDashboardService(
    private val uploadDao: UploadDao,
    private val prefsStore: PrefsStore
) {
    /** Computes a fresh snapshot. Caller caches the result for 5 minutes. */
    suspend fun computeSnapshot(): CostSnapshot = withContext(Dispatchers.IO) {
        val photoCount = uploadDao.countByMediaType(UploadDao.MEDIA_TYPE_PHOTO)
        val videoCount = uploadDao.countByMediaType(UploadDao.MEDIA_TYPE_VIDEO)
        val photoBytes = uploadDao.sumSizeByMediaType(UploadDao.MEDIA_TYPE_PHOTO)
        val videoBytes = uploadDao.sumSizeByMediaType(UploadDao.MEDIA_TYPE_VIDEO)
        val totalBytes = photoBytes + videoBytes

        val pending = uploadDao.getPending() // existing Phase 2 method
        val pendingCount = pending.size
        val pendingBytes = pending.sumOf { it.size }

        val permanentlyFailedCount = uploadDao.countByStatus(UploadDao.STATUS_PERMANENTLY_FAILED)

        val egressBytes = prefsStore.getEgressBytesMonth()
        val byYear = computeByYear(uploadDao)

        CostSnapshot(
            photoCount = photoCount,
            videoCount = videoCount,
            totalBytes = totalBytes,
            pendingCount = pendingCount,
            pendingBytes = pendingBytes,
            permanentlyFailedCount = permanentlyFailedCount,
            monthlyStorageCostUsd = B2Pricing.monthlyStorageCostUsd(totalBytes),
            monthlyEgressCostUsd = if (egressBytes > 0L) B2Pricing.egressCostUsd(egressBytes) else null,
            byYear = byYear
        )
    }

    private fun computeByYear(dao: UploadDao): List<YearBreakdown> {
        val rows = dao.getCloudView() // Phase 3 method — all uploaded, not cloud-deleted
        return rows
            .groupBy { yearOf(it.dateTaken) }
            .map { (year, rs) ->
                YearBreakdown(
                    year = year,
                    bytes = rs.sumOf { it.size },
                    photoCount = rs.count { it.mediaType == "photo" },
                    videoCount = rs.count { it.mediaType == "video" }
                )
            }
            .sortedByDescending { it.year }
            .take(5)
    }

    private fun yearOf(dateTakenMs: Long): Int {
        val cal = Calendar.getInstance().apply { timeInMillis = dateTakenMs }
        return cal.get(Calendar.YEAR)
    }
}

data class CostSnapshot(
    val photoCount: Int,
    val videoCount: Int,
    val totalBytes: Long,
    val pendingCount: Int,
    val pendingBytes: Long,
    val permanentlyFailedCount: Int,
    val monthlyStorageCostUsd: Double,
    val monthlyEgressCostUsd: Double?, // null when no egress recorded
    val byYear: List<YearBreakdown>
)

data class YearBreakdown(val year: Int, val bytes: Long, val photoCount: Int, val videoCount: Int)
```

### Required Phase 2/3 DAO methods

`computeSnapshot` uses several existing DAO methods:
- `getPending()` (Phase 2) — pending queue rows
- `countByStatus(status)` (Phase 2) — counts by status string
- `getCloudView()` (Phase 3 Task 20) — uploaded rows not cloud-deleted

If any of these methods do not exist exactly under these names, use the closest equivalent and adapt the snapshot computation. No new DAO methods should be added here.

### Egress instrumentation

The egress counter is written by the Phase 3 thumbnail cache fetcher (`B2ThumbnailFetcher`). To avoid editing Task 29's code in Phase 4, this task adds a small wrapper at the dashboard's first construction:

```kotlin
class CostDashboardService(...) {
    init {
        // Wrap the existing thumbnail fetcher with an egress recorder if not already wrapped.
        // PhotoBackupApp.thumbnailCacheFactory exposes a setter for this; if not, this no-ops.
    }
}
```

If wrapping is not feasible (because `B2ThumbnailFetcher` is final and not extension-friendly), simply leave the counter at 0 and the UI hides the egress section per PRD §5. The integration owner (Task 42) can later expose a hook in `PhotoBackupApp` if the user wants this surface populated.

In practice, the minimal egress wrapper looks like this in `PhotoBackupApp.onCreate` (added in Task 42, mentioned here for context):

```kotlin
thumbnailCacheFactory = ThumbnailCacheFactory(
    context = this,
    s3Uploader = s3Uploader,
    prefsStore = prefsStore,
    egressRecorder = { bytes -> prefsStore.setEgressBytesMonth(prefsStore.getEgressBytesMonth() + bytes) }
)
```

The `egressRecorder` parameter is a new constructor argument on `ThumbnailCacheFactory` — its addition is a Phase 3 contract extension. Task 38 documents the need; the actual edit to `ThumbnailCacheFactory` is made by Task 38 here (one line). Document this as the only file outside Task 38's nominal scope it touches.

Add to Task 38's ownership: `app/src/main/java/com/hriyaan/photostorage/thumbnail/ThumbnailCacheFactory.kt` (1-line change to add an optional `egressRecorder: ((Long) -> Unit)? = null` constructor parameter, invoked from the fetcher on successful B2 fetch with the response body length).

## `CostDashboardActivity`

A simple linear layout. Compose is acceptable too — match the project's existing precedent. Below is the View-based sketch.

### Layout

```xml
<ScrollView ...>
  <LinearLayout android:orientation="vertical" android:padding="16dp">

    <!-- At a glance -->
    <TextView android:id="@+id/header_total" .../>
    <TextView android:id="@+id/header_cost"  .../>

    <View android:layout_height="1dp" android:background="?android:attr/dividerVertical" .../>

    <!-- By media type -->
    <TextView android:text="@string/cost_by_media_type" .../>
    <TextView android:id="@+id/row_photos" .../>
    <TextView android:id="@+id/row_videos" .../>

    <View android:layout_height="1dp" .../>

    <!-- Pending / failed -->
    <TextView android:text="@string/cost_pending_header" .../>
    <TextView android:id="@+id/row_pending" .../>
    <TextView android:id="@+id/row_failed" .../>

    <View android:layout_height="1dp" .../>

    <!-- By year (optional rendering — hide if list is empty) -->
    <TextView android:id="@+id/header_by_year" android:text="@string/cost_by_year_header" .../>
    <LinearLayout android:id="@+id/by_year_container" android:orientation="vertical" .../>

    <View android:layout_height="1dp" .../>

    <!-- Egress (hidden unless monthlyEgressCostUsd != null) -->
    <TextView android:id="@+id/row_egress" .../>

    <!-- Footnote -->
    <TextView android:id="@+id/footnote" android:textSize="12sp" android:textColor="?android:attr/textColorSecondary" .../>

  </LinearLayout>
</ScrollView>
```

### Behavior

```kotlin
class CostDashboardActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_cost_dashboard)
        val app = application as PhotoBackupApp
        val service = CostDashboardService(app.uploadDatabase.dao, app.prefsStore)
        lifecycleScope.launch {
            val snap = service.computeSnapshot()
            render(snap)
        }
    }

    private fun render(snap: CostSnapshot) {
        findViewById<TextView>(R.id.header_total).text = getString(
            R.string.cost_header_total,
            (snap.photoCount + snap.videoCount).format(),
            formatBytes(snap.totalBytes)
        )
        findViewById<TextView>(R.id.header_cost).text = getString(
            R.string.cost_header_cost,
            formatUsd(snap.monthlyStorageCostUsd)
        )
        findViewById<TextView>(R.id.row_photos).text = getString(
            R.string.cost_row_photos,
            snap.photoCount.format(),
            formatBytes(snap.totalBytes - byteSizeOfVideos(snap)),
            formatUsd(B2Pricing.monthlyStorageCostUsd(snap.totalBytes - byteSizeOfVideos(snap)))
        )
        // ... similar for videos, pending, failed, by-year, egress
        findViewById<TextView>(R.id.footnote).text = getString(
            R.string.cost_footnote, B2Pricing.LAST_UPDATED
        )
    }
}
```

### Caching

The snapshot is fast to compute (single-digit ms even at 10K rows), but the activity caches the result for 5 minutes per PRD §5. Cache key: nothing — just a `WeakReference` in `PhotoBackupApp` or an in-process `Pair<Long, CostSnapshot>`. On resume after 5 minutes, recompute.

Implementation can skip the cache entirely if the simple compute-on-open behavior is fast enough (it is, at Phase 4 scale) — the cache exists for forward compat. Tests should still pass without the cache.

## Strings

```xml
<string name="cost_dashboard_title">Storage &amp; cost</string>
<string name="cost_header_total">Total backed up: %1$s items · %2$s</string>
<string name="cost_header_cost">Estimated monthly cost: ≈ %1$s</string>
<string name="cost_by_media_type">By media type</string>
<string name="cost_row_photos">Photos: %1$s · %2$s · %3$s / month</string>
<string name="cost_row_videos">Videos: %1$s · %2$s · %3$s / month</string>
<string name="cost_pending_header">Pending and failed</string>
<string name="cost_pending">Waiting to upload: %1$s · %2$s</string>
<string name="cost_failed">Failed (permanent): %1$s</string>
<string name="cost_by_year_header">By year</string>
<string name="cost_year_row">%1$d: %2$s · %3$s photos, %4$s videos</string>
<string name="cost_egress">Thumbnail viewing this month: ≈ %1$s · ≈ %2$s</string>
<string name="cost_footnote">B2 pricing as of %1$s. These numbers are estimates.</string>
```

## Implementation notes

- `formatBytes` follows IEC binary (KiB, MiB, GiB) for display consistency with Android, but the *math* in `B2Pricing` uses decimal GB (B2's billing convention is `10^9`-byte gigabytes). Make this explicit in code comments at the conversion points — readers should not have to compare numbers and wonder why.
- USD formatting: `"$%.2f".format(value)` is enough for Phase 4 (no localization in this task).
- The "By year" section is optional per PRD (P3 within Phase 4). Render if `byYear.isNotEmpty()`, otherwise hide the header and container with `View.GONE`.
- Egress row: show only when `monthlyEgressCostUsd != null` (the snapshot already nulls it out when there is nothing to display).
- Do not write to `PrefsStore.EgressBytesMonth` from this task except through the `egressRecorder` hook in `ThumbnailCacheFactory`. The dashboard never resets the counter manually — the month flip is handled inside `PrefsStore.setEgressBytesMonth` (see Task 32).

## Constraints

- Read-only. The dashboard does not modify `uploads`, `share_links`, or any preference except via the egress recorder hook.
- No network calls. All math is local.
- Constants in `B2Pricing` are the single source of truth for pricing. Do not hard-code pricing elsewhere.
- The activity must remain responsive during snapshot compute (use `lifecycleScope` + `withContext(Dispatchers.IO)`).

## Dependencies (by interface)

- `UploadDao` (Task 31) — `countByMediaType`, `sumSizeByMediaType`, `getCloudView` (Phase 3), `getPending` (Phase 2), `countByStatus` (Phase 2)
- `PrefsStore` (Task 32) — `getEgressBytesMonth`, `setEgressBytesMonth`
- `ThumbnailCacheFactory` (Phase 3 Task 29 — modified here to accept an optional `egressRecorder`)

## Acceptance criteria

- [ ] `B2Pricing.monthlyStorageCostUsd(1_000_000_000L)` returns `0.006` (1 GB at $0.006).
- [ ] `B2Pricing.egressCostUsd(1_000_000_000L)` returns `0.010`.
- [ ] `CostDashboardService.computeSnapshot()` on an empty DB returns counts and bytes of 0 and `monthlyEgressCostUsd = null`.
- [ ] Adding rows to `uploads` (via tests or manual upload) increases the snapshot's `totalBytes` and `photoCount` / `videoCount`.
- [ ] The activity renders all required sections and hides the egress section when egress is 0.
- [ ] The "By year" section shows at most 5 years, newest first, when populated.
- [ ] The footnote includes `B2Pricing.LAST_UPDATED`.
- [ ] Scrolling through Cloud view (Phase 3) increments the egress counter via the `ThumbnailCacheFactory` hook (verify by reading `EgressBytesMonth` before and after).

## Out of scope

- Live B2 pricing via API call.
- Cost projections (e.g., "at current growth, you will hit 100 GB in 6 months").
- CSV export of the snapshot.
- Per-month historical chart.
- Bandwidth throttling settings.
- Storage tiering recommendations.
