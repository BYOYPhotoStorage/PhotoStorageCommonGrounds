# Task 51 — Idempotent photo lifecycle E2E scenario

> Read [`../phase4-implementation-overview.md`](../phase4-implementation-overview.md), [`../../decisions/2026-07-03-mobile-testing-strategy-maestro-companion-app.md`](../../decisions/2026-07-03-mobile-testing-strategy-maestro-companion-app.md), [`../phase4-tasks/45-e2e-orchestration-runner.md`](../phase4-tasks/45-e2e-orchestration-runner.md), [`../phase4-tasks/44-cloud-verification-harness.md`](../phase4-tasks/44-cloud-verification-harness.md), [`../phase3-tasks/27-index-sync-worker.md`](../phase3-tasks/27-index-sync-worker.md), [`../phase3-tasks/28-reinstall-recovery.md`](../phase3-tasks/28-reinstall-recovery.md), and [`../phase3-tasks/25-deletion-engine.md`](../phase3-tasks/25-deletion-engine.md) first.

## Goal

Create a single, repeatable end-to-end scenario that exercises the full photo
lifecycle:

1. Upload photos/videos that already exist on the device before the app is
   launched.
2. Back up the SQLite index to B2.
3. Inject new photos/videos after the app is running and verify they upload.
4. Delete the new items locally only — cloud copies must remain.
5. Delete the new items from the cloud — B2 objects and thumbnails must be
   removed.
6. Re-sync the SQLite index to B2 and verify the remote index matches the
   post-deletion local state.

The scenario must be **idempotent**: starting from an empty test bucket and a
clean app, running it N times produces the same deterministic end state.

## Scope

- One new orchestration scenario script.
- Two or three small Maestro flows for the app-driven deletion steps.
- A host-side helper that downloads the remote SQLite index and compares it with
  the pulled local `uploads.db`.
- Makefile target to run the scenario.

## Files you own

New files under `photoStorageApp/`:

- `e2e-runner/scenarios/idempotent-lifecycle.sh`
- `maestroTests/idempotent-lifecycle-onboard.yaml`
- `maestroTests/idempotent-lifecycle-delete-local.yaml`
- `maestroTests/idempotent-lifecycle-delete-cloud.yaml`
- `e2e-runner/verify_index.py` (or extend `verify_bucket.py`)

Modified files:

- `e2e-runner/lib/common.sh` — add helpers for remote-index download and index-sync trigger.
- `Makefile` — add `e2e-idempotent` target.

## Preconditions

- Reference media exists in `s3://refImages0307/smoke/`.
- `B2_KEY_ID`, `B2_APPLICATION_KEY`, and optionally `B2_UPLOAD_BUCKET` are set.
- A device or emulator is connected and both APKs are installed (or
  `SKIP_INSTALL=false`).

## Scenario

### Tag convention

Use two tags per run so the existing and new media can be verified independently:

- `lifecycle-existing-${RUN_ID}` — injected before the app is launched.
- `lifecycle-new-${RUN_ID}` — injected while the app is running.

### Step-by-step

#### 1. Empty the test bucket

Delete every object under `photos/`, `videos/`, `thumbnails/`, and `index/` in
`$B2_UPLOAD_BUCKET`. The scenario must start from a known-empty cloud state.

```bash
aws s3 rm "s3://${B2_UPLOAD_BUCKET}/" --recursive \
  --endpoint-url="https://s3.${B2_REGION}.backblazeb2.com" || true
```

#### 2. Clean the device and app

```bash
clear_main_app_data
wipe_all_test_media
```

`pm clear` removes the app’s `uploads.db` and preferences. `wipe_all_test_media`
removes all `ps_test_*` items from `MediaStore`.

#### 3. Inject “existing” media before launch

Inject a mix of photos and videos that will already be on the device when the
app starts:

```bash
inject_media "lifecycle-existing-${RUN_ID}" 2 "photo" "${B2_REF_PREFIX}" 1001
inject_media "lifecycle-existing-${RUN_ID}" 1 "video" "${B2_REF_PREFIX}" 1002
```

#### 4. Launch the app and backfill existing media

Run the Maestro onboarding flow that logs in with the test shortcut and enables
auto-upload. The foreground service / `InitialBackfillWorker` discovers the
existing media and uploads it.

```bash
run_maestro maestroTests/idempotent-lifecycle-onboard.yaml
```

