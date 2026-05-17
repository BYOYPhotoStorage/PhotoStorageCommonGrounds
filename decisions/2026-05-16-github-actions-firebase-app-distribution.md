# ADR: Distribute Internal Builds via GitHub Actions + Firebase App Distribution

## Status

Accepted

## Context

The MVP is feature-complete and needs to land on internal testers' phones to validate the upload pipeline against real B2 buckets and a variety of devices. We needed a way to:

1. Build a signed release APK reproducibly without depending on a local developer machine.
2. Hand that APK to a closed group of testers without publishing to the Play Store.
3. Keep secrets (keystore, Firebase service account, B2 test creds) out of the app repo.

The app source lives in [BYOYPhotoStorage/PhotoStorageAndroidApp](https://github.com/BYOYPhotoStorage/PhotoStorageAndroidApp), which already uses GitHub for code review — so reusing GitHub for CI was the path of least resistance.

## Decision

Use a **GitHub Actions workflow** (`.github/workflows/firebase-distribution.yml`) that builds a signed release APK and pushes it to **Firebase App Distribution** via the community [`wzieba/Firebase-Distribution-Github-Action`](https://github.com/wzieba/Firebase-Distribution-Github-Action) action.

Workflow shape:

- **Trigger:** `workflow_dispatch` only. Each tester drop is intentional — no auto-distribution on push.
- **Inputs:** `release_notes` (free-text shown to testers) and `tester_groups` (comma-separated, defaults to `qa-testers`).
- **Build:** JDK 17 + Gradle, `./gradlew :app:assembleRelease`. Signing config in `app/build.gradle.kts` reads the keystore path/password/alias/key-password from environment variables, so the same Gradle file produces unsigned debug builds locally and signed release builds in CI.
- **Distribution:** `wzieba/Firebase-Distribution-Github-Action@v1` uploads the APK to the configured Firebase app, tagged with the input release notes and routed to the chosen tester group(s).
- **Backup artifact:** the APK is also uploaded as a workflow artifact with 14-day retention, so a build can be re-distributed manually if Firebase upload fails.

### Required GitHub secrets

| Secret | Purpose |
|--------|---------|
| `ANDROID_KEYSTORE_BASE64` | Base64-encoded `.jks` decoded into `$RUNNER_TEMP` at build time |
| `ANDROID_KEYSTORE_PASSWORD` | Keystore password |
| `ANDROID_KEY_ALIAS` | Signing-key alias inside the keystore |
| `ANDROID_KEY_PASSWORD` | Signing-key password |
| `FIREBASE_APP_ID` | Firebase Android app ID (e.g. `1:1234567890:android:abc123`) |
| `FIREBASE_SERVICE_ACCOUNT` | JSON of a service account with App Distribution Admin role |

## Consequences

### Positive

- **No local-machine dependency.** Anyone with write access to the repo can mint a tester build by clicking "Run workflow" — no Android Studio, no signing config on their laptop.
- **Secrets stay in GitHub.** The keystore never leaves CI as a file; the Firebase service account never sits on a developer laptop. The app repo holds zero credentials.
- **Cheap and Play-Store-free.** Firebase App Distribution is free for the tester volumes we expect, and we don't pay the Play Internal Track latency tax for closed testing.
- **Backup artifact.** The 14-day workflow artifact means a Firebase outage doesn't block tester delivery — the APK can be downloaded from the run page and sideloaded.
- **Per-drop release notes.** The `release_notes` workflow input is shown in the Firebase tester UI, so testers know what each drop is supposed to validate.

### Negative

- **Manual trigger only.** We have to remember to dispatch the workflow; there is no "push to main → tester build" automation yet. This is intentional for the MVP (we don't want noisy tester builds), but will need revisiting once distribution cadence picks up.
- **Third-party action dependency.** `wzieba/Firebase-Distribution-Github-Action` is community-maintained, not first-party Google. If it goes unmaintained, we can fall back to the Firebase CLI (`firebase appdistribution:distribute`) — a small migration but not free.
- **Tester onboarding friction.** Firebase App Distribution requires each tester to accept an email invite, install a config profile / sign in with their Google account, and then sideload. Lower friction than Play closed testing, but higher than a TestFlight-style flow.
- **Keystore lives in one place.** If GitHub secrets are wiped, the keystore must be re-uploaded from a secure offline backup. Losing the keystore entirely would mean a new package ID for the next signed build.

## Alternatives Considered

### Google Play Internal Testing track

- **Pros:** First-party, well-known to testers, supports staged rollouts.
- **Cons:** Requires a Play Console account and an app listing, reviews can delay tester builds by hours, and locks signing to Play App Signing — heavier than what an MVP needs.
- **Verdict:** Rejected for the MVP. Revisit when we want a wider beta or are close to public launch.

### Manual APK email / Slack drop

- **Pros:** Zero infrastructure.
- **Cons:** No version tracking, no per-tester install telemetry, signing key would live on a developer laptop, no audit trail.
- **Verdict:** Rejected. Doesn't scale past two testers and creates a credential-hygiene problem.

### Fastlane + Firebase

- **Pros:** Industry-standard, declarative lane definitions, easy to extend to iOS later.
- **Cons:** Adds Ruby tooling and a `Fastfile` for what is currently a ~60-line GitHub Actions workflow. The marginal value over the current workflow is small.
- **Verdict:** Deferred. If we add iOS or need richer release orchestration (release-train branches, versioning), Fastlane becomes attractive.

## Mitigations

- The signing config in `app/build.gradle.kts` is gated on `KEYSTORE_PATH` being set, so local debug builds keep working without keystore env vars.
- The workflow artifact (14-day retention) is a documented fallback for Firebase upload failures.
- The keystore base64 should be backed up to a secure password manager outside GitHub, with at least two team members holding the recovery copy.
