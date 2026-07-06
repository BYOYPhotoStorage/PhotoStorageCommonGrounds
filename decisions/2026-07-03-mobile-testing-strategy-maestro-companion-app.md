# ADR: Mobile End-to-End Testing Strategy — Maestro + Companion App + ADB

## Status

Accepted

## Context

The MVP Android app is functionally complete and distributed to internal testers. The next validation gate is proving that the upload pipeline behaves correctly under real-world conditions:

- New photos and videos appear in the camera roll and get uploaded.
- Deletions from the device or from the cloud are reflected correctly.
- The app recovers from offline periods and resumes uploads.
- Local state (SQLite DB) remains consistent with cloud state.

Manual testing on a single device is not repeatable or scalable. We need an automated or semi-automated testing approach that can:

1. Put the device into a known state (media on device, app data cleared or not).
2. Drive the app through user-facing flows.
3. Inject system-level events such as new media and network changes.
4. Verify outcomes both on the device (DB, UI) and in the cloud (S3/B2 test bucket).

This ADR records the chosen testing stack and the trade-offs that led to it.

## Requirements

From brainstorming, the following scenarios must be supported now or in the near future:

- Inject deterministic photos and videos into the camera roll / MediaStore.
- Verify those objects land in the test bucket with correct content.
- Delete media from the device and observe app behavior.
- Verify video uploads, metadata, and thumbnails.
- Delete from cloud only, device only, or both, and assert each outcome.
- Download and inspect the SQLite database.
- Test offline behavior: capture media while offline, come back online, verify sync.

## Options Considered

### Option A: Pure Maestro

Use Maestro for all end-to-end tests. Flows are written in YAML and drive the UI directly.

- **Pros:** Very fast to set up, no compilation, readable tests, built-in flakiness handling, cross-platform if we add iOS later.
- **Cons:** Maestro cannot write to MediaStore, cannot toggle airplane mode, cannot read the app’s SQLite DB, and cannot verify S3 state. It is strictly UI-centric.
- **Verdict:** Rejected as a standalone solution. It covers only the UI-driving part of the requirements.

### Option B: Native Android Instrumented Tests (Espresso + UI Automator)

Write all E2E tests in Kotlin/JUnit inside the app repo, using Espresso for in-app interactions and UI Automator for cross-app/system interactions. A companion app could still be used for media injection.

- **Pros:** Maximum control, runs in Android Studio/Gradle, can access Android APIs directly, no extra CLI tools for the team to learn.
- **Cons:** More code to maintain, slower test authoring, harder for non-Android engineers to contribute, requires building and installing test APKs.
- **Verdict:** Viable, but heavier than necessary for the current stage. Kept as a fallback if Maestro-based flows prove too limiting.

### Option C: Appium

Use Appium with the UIAutomator2 driver for cross-platform, black-box automation.

- **Pros:** Mature ecosystem, real device cloud support, cross-platform.
- **Cons:** Slower execution, higher maintenance burden, complex setup, overkill for an Android-only native app.
- **Verdict:** Rejected. The project is Android-only and the team does not need WebDriver ecosystem portability.

### Option D: Maestro + Companion App + ADB + Cloud Harness (selected)

Use Maestro as the UI orchestrator, a separate Android companion app for media injection and device-state setup, `adb` shell commands for network toggling and DB extraction, and a small cloud-side harness (Python/Kotlin) for bucket verification.

- **Pros:**
  - Maestro gives fast, readable UI tests.
  - Companion app gives full control over MediaStore without touching the main app.
  - `adb` gives reliable network and app-data control.
  - Cloud harness gives independent verification of upload correctness.
  - Each layer can be developed and debugged independently.
- **Cons:**
  - Multiple moving parts (four tools instead of one).
  - Requires orchestration script to run a full scenario end to end.
  - Maestro cannot directly invoke companion app actions or `adb`; a wrapper script is required.
- **Verdict:** Selected. It covers every requirement from brainstorming with the least total implementation cost.

## Decision

Adopt a **hybrid testing stack**:

1. **Maestro** for driving the main app UI and asserting on-screen state.
2. **Companion Android app** (`photo-storage-tester`) for injecting and deleting MediaStore photos/videos.
3. **`adb` shell scripts** for network control, app data reset, and SQLite extraction.
4. **Cloud verification harness** (Python/Kotlin CLI) for asserting S3/B2 bucket state.
5. **Orchestration runner** (shell or Python) that sequences the above into reproducible scenarios.

This is intentionally not a single framework. The problem spans UI, system state, and cloud state, and no single tool covers all three well.

## Consequences

### Positive

- **Full coverage of brainstormed scenarios.** Media injection, offline toggling, DB inspection, and cloud verification are all supported.
- **Fast UI authoring.** Maestro YAML can be written and iterated on without recompiling the app.
- **Main app stays clean.** The companion app is a separate module/APK; no test code ships in release builds.
- **Independent verification.** The cloud harness can assert uploads even when the device UI is not changing.
- **Works on physical devices and emulators.** `adb` and Maestro both support real hardware.

### Negative

- **Orchestration complexity.** A full scenario requires coordinating four tools. This is more complex than a pure Maestro or pure Espresso suite.
- **Local-only SQLite extraction on release builds.** `run-as` and DB pull require a debuggable build. Release builds will need a debug-only export path.
- **Maestro limitations remain.** Complex conditional logic or dynamic data in Maestro flows can become unwieldy; we may eventually outgrow it.
- **CI cost.** Running emulators, Maestro, and a cloud harness in GitHub Actions consumes runner minutes and requires secret management for test bucket credentials.

## Mitigations

- Start with a small orchestration script (`run-scenario.sh`) that documents the exact order of operations. Replace with a Python runner only after the pattern stabilizes.
- Keep the companion app minimal: one screen, one intent-based API for injection/cleanup, no UI polish.
- Commit a small set of reference media files with known hashes so cloud verification is deterministic.
- Use a dedicated, lifecycle-managed test bucket and clean it up after each run.
- If Maestro becomes limiting for complex flows, migrate only those flows to Espresso/UI Automator; do not rewrite everything prematurely.

## Open Questions

1. **Companion app location:** Same repo as the main app, as a separate Gradle module `:companion`. This keeps versions aligned and lets the orchestration runner build both APKs from one checkout. (See `specs/phase4-tasks/43-companion-app.md`.)
2. **Cloud harness language:** Python with `boto3`, because it is faster to script and has better AWS/B2 tooling than a Kotlin CLI. (See `specs/phase4-tasks/44-cloud-verification-harness.md`.)
3. **Debug-only DB access:** Not required for the first scenarios. `run-as` on debug builds is sufficient. If release-build E2E becomes necessary, add a debug-only export path later rather than a `ContentProvider`.
4. **CI runner:** Deferred. Local emulators and physical devices are supported first; CI runner choice will be recorded in a follow-up ADR once scenarios are stable.
5. **Maestro flow versioning:** Maestro flows live in the main app repo (`maestroTests/`) and ship with the app release they test. This mirrors the companion app and cloud harness living in the same repo.
