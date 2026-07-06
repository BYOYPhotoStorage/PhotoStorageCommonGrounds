# Task 41 — Settings Screen

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md) first.

## Goal

A single `SettingsActivity` that consolidates every user-facing toggle introduced across Phases 2–4. Replaces the minimal Phase 2 Wi-Fi toggle and the Phase 3 "Back up index now" affordance. The activity is the only Phase 4-introduced UI that the user reaches from the gallery overflow menu day-to-day; everything else (cost dashboard, active share links, recovery) is launched from inside Settings.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/ui/SettingsActivity.kt` (new)
- `app/src/main/res/values/strings.xml` (modify — add this task's keys only)

## Layout style

Compose-based, matching the project's existing precedent for non-trivial screens. A `LazyColumn` with section headers and rows. Do NOT use AndroidX Preference library — its theming clashes and the project does not depend on it.

```kotlin
class SettingsActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { MaterialTheme { SettingsScreen() } }
    }
}

@Composable
private fun SettingsScreen() {
    val app = LocalContext.current.applicationContext as PhotoBackupApp
    val prefs = app.prefsStore
    val state = remember { mutableStateOf(SettingsState.from(prefs)) }

    Scaffold(topBar = { SettingsTopBar() }) { padding ->
        LazyColumn(modifier = Modifier.padding(padding)) {
            backupSection(state, prefs)
            videosSection(state, prefs)
            storageManagementSection(state, prefs)
            storageAndCostSection()
            backupIndexSection(prefs)
            sharingSection()
            accountSection()
            aboutSection()
        }
    }
}
```

The `SettingsState` data class mirrors every preference the screen surfaces. Update functions write through `PrefsStore`. Each row is implemented as a small composable in the same file — keep the file flat, do not split into a dozen sub-files.

## Sections (in display order)

### 1. Backup

| Row | Type | Pref | Side effects |
|-----|------|------|--------------|
| Auto-upload | Switch | `auto_upload_enabled` (Phase 2) | Start / stop `UploadForegroundService` |
| Backup timing | Radio (in sub-screen or expanded inline) | `upload_mode` (Task 32) | None — next worker tick picks it up |
| Wi-Fi only | Switch | `wifi_only_uploads` (Phase 2) | Re-evaluates the upload mode gate (Task 36) |

Auto-upload OFF disables all rows below (gray them out). The user can still scroll and read the controls.

#### Backup-timing UI

A row labeled "Backup timing" with the current value as the trailing text ("Immediate" / "Scheduled (nightly)" / "Hybrid"). Tap → bottom sheet with three radio options + one-line descriptions:

- **Immediate** — "Upload as photos are taken."
- **Scheduled (nightly)** — "Upload during the nightly window (02:00–04:00 local)."
- **Hybrid** — "Upload on Wi-Fi; defer mobile data to the nightly window."

Single-selection. Tap → write to `PrefsStore.setUploadMode(...)` and dismiss.

### 2. Videos

| Row | Type | Pref |
|-----|------|------|
| Include videos | Switch | `videos_enabled` (Task 32) |
| Quality mode | Radio sub-screen | `video_quality_mode` (Task 32) |
| Duration threshold (minutes) | Slider, conditional | `video_duration_threshold_minutes` (Task 32) |
| Target resolution | Radio (720p / 1080p) | `video_target_resolution` (Task 32) |

Toggling "Include videos" ON requests `READ_MEDIA_VIDEO` (re-prompt allowed via Android settings if previously denied). If the permission ends up denied, revert the switch to OFF and toast: "Videos disabled — permission denied."

The duration threshold slider is only visible when `video_quality_mode = duration_based`. Slider range: 1–60 minutes, step 1.

Toggling `videos_enabled` ON also triggers the foreground service to register the video observer (handled inside `UploadForegroundService` via the `PrefsStore` change listener added in Task 34) — no extra wiring needed here.

### 3. Storage management

| Row | Type | Pref |
|-----|------|------|
| Auto-delete strategy | Radio sub-screen | `local_delete_strategy` (Task 32) |
| Days after upload | Number stepper, conditional | `local_delete_days` (Task 32) |
| New photos before deletion | Number stepper, conditional | `local_delete_count` (Task 32) |

Strategy choices: "Never", "Immediately after upload", "After X days", "After N new photos". Tapping reveals an inline help line under the row explaining what happens.

The conditional fields are only visible for the matching strategy.

A trailing "Review pending deletions now" button launches `LocalDeleteScheduler.runNow(context)` (Task 37). Useful for testing and for users who want to action a queued batch on demand.

### 4. Storage & cost

A single row with chevron, opens `CostDashboardActivity` (Task 38). No state.

### 5. Backup index

| Row | Type | Notes |
|-----|------|-------|
| Last backed up | Read-only text | Reads `last_index_sync_at` (Phase 3) |
| Back up index now | Button | Calls `IndexSyncScheduler.runNow(context)` |
| Restore index from cloud | Button | Launches `IndexRecoveryActivity` with `FORCE_RESTORE_PROMPT = true` |

### 6. Active share links

A single row with chevron, opens `ActiveShareLinksActivity` (Task 39). Trailing text is the active-link count: "3 active" / "None". The count comes from `ShareLinkService.activeLinks().size` resolved on screen attach.

### 7. Account

| Row | Type | Notes |
|-----|------|-------|
| Re-enter credentials | Button | Launches `OnboardingActivity` in re-credential mode (existing Phase 2 affordance) |
| Sign out | Button | Confirmation dialog → clears B2 credentials, KEEPS the SQLite index intact, finishes back to `MainActivity` which routes to onboarding |

The "Sign out" confirmation copy: "Sign out of B2?\nYour SQLite index stays on this device. You can sign back in with the same or different credentials."

### 8. About

| Row | Notes |
|-----|-------|
| App version | Read-only — `BuildConfig.VERSION_NAME` (`BuildConfig.VERSION_CODE`) |
| B2 endpoint | Read-only — current endpoint from credentials (e.g., `s3.us-west-001.backblazeb2.com`) |
| Project repo | Tap → opens the README URL in a browser |

## Strings

```xml
<string name="settings_title">Settings</string>

