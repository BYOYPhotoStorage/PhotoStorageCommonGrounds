# Task 45 — E2E Orchestration Runner

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md), [`../../decisions/2026-07-03-mobile-testing-strategy-maestro-companion-app.md`](../../decisions/2026-07-03-mobile-testing-strategy-maestro-companion-app.md), [`43-companion-app.md`](43-companion-app.md), and [`44-cloud-verification-harness.md`](44-cloud-verification-harness.md) first.

## Goal

Provide a thin shell orchestration layer that sequences the four tools from the mobile-testing ADR into reproducible end-to-end scenarios:

1. `adb` for device/app state control.
2. The companion app for media injection and cleanup (pulling from `refImages0307`).
3. Maestro for UI driving and onboarding.
4. The Python cloud harness for bucket verification in `test0307`.

The runner should be easy to run locally, easy to inspect when something fails, and easy to extend with new scenarios.

## Scope

- A Bash entry point: `e2e-runner/run-scenario.sh`.
- One or more scenario files in `e2e-runner/scenarios/`.
- Minimal Python glue only if Bash becomes unwieldy.
- Artifact collection on failure.

## Files you own

New files under `photoStorageApp/e2e-runner/`:

- `run-scenario.sh`
- `scenarios/photo-upload-baseline.sh`
- `scenarios/offline-then-sync.sh`
- `scenarios/device-delete-keeps-cloud.sh`
- `lib/common.sh` (shared helpers)

Modified files:

- `photoStorageApp/Makefile` — add `e2e-baseline`, `e2e-clean` targets.

## Reference media setup

Before running scenarios, populate the reference bucket that the companion app reads from:

```bash
# Upload some reference images and videos
aws s3 cp ./reference-assets/ s3://refImages0307/smoke/ \
  --endpoint-url=https://s3.us-west-004.backblazeb2.com --recursive
```

The companion app lists `refImages0307`, optionally restricted by `prefix`, and randomly selects the requested number of items.

## Runner design

`run-scenario.sh` is a thin dispatcher:

```bash
./e2e-runner/run-scenario.sh scenarios/photo-upload-baseline.sh
```

It sets up the environment, runs the scenario, collects artifacts, and always cleans up. Scenario scripts are ordinary Bash files sourced by the runner; they implement the test-specific sequence.

### Shared environment

`e2e-runner/lib/common.sh` exports:

```bash
DEVICE_ID=$(adb get-serialno)
MAIN_PACKAGE=com.hriyaan.photostorage
COMPANION_PACKAGE=com.photostorage.tester
RUN_ID=$(uuidgen | cut -d'-' -f1)
REPORT_DIR=./e2e-reports
ARTIFACT_DIR=./e2e-artifacts/${RUN_ID}
B2_UPLOAD_BUCKET=${B2_UPLOAD_BUCKET:-test0307}
B2_REF_BUCKET=${B2_REF_BUCKET:-refImages0307}
B2_REF_PREFIX=${B2_REF_PREFIX:-smoke/}
MAESTRO_BIN="${HOME}/.maestro/bin/maestro"
MAESTRO_JAVA_HOME="${HOME}/.jdks/jdk-17.0.19+10/Contents/Home"
```

It also defines helpers:

```bash
wait_for_logcat(tag, pattern, timeout)
wait_for_report(tag, timeout)
pull_db(run_id)
pull_report(tag)
clear_main_app_data()
grant_main_app_permissions()
inject_media(tag, count, type, prefix?, seed?)
delete_media_by_tag(tag)
run_maestro(flow)
verify_bucket(tag)
cleanup_bucket(tag)
```

## Baseline scenario: inject and upload a photo

`e2e-runner/scenarios/photo-upload-baseline.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "$0")/../lib/common.sh"

log "RUN_ID=${RUN_ID}"

# 1. Reset state
log "Clearing main app data and wiping previous test media"
clear_main_app_data
adb shell am start-service \
  -a com.photostorage.tester.WIPE_ALL_TEST_MEDIA \
  -n "${COMPANION_PACKAGE}/.InjectionService" || true

# 2. Install / ensure APKs are present
log "Installing main and companion APKs"
./gradlew :app:installDebug :companion:installDebug --console=plain --quiet

# 3. Inject photos from the reference bucket
log "Injecting photos from ${B2_REF_BUCKET}/${B2_REF_PREFIX}"
inject_media "${RUN_ID}" 3 "photo" "${B2_REF_PREFIX}" 42

# 4. Run Maestro onboarding and wait for gallery
log "Driving UI with Maestro"
run_maestro maestroTests/login-with-test0307.yaml

# 5. Wait for the upload worker to finish
log "Waiting for IndexSyncWorker / UploadWorker completion"
wait_for_logcat "UploadWorker" "upload complete.*ps_test_${RUN_ID}" 120

# 6. Pull the companion report and verify the upload bucket
log "Verifying bucket state"
pull_report "${RUN_ID}"
verify_bucket "${RUN_ID}"

# 7. Pull DB for post-hoc inspection
log "Pulling SQLite DB"
pull_db "${RUN_ID}"

log "Scenario passed"
```

The scenario is intentionally linear. If any step fails, `set -e` exits and the runner's cleanup trap runs.

## Offline scenario: capture while offline, sync when back