Wait until the existing items reach `status = 'uploaded'`:

```bash
wait_for_upload "lifecycle-existing-${RUN_ID}"
```

#### 5. Back up the SQLite index

Trigger the manual index sync from the UI (Maestro taps “Back up index now” in
the gallery overflow menu) and wait for `IndexSyncWorker` success:

```bash
run_maestro maestroTests/idempotent-lifecycle-backup-index.yaml
wait_for_logcat "IndexSyncWorker" "Index uploaded" 60
```

Verify the remote index exists:

```bash
python e2e-runner/verify_index.py --bucket "${B2_UPLOAD_BUCKET}" --check-exists
```

Pull the local DB for a baseline snapshot:

```bash
pull_db "${RUN_ID}-baseline"
```

#### 6. Inject “new” media after launch

While the app is running, inject additional photos and videos. The foreground
service should detect them and upload them.

```bash
inject_media "lifecycle-new-${RUN_ID}" 2 "photo" "${B2_REF_PREFIX}" 2001
inject_media "lifecycle-new-${RUN_ID}" 1 "video" "${B2_REF_PREFIX}" 2002
```

Wait for uploads:

```bash
wait_for_upload "lifecycle-new-${RUN_ID}"
```

Verify both sets exist in the bucket:

```bash
verify_bucket --tag "lifecycle-existing-${RUN_ID}"
verify_bucket --tag "lifecycle-new-${RUN_ID}"
```

#### 7. Delete new media locally only

From the app UI, switch to the **Local** or **Merged** gallery view, select the
new items, and choose the device-only delete option. In the current
`DeletionEngine` semantics, deleting a `Synced` item from the Local view removes
the `MediaStore` entry and sets `localPresent = 0`, but leaves the cloud object
intact.

```bash
run_maestro maestroTests/idempotent-lifecycle-delete-local.yaml
```

Wait for the gallery to refresh, then verify:

- **Bucket:** objects for `lifecycle-new-${RUN_ID}` are still present.
- **Local DB:** rows for the new items have `localPresent = 0` and
  `status = 'uploaded'`.

```bash
pull_db "${RUN_ID}-after-local-delete"
python e2e-runner/verify_db.py \
  --db "${ARTIFACT_DIR}/upload-${RUN_ID}-after-local-delete.db" \
  --tag "lifecycle-new-${RUN_ID}" \
  --expect-local-present false \
  --expect-status uploaded
```

#### 8. Delete new media from the cloud

Switch to the **Cloud** gallery view, select the same new items, and delete
them. In Cloud view, `DeletionEngine` deletes the B2 photo object and its
thumbnail, then soft-deletes the DB record (`status = 'cloud_deleted'`,
`cloud_deleted_at` set).

```bash
run_maestro maestroTests/idempotent-lifecycle-delete-cloud.yaml
```

Wait for cloud deletion to complete, then verify:

- **Bucket:** objects for `lifecycle-new-${RUN_ID}` are gone (both originals and
  thumbnails).
- **Bucket:** objects for `lifecycle-existing-${RUN_ID}` remain.
- **Local DB:** rows for the new items have `status = 'cloud_deleted'` and
  `localPresent = 0`.

```bash
verify_bucket --tag "lifecycle-new-${RUN_ID}" --expect-found false
verify_bucket --tag "lifecycle-existing-${RUN_ID}"
```

#### 9. Re-sync the SQLite index

Trigger the manual index sync again and wait for success:

```bash
run_maestro maestroTests/idempotent-lifecycle-backup-index.yaml
wait_for_logcat "IndexSyncWorker" "Index uploaded" 60
```

Pull the final local DB and download the remote index:

```bash
pull_db "${RUN_ID}-final"
python e2e-runner/verify_index.py \
  --bucket "${B2_UPLOAD_BUCKET}" \
  --local-db "${ARTIFACT_DIR}/upload-${RUN_ID}-final.db" \
  --tag "lifecycle-new-${RUN_ID}"
```

The remote index must contain the same rows/states as the local DB:

- Existing items: `status = 'uploaded'`, `localPresent = 1`.
- New items: `status = 'cloud_deleted'`, `localPresent = 0`,
  `cloud_deleted_at` populated.

This proves the index backup is updated after deletions and is not stuck at the
state captured in Step 5.

#### 10. Cleanup

Always run cleanup, even on failure:

