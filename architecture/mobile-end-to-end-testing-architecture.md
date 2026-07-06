# Mobile End-to-End Testing Architecture

## Overview

This document describes the testing architecture for validating the Photo Storage Android app. It is built around the principle that no single tool can cover UI behavior, system state, and cloud state together, so each responsibility is assigned to the tool best suited for it.

The architecture is intentionally decoupled. Each component can be developed, run, and debugged independently, and a thin orchestration layer combines them into full scenarios.

## Components

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Orchestration Runner                             │
│              (Bash/Python — sequences the full scenario)                 │
└─────────────────────────────────────────────────────────────────────────┘
        │                  │                  │                  │
        ▼                  ▼                  ▼                  ▼
┌──────────────┐  ┌────────────────┐  ┌─────────────┐  ┌──────────────────┐
│   Maestro    │  │  Companion App │  │     adb     │  │  Cloud Harness   │
│  UI driver   │  │  MediaStore    │  │  shell cmds │  │  S3/B2 verifier  │
│              │  │  injector      │  │             │  │                  │
└──────┬───────┘  └───────┬────────┘  └──────┬──────┘  └────────┬─────────┘
       │                  │                  │                  │
       │                  │                  │                  │
       ▼                  ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Device Under Test                                │
│                    Photo Storage Android App                             │
│                                                                          │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐     │
│   │   Activities │    │   Workers    │    │   SQLite + Prefs     │     │
│   │  (Gallery,   │◀──▶│ (IndexSync,  │◀──▶│  (Upload records,    │     │
│   │  Onboarding) │    │  Initial     │    │   credentials)       │     │
│   │              │    │  Backfill)   │    │                      │     │
│   └──────┬───────┘    └──────┬───────┘    └──────────┬───────────┘     │
│          │                   │                       │                  │
│          └───────────────────┼───────────────────────┘                  │
│                              │                                          │
│                              ▼                                          │
│                   ┌──────────────────────┐                              │
│                   │      MediaStore      │                              │
│                   │  (camera roll)       │                              │
│                   └──────────────────────┘                              │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                   ┌──────────────────────┐
                   │   S3 / B2 Test       │
                   │   Bucket             │
                   └──────────────────────┘
```

### 1. Maestro — UI orchestrator

Maestro runs YAML flows against the installed APK. It treats the main app as a black box.

**Responsibilities**
- Launch the app, clear state, and navigate onboarding.
- Tap buttons, grant permissions, and scroll the gallery.
- Assert that UI elements are visible or hidden.
- Trigger user actions such as “Start backup” or “Delete from cloud”.

**Inputs**
- Installed `com.hriyaan.photostorage` APK
- YAML flow files in a `maestro/` directory

**Outputs**
- Pass/fail per flow
- Screenshots and hierarchy dumps on failure

**Limitations**
- Cannot inject media.
- Cannot toggle network state.
- Cannot read app databases or cloud state.

### 2. Companion App — MediaStore controller

A separate Android application whose only job is to manipulate the device state that the main app observes through `MediaStore`.

**Responsibilities**
- List and download reference photos and videos from a dedicated B2 bucket (`refImages0307`).
- Randomly select a requested number of items, salt them so each injection is unique, and insert them into `MediaStore`.
- Tag injected media so tests can identify and clean it up later (e.g. filename prefix `ps_test_<run-id>`).
- Delete injected media by tag or by full wipe.
- Report back to the orchestration runner via logcat, broadcast, and a JSON status file.

**Intent-based API**

The companion app exposes actions via explicit intents so the runner can trigger it without touching its UI:

```
com.photostorage.tester.INJECT_PHOTOS
  --es tag <run-id>
  --ei count 10
  --es type photo|video|both
  --es refBucket refImages0307
  --es prefix smoke/
  --ei seed 42

com.photostorage.tester.DELETE_BY_TAG
  --es tag <run-id>