<string name="settings_section_backup">Backup</string>
<string name="settings_auto_upload">Auto-upload</string>
<string name="settings_timing">Backup timing</string>
<string name="settings_timing_immediate">Immediate</string>
<string name="settings_timing_scheduled">Scheduled (nightly)</string>
<string name="settings_timing_hybrid">Hybrid</string>
<string name="settings_timing_immediate_desc">Upload as photos are taken.</string>
<string name="settings_timing_scheduled_desc">Upload during the nightly window (02:00–04:00 local).</string>
<string name="settings_timing_hybrid_desc">Upload on Wi-Fi; defer mobile data to the nightly window.</string>
<string name="settings_wifi_only">Wi-Fi only</string>

<string name="settings_section_videos">Videos</string>
<string name="settings_videos_enabled">Include videos</string>
<string name="settings_videos_quality">Quality mode</string>
<string name="settings_videos_quality_original">Upload originals</string>
<string name="settings_videos_quality_compressed">Compress all videos</string>
<string name="settings_videos_quality_duration_based">Compress long videos only</string>
<string name="settings_videos_threshold">Compress videos longer than</string>
<string name="settings_videos_threshold_units">%1$d min</string>
<string name="settings_videos_target_resolution">Compressed resolution</string>
<string name="settings_videos_permission_denied">Videos disabled — permission denied.</string>

<string name="settings_section_storage">Storage management</string>
<string name="settings_storage_strategy">Auto-delete strategy</string>
<string name="settings_storage_strategy_never">Never</string>
<string name="settings_storage_strategy_immediate">Immediately after upload</string>
<string name="settings_storage_strategy_after_days">After X days</string>
<string name="settings_storage_strategy_after_count">After N new photos</string>
<string name="settings_storage_days">Days after upload</string>
<string name="settings_storage_count">New photos before deletion</string>
<string name="settings_storage_review_now">Review pending deletions now</string>

<string name="settings_section_storage_cost">Storage &amp; cost</string>
<string name="settings_section_index">Backup index</string>
<string name="settings_index_last">Last backed up: %1$s</string>
<string name="settings_index_never">Last backed up: never</string>
<string name="settings_index_back_up_now">Back up index now</string>
<string name="settings_index_restore">Restore index from cloud</string>

<string name="settings_section_sharing">Active share links</string>
<string name="settings_sharing_active_count">%1$d active</string>
<string name="settings_sharing_none">None</string>

<string name="settings_section_account">Account</string>
<string name="settings_account_re_enter">Re-enter credentials</string>
<string name="settings_account_sign_out">Sign out</string>
<string name="settings_account_sign_out_confirm">Sign out of B2?\nYour SQLite index stays on this device. You can sign back in with the same or different credentials.</string>