`e2e-runner/scenarios/offline-then-sync.sh`:

```bash
# ... setup ...
adb shell cmd connectivity airplane-mode enable
inject_media "${RUN_ID}" 3 "both" "${B2_REF_PREFIX}" 42
run_maestro flows/wait-for-gallery.yaml

# Assert no upload happened while offline
if wait_for_logcat "UploadWorker" "upload complete" 20; then
  log "ERROR: upload happened while offline"
  exit 1
fi

adb shell cmd connectivity airplane-mode disable
wait_for_logcat "UploadWorker" "upload complete.*ps_test_${RUN_ID}" 120
pull_report "${RUN_ID}"
verify_bucket "${RUN_ID}"
```

## Device-delete scenario

`e2e-runner/scenarios/device-delete-keeps-cloud.sh`:

```bash
# ... inject and upload ...
adb shell am start-service \
  -a com.photostorage.tester.DELETE_BY_TAG \
  --es tag "${RUN_ID}" \
  -n "${COMPANION_PACKAGE}/.InjectionService"

# Trigger a sync in the main app
adb shell am broadcast -a com.hriyaan.photostorage.SYNC_NOW || true
wait_for_logcat "IndexSyncWorker" "sync complete" 60

# Verify upload-bucket objects still exist
python e2e-runner/verify_bucket.py \
  --tag "${RUN_ID}" \
  --report "${REPORT_DIR}/${RUN_ID}.json"
```

## Artifact collection

The runner creates `e2e-artifacts/${RUN_ID}/` and, on failure or always, collects:

- `report-${RUN_ID}.json` — pulled injection report.
- `verified-${RUN_ID}.json` — cloud harness output.
- `upload-${RUN_ID}.db` — pulled SQLite database.
- `logcat-${RUN_ID}.txt` — filtered logcat.
- `screenshot-${RUN_ID}.png` — Maestro screenshot on failure.
- `maestro-output/` — Maestro logs.

The runner zips these and prints the path.

## Cleanup trap

`run-scenario.sh` uses a Bash `trap` so cleanup always runs:

```bash
cleanup() {
  local exit_code=$?
  log "Cleaning up"
  cleanup_bucket "${RUN_ID}" || true
  adb shell am start-service \
    -a com.photostorage.tester.DELETE_BY_TAG \
    --es tag "${RUN_ID}" \
    -n "${COMPANION_PACKAGE}/.InjectionService" || true
  adb shell cmd connectivity airplane-mode disable || true
  if [[ $exit_code -ne 0 ]]; then
    log "FAILED — artifacts in ${ARTIFACT_DIR}"
  fi
}
trap cleanup EXIT
```

## Makefile targets

Add to `photoStorageApp/Makefile`:

```makefile
E2E_RUNNER := ./e2e-runner/run-scenario.sh

.PHONY: e2e-baseline e2e-clean

e2e-baseline: ## Run the baseline inject + upload E2E scenario
	$(E2E_RUNNER) $(PWD)/e2e-runner/scenarios/photo-upload-baseline.sh

e2e-clean: ## Delete all ps_test_ objects from the test bucket and device
	python e2e-runner/cleanup_bucket.py --all-test-media
	adb shell am start-service \
	  -a com.photostorage.tester.WIPE_ALL_TEST_MEDIA \
	  -n com.photostorage.tester/.InjectionService || true
```

## Configuration

The runner relies on the same environment variables as the cloud harness (Task 44), plus:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `B2_KEY_ID` | yes | — | Passed to `verify_bucket.py` and used by the companion app if overridden |
| `B2_APPLICATION_KEY` | yes | — | Passed to `verify_bucket.py` and used by the companion app if overridden |
| `B2_UPLOAD_BUCKET` | no | `test0307` | Target upload bucket |
| `B2_REF_BUCKET` | no | `refImages0307` | Reference media bucket |
| `B2_REF_PREFIX` | no | `smoke/` | Prefix the companion app lists under |
| `ANDROID_SERIAL` | no | — | adb device selector |
| `SKIP_INSTALL` | no | `false` | Skip `./gradlew installDebug` if APKs are already installed |

## CI integration sketch

A GitHub Actions job (not implemented in this task) would:

1. Check out the repo.
2. Set up JDK 17 and Python.
3. Export B2 credentials from GitHub secrets.
4. Start an emulator (API 33 or 34).
5. Run `./gradlew :app:installDebug :companion:installDebug`.
6. Run `make e2e-baseline`.
7. On failure, upload `e2e-artifacts/`.
8. Always run `make e2e-clean`.

## Scenario ideas for the future

- `cloud-delete-syncs-to-device`
- `both-delete-removes-everywhere`
- `large-batch-resumes-after-kill`
- `dedup-does-not-reupload`

Each future scenario is just a new file in `e2e-runner/scenarios/` plus a small Makefile target.

## Notes

- Keep the runner in Bash until the pattern stabilizes. Replace with Python only after several scenarios prove the shape is right.
- The runner should fail loudly and collect everything needed to debug a flaky upload.
- Avoid sleeping blindly. Prefer logcat polling and file-existence checks.
- Document any Maestro flow assumptions (e.g., button IDs) in the scenario file header.
- Upload reference media to `refImages0307` before running scenarios. The runner does not manage the reference bucket contents.
