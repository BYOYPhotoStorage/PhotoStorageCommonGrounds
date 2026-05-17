# Roadmap

_Last updated: 2026-05-16_

This document sequences post-MVP work. The MVP validated the core upload pipeline (credentials → local photo → B2 with thumbnail). The phases below move from "works when you tap" to "backs up your life automatically."

For the full feature brainstorm and architectural Q&A that shaped this plan, see [`ideas/core-features-brainstorm.md`](./ideas/core-features-brainstorm.md) and [`ideas/project-clarification-qa.md`](./ideas/project-clarification-qa.md).

---

## Phase 2: Auto-Upload & Resilience

The highest-impact gap right now: the user must manually tap each photo to upload it. This phase makes the app run silently in the background and recover from failures.

| Priority | Feature | Notes |
|----------|---------|-------|
| P0 | Background upload service | `MediaStore.ContentObserver` for real-time detection + `WorkManager` nightly scan as fallback. Required: foreground service with persistent notification. |
| P0 | Upload queue with retry | SQLite-backed queue, exponential backoff, resume across app restarts and reboots. |
| P1 | Duplicate detection | Level 1: filename + size + modified date. Level 2: SHA-256 hash stored in SQLite. |
| P1 | Notifications | Foreground upload progress, completion summaries, error alerts for repeated failures. |
| P2 | Wi-Fi-only upload toggle | Respects metered networks. Configurable in settings. |

**Validation before Phase 3:** Internal testers confirm photos taken while the app is backgrounded appear in B2 without manual action.

---

## Phase 3: Gallery & Sync

Once uploads happen automatically, the gallery needs to reflect both local and cloud state, and the SQLite index must survive reinstalls.

| Priority | Feature                 | Notes                                                                                                |
| -------- | ----------------------- | ---------------------------------------------------------------------------------------------------- |
| P0       | View mode switching     | Local only / Cloud only / Merged (default). See Q&A Q12.                                             |
| P0       | Context-aware deletion  | Local view deletes local only; Cloud view deletes cloud only; Merged deletes both with confirmation. |
| P1       | SQLite index sync to B2 | Nightly upload of the SQLite DB (only if changed). Enables reinstall recovery.                       |
| P1       | Reinstall recovery flow | On sign-in, download index from B2, then ask: "Catch up from [date] or full rescan?"                 |
| P2       | Thumbnail caching       | Cache B2-downloaded thumbnails locally to avoid repeated egress.                                     |

---

## Phase 4: Settings & Polish

Controls for power users and refinements for daily use.

| Priority | Feature              | Notes                                                                                                             |
| -------- | -------------------- | ----------------------------------------------------------------------------------------------------------------- |
| P0       | First-backup flow    | After onboarding, ask "Upload from today only" or "Upload entire gallery history" before auto-upload starts.      |
| P1       | Upload mode settings | Immediate / Scheduled (nightly) / Hybrid. See Q&A Q6.                                                             |
| P1       | Video handling       | Toggle for full-size vs. compressed upload, duration-based rules, option to disable.                              |
| P2       | Deletion strategy    | Configurable: never auto-delete, delete immediately after upload, delete after X days, delete after N new photos. |
| P2       | Cost dashboard       | Estimated B2 storage cost based on indexed photo count and size.                                                  |
| P3       | Sharing              | Time-limited B2 share links, album sharing.                                                                       |

---

## Out of Scope (for now)

These are tracked in [`ideas/core-features-brainstorm.md`](./ideas/core-features-brainstorm.md) but deliberately deferred:

- Client-side encryption (B2 at-rest is sufficient per Q&A Q3)
- Multi-device sync with conflict resolution
- Desktop companion app
- Face detection / on-device tagging
- RAW/DNG support
- Other S3-compatible providers (Wasabi, Storj, MinIO)