<string name="settings_section_about">About</string>
<string name="settings_about_version">App version: %1$s (%2$d)</string>
<string name="settings_about_endpoint">B2 endpoint: %1$s</string>
<string name="settings_about_repo">Project repository</string>
```

## State management

```kotlin
data class SettingsState(
    val autoUpload: Boolean,
    val uploadMode: String,
    val wifiOnly: Boolean,
    val videosEnabled: Boolean,
    val videoQualityMode: String,
    val videoDurationThresholdMinutes: Int,
    val videoTargetResolution: String,
    val localDeleteStrategy: String,
    val localDeleteDays: Int,
    val localDeleteCount: Int,
    val lastIndexSyncAt: Long?,
    val activeShareLinkCount: Int
) {
    companion object {
        fun from(prefs: PrefsStore): SettingsState = SettingsState(
            autoUpload = prefs.isAutoUploadEnabled(),
            uploadMode = prefs.getUploadMode(),
            wifiOnly = prefs.isWifiOnly(),
            videosEnabled = prefs.getVideosEnabled(),
            videoQualityMode = prefs.getVideoQualityMode(),
            videoDurationThresholdMinutes = prefs.getVideoDurationThresholdMinutes(),
            videoTargetResolution = prefs.getVideoTargetResolution(),
            localDeleteStrategy = prefs.getLocalDeleteStrategy(),
            localDeleteDays = prefs.getLocalDeleteDays(),
            localDeleteCount = prefs.getLocalDeleteCount(),
            lastIndexSyncAt = prefs.getLastIndexSyncAt(),
            activeShareLinkCount = 0 // populated async on attach
        )
    }
}
```

Updates write through `PrefsStore` immediately and refresh local state. The Compose recomposition naturally reflects the change.

Active share link count is fetched in a `LaunchedEffect(Unit) { ... }` block.

## Implementation notes

- All preference writes are immediate and idempotent. There is no "Save" button — settings apply on toggle.
- Reverting a Switch on permission denial: drive the switch state from `SettingsState` (which is read back from `PrefsStore` after the write). The permission rationale flow updates the pref to `false` if denied, which the switch reflects on the next recomposition.
- The "Backup timing" bottom sheet can be implemented with `ModalBottomSheet` (Material3) — match whatever Compose flavor Phase 3 already uses. If Phase 3 has not introduced Compose Material3, fall back to `BottomSheetDialogFragment` with an XML layout (still inside this task's ownership; do not modify Phase 3 themes).
- For the `R.string.settings_timing_*_desc` strings, keep them under 80 chars so they fit the bottom-sheet rows on small phones without wrapping awkwardly.
- The "Re-enter credentials" button reuses Phase 2's `OnboardingActivity`. If the activity does not support a re-credential entry mode, add a small extra `FORCE_RECREDENTIAL` intent extra that bypasses the welcome screen. That edit is allowed by Task 41 because no other Phase 4 task touches that activity.
- The "Sign out" action clears the credentials via Phase 2's existing `PrefsStore.clearCredentials()` (or equivalent) and finishes back to `MainActivity`.

## Constraints

- One settings screen — do not create per-section activities. Sub-screens that DO exist (cost dashboard, active share links, recovery) are launched from this screen, not built into it.
- Settings writes never gate on confirmation except for "Sign out".
- The screen must remain responsive when `auto_upload_enabled` changes (it has to send a start/stop to the foreground service).
- Do not surface unimplemented features. Specifically, do not list "Bandwidth limit", "Encryption", or "Multiple buckets" — those are out of scope.

## Dependencies (by interface)

- `PrefsStore` (Task 32 + Phase 2/3) — every read and write
- `IndexSyncScheduler` (Phase 3 Task 27) — `runNow`
- `IndexRecoveryActivity` (Phase 3 Task 28) — re-launch with `FORCE_RESTORE_PROMPT`
- `LocalDeleteScheduler` (Task 37) — `runNow`
- `CostDashboardActivity` (Task 38)
- `ActiveShareLinksActivity` (Task 39)
- `ShareLinkService` (Task 39) — `activeLinks().size`
- `OnboardingActivity` (Phase 2) — `FORCE_RECREDENTIAL` mode (may require a one-line addition to the activity by this task)
- `UploadForegroundService` (Phase 2) — `start(context)` / `stop(context)`

## Acceptance criteria

- [ ] Screen renders eight sections in the documented order without crashes.
- [ ] Toggling auto-upload off stops the foreground service; toggling on starts it.
- [ ] Switching Backup timing through the three modes and taking a test photo behaves per the Task 36 mode definitions (immediate uploads, scheduled defers, hybrid Wi-Fi-only).
- [ ] Toggling "Include videos" on requests `READ_MEDIA_VIDEO`; denial reverts the switch.
- [ ] Changing the quality mode and duration threshold persists and is honored by the next video upload.
- [ ] Selecting a local-delete strategy persists; `Review pending deletions now` enqueues a one-shot run of `LocalDeleteWorker`.
- [ ] "Back up index now" runs the index sync worker; the "Last backed up" row updates after completion.
- [ ] "Restore index from cloud" launches `IndexRecoveryActivity` with the replace-confirmation prompt.
- [ ] "Active share links" row reflects the current count.
- [ ] "Sign out" clears credentials and routes back to onboarding.
- [ ] "About" shows the correct version name and B2 endpoint.

## Out of scope

- A search field inside settings.
- Settings backup/restore JSON.
- Tablet two-pane layout.
- Themes / dark-mode toggle (the app respects system dark mode automatically; no override here).
- A "Reset all settings" affordance.
- In-app help / FAQ entries — link to the project README only.