```bash
cleanup_bucket --all-test-media
adb shell am start-service \
  -a com.photostorage.tester.WIPE_ALL_TEST_MEDIA \
  -n com.photostorage.tester/.InjectionService || true
```

## Idempotency requirement

After cleanup, re-running the scenario from Step 1 must produce the same final
state:

- Empty test bucket.
- No `ps_test_*` media on the device.
- Final remote index contains the same pattern of uploaded existing items and
  cloud-deleted new items as the previous run.

If the scenario is not idempotent, the most likely causes are:

1. The test bucket was not fully emptied.
2. `wipe_all_test_media` left injected media behind.
3. The app is using non-deterministic keys or timestamps in `S3KeyBuilder`.
4. Soft-delete cleanup ran during the scenario and hard-deleted records before
  the final index sync.

## Maestro flow assumptions

Document these in each flow file header:

- `idempotent-lifecycle-onboard.yaml`: logs in with `loginTest0307Button`,
  enables auto-upload, and lands in the gallery.
- `idempotent-lifecycle-delete-local.yaml`: selects the items whose display
  names contain `lifecycle-new-${RUN_ID}` from the Local/Merged view and deletes
  only the local copy.
- `idempotent-lifecycle-delete-cloud.yaml`: selects the same items from the
  Cloud view and deletes them from B2.
- `idempotent-lifecycle-backup-index.yaml`: opens the gallery overflow menu,
  taps “Back up index now,” and waits for the status line to update.

## Acceptance criteria

- [ ] Scenario script runs end-to-end and exits `0`.
- [ ] Existing media is uploaded before any new media is injected.
- [ ] After Step 5, `index/photo-storage-index.sqlite` exists in the test bucket
      and matches the local DB.
- [ ] After Step 6, both existing and new media are found in the bucket.
- [ ] After Step 7, new media is no longer in `MediaStore` but still exists in
      B2; DB `localPresent = 0`.
- [ ] After Step 8, new media B2 objects and thumbnails are gone; existing
      media remains.
- [ ] After Step 9, the remote index matches the final local DB, including
      `cloud_deleted` records.
- [ ] Running the scenario a second time after cleanup produces the same final
      DB pattern and an empty bucket.

## Verification helpers

### `verify_index.py`

A small Python CLI that:

1. Downloads `index/photo-storage-index.sqlite` from B2 to a temp file.
2. Opens both the remote index and the supplied local DB.
3. Compares rows for the given tag(s) on key columns:
   - `filename`
   - `status`
   - `localPresent`
   - `cloud_deleted_at`
   - `photo_b2_path`
   - `thumbnail_b2_path`
4. Exits `0` only if the states match.

Example:

```bash
python e2e-runner/verify_index.py \
  --bucket test0307 \
  --local-db ./e2e-artifacts/run-abc/upload-run-abc-final.db \
  --tag lifecycle-existing-run-abc \
  --tag lifecycle-new-run-abc
```

### `verify_db.py` extension

Add flags for asserting local-only delete state:

```bash
python e2e-runner/verify_db.py \
  --db upload.db \
  --tag lifecycle-new-run-abc \
  --expect-local-present false \
  --expect-status uploaded
```

## Makefile target

Add to `photoStorageApp/Makefile`:

```makefile
.PHONY: e2e-idempotent

e2e-idempotent: ## Run the idempotent photo lifecycle E2E scenario
	$(E2E_RUNNER) $(PWD)/e2e-runner/scenarios/idempotent-lifecycle.sh
```

## Out of scope

- Multi-device conflict resolution.
- Differential or incremental SQLite index sync (Phase 4 left this out of
  scope).
- Running the scenario against a bucket that is not first emptied.
- Testing at the 70K-photo scale; this scenario uses a small, fixed media set.
- Verifying that `SoftDeleteCleanupWorker` eventually hard-deletes records after
  24 hours (that is a separate scheduled-worker test).

## Notes

- The scenario intentionally does **not** use the nightly `IndexSyncWorker`
  schedule; it drives the sync manually so the test is deterministic and fast.
- Because the local DB is the source of truth, the only way to prove the remote
  index is correct is to download it and compare rows. Bucket verification alone
  is not enough.
- If the app’s delete UI does not expose a “delete from phone only” option in
  the current build, use the Local gallery view for Step 7 — its semantics are
  already local-only for `Synced` items.
