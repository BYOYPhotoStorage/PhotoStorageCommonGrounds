# Project Status

_Last updated: 2026-05-16_

## Where the code lives

The Android app is implemented in a separate repository:

**[BYOYPhotoStorage/PhotoStorageAndroidApp](https://github.com/BYOYPhotoStorage/PhotoStorageAndroidApp)**

This repo (`PhotoStorageCommonGrounds`) remains the home for product, architecture, and decision docs.

## Current phase

MVP build complete and distributing to internal testers via Firebase App Distribution. Now validating the upload pipeline end-to-end before scoping post-MVP work.

## What's shipped

All 9 MVP tasks from [`specs/mvp-tasks/`](./specs/mvp-tasks/) are implemented on `main` of the app repo:

| # | Task | Status |
|---|------|--------|
| 01 | Project scaffolding | done |
| 02 | Data layer (SQLite) | done |
| 03 | Credential storage | done |
| 04 | MediaStore query | done |
| 05 | Thumbnail generator | done |
| 06 | B2/S3 uploader | done |
| 07 | Onboarding UI | done |
| 08 | Gallery UI | done |
| 09 | Integration & wire-up | done |

## Distribution

- CI: GitHub Actions workflow at `.github/workflows/firebase-distribution.yml` in the app repo
- Trigger: manual (`workflow_dispatch`) with optional release notes and tester-group inputs
- Output: signed release APK uploaded to Firebase App Distribution, plus a 14-day workflow artifact
- Default tester group: `qa-testers`
- See [`decisions/2026-05-16-github-actions-firebase-app-distribution.md`](./decisions/2026-05-16-github-actions-firebase-app-distribution.md) for the why and the secret list

## Deviations from spec

- **Extra UI helper.** `SquareFrameLayout` exists in `ui/` to satisfy the spec's "square FrameLayout via aspect-ratio" requirement for the gallery item layout. Additive, not breaking.

(The package ID rename — `com.photobackup.app` → `com.hriyaan.photostorage` — has been backported into the specs and is no longer a deviation.)

## What's next

Post-MVP scope (see [`specs/mvp-prd.md`](./specs/mvp-prd.md) "Out of Scope") is not yet planned. Expected validation gates before that planning starts:

- Internal testers complete an end-to-end upload and confirm thumbnails + originals land in the expected B2 paths
- B2 checksum-quirk workaround holds in real-device builds (see the [S3 API ADR](./decisions/2026-05-09-use-s3-compatible-api-over-native-b2-api.md))