com.photostorage.tester.WIPE_ALL_TEST_MEDIA
```

**Why a separate app?**
- Keeps test code out of the main app’s release APK.
- Can hold broad media permissions without changing the main app’s permission model.
- Can be built, installed, and updated independently.

### 3. `adb` — System and app state control

Standard Android Debug Bridge commands are used for operations that Android apps cannot perform on themselves.

**Responsibilities**
- Install/uninstall APKs.
- Clear app data: `adb shell pm clear com.hriyaan.photostorage`
- Toggle airplane mode: `adb shell cmd connectivity airplane-mode enable|disable`
- Extract the SQLite DB: `adb shell run-as ... cp ...` + `adb pull`
- Capture logcat: `adb logcat -s IndexSyncWorker:D`
- Capture screenshots and screen recordings.

**Why `adb`?**
- Reliable, well-documented, and works on emulators and most physical devices.
- Does not require code changes in the main app.

### 4. Cloud Verification Harness — Independent bucket assertions

A host-side program (Python or Kotlin CLI) that talks directly to the test bucket and verifies uploads independently of the device.

**Responsibilities**
- List objects by prefix.
- Compare ETag / SHA256 / size against reference files.
- Check metadata headers such as original timestamp or content type.
- Assert that deleted objects are removed or tombstoned according to policy.
- Create and clean up test buckets.

**Inputs**
- Test run ID
- Reference asset manifest (filename → expected hash, size, MIME type)
- S3/B2 credentials from environment variables

**Outputs**
- JSON or human-readable report
- Non-zero exit code on mismatch

**Why independent?**
- It prevents the app from “passing” its own tests through a bug in its own verification logic.
- It can run on the CI host while the device test runs on an emulator or physical device.

### 5. Orchestration Runner — Scenario glue

A script that runs the components in the right order and passes context (such as a run ID) between them.

**Responsibilities**
- Generate a unique run ID.
- Reset device and app state.
- Call companion app intents to inject media.
- Toggle network state via `adb`.
- Run Maestro flows.
- Wait for background workers to complete.
- Run cloud verification.
- Pull DB and artifacts on failure.
- Clean up the test bucket.

**Example scenario: offline capture then sync**

```
1. RUN_ID=$(uuidgen)
2. adb shell pm clear com.hriyaan.photostorage
3. adb shell am start -a com.photostorage.tester.INJECT_PHOTOS --es tag $RUN_ID --ei count 5
4. adb shell cmd connectivity airplane-mode enable
5. maestro test flows/launch_and_observe.yaml
6. adb shell cmd connectivity airplane-mode disable
7. adb shell am start -a com.hriyaan.photostorage.SYNC_NOW   # optional debug intent
8. poll logcat until IndexSyncWorker succeeds or timeout
9. python verify_bucket.py --tag $RUN_ID
10. adb shell run-as com.hriyaan.photostorage cp databases/upload.db /sdcard/Download/db-$RUN_ID.db
11. adb pull /sdcard/Download/db-$RUN_ID.db ./artifacts/
12. python verify_db.py ./artifacts/db-$RUN_ID.db --tag $RUN_ID
13. python cleanup_bucket.py --tag $RUN_ID
```

## Data Flows

### Inject and upload a photo

1. Runner generates `RUN_ID`.
2. Companion app injects `ps_test_RUN_ID_001.jpg` into `MediaStore`.
3. Runner launches the main app via Maestro and taps through onboarding/permission grant.
4. Main app queries `MediaStore`, sees the new file, and enqueues `InitialBackfillWorker`/`IndexSyncWorker`.
5. Worker uploads the file to the test bucket using `S3Uploader`.
6. Cloud harness lists the bucket, finds the object, and verifies its hash.
7. Runner pulls the SQLite DB and confirms the upload record state.

### Delete from device only

1. Companion app deletes `ps_test_RUN_ID_001.jpg` from `MediaStore`.
2. Runner triggers a sync in the main app.
3. Main app marks the local record as “device deleted” but retains the cloud copy.
4. Cloud harness confirms the object still exists in the bucket.
5. DB verifier confirms the record is present but flagged as locally deleted.

### Offline then online

1. Runner enables airplane mode.
2. Companion app injects media while offline.
3. Runner launches the main app and verifies that it observes the media but does not upload (via logcat / UI state).
4. Runner disables airplane mode.
5. Workers resume and upload the queued files.
6. Cloud harness verifies all injected media is present.

## Repository Layout (proposal)

```
photoStorageApp/                       # existing app repo
├── app/
├── companion/                         # new: media injector app
│   ├── build.gradle.kts
│   └── src/main/...
├── maestro/                           # new: UI flows
│   ├── onboarding.yaml
│   ├── backup_flow.yaml
│   ├── delete_dialogs.yaml
│   └── offline_scenario.yaml
├── e2e-runner/                        # new: orchestration + cloud harness
│   ├── runner.py
│   ├── verify_bucket.py
│   ├── verify_db.py
│   ├── cleanup_bucket.py
│   └── reference-assets/
│       ├── photo-1.jpg
│       ├── photo-2.heic
│       ├── video-1.mp4
│       └── manifest.json
└── settings.gradle.kts                # include ':companion'
```

The companion app and Maestro flows live in the main app repo so they stay in sync with app releases. The orchestration runner can live there too, or in `PhotoStorageCommonGrounds` if it is shared across future platforms.

## CI Integration Sketch

For GitHub Actions:

1. Start Android emulator (API 33 or 34 to match `minSdk`).
2. Build and install both APKs.
3. Start `maestro` installation.
4. Run the orchestration runner.
5. On failure, upload artifacts: screenshots, screen recording, logcat, pulled DB.
6. Always clean up the test bucket.

The cloud harness runs on the Ubuntu runner, not on the device, so it can use standard AWS tooling.

## Security and Isolation

- The test bucket must be separate from production buckets.
- Test credentials must be short-lived or scoped to the test bucket only.
- Injected media should not contain real PII; use generated images/videos.
- DB extraction only works on debug builds. Release builds must not expose debug interfaces.
- The companion app should be signed with a debug key and never distributed to end users.

## Decision Rationale

This architecture was chosen because the problem domain spans three independent surfaces:

1. **UI surface** — best tested with a black-box UI driver (Maestro).
2. **System surface** — requires an app with media permissions and `adb` for network control.
3. **Cloud surface** — requires an independent verifier on the host.

Trying to force all three into a single framework would create either blind spots (Maestro-only) or heavy maintenance (Appium/Espresso for everything). The hybrid approach assigns each surface to the right tool and connects them with a thin orchestration layer.
