# Mobile Testing Framework Comparison

## Goal

Choose a UI/end-to-end testing approach for the Photo Storage Android app that supports:

- Driving the main app’s UI (onboarding, gallery, backup, delete dialogs).
- Interacting with system surfaces outside the app (camera roll / MediaStore, network state).
- Running on physical devices and CI emulators.
- Remaining maintainable by a small team without dedicated QA automation engineers.

This document compares the leading options and explains why the project is leaning toward a hybrid stack rather than a single framework.

## Comparison Criteria

| Criterion | Description |
|-----------|-------------|
| **Android native support** | Works with XML/ViewBinding, Activities, WorkManager, and `minSdk 33`. |
| **Cross-app support** | Can interact with other apps, system dialogs, and the camera roll. |
| **Media injection** | Ability to add photos/videos to the device for the app to discover. |
| **Network control** | Ability to toggle offline/online state during a test. |
| **Cloud assertion** | Ability to verify state in an S3/B2 bucket independently. |
| **DB inspection** | Ability to read the app’s SQLite database. |
| **Setup cost** | Effort to get the first test running. |
| **Maintenance cost** | Effort to keep tests passing as the UI evolves. |
| **CI friendliness** | How well it runs in GitHub Actions / Firebase Test Lab / emulators. |
| **Team fit** | How well it matches the existing Kotlin/Android skill set. |

## Frameworks Evaluated

### 1. Maestro

Open-source YAML-based mobile testing framework from Mobile.dev.

| Criterion | Rating | Notes |
|-----------|--------|-------|
| Android native support | ✅ Excellent | Uses accessibility hierarchy; works with any native Android app. |
| Cross-app support | ⚠️ Limited | Can tap system permission dialogs, but not deep cross-app flows. |
| Media injection | ❌ No | Cannot write to `MediaStore`. Needs a companion app or `adb`. |
| Network control | ❌ No | Cannot toggle airplane mode directly; issue [#1507](https://github.com/mobile-dev-inc/maestro/issues/1507) requests `adb shell` support. |
| Cloud assertion | ❌ No | No S3/B2 client; needs external harness. |
| DB inspection | ❌ No | No filesystem/DB access. |
| Setup cost | 🟢 Very low | `curl | bash` install; YAML flows need no compilation. |
| Maintenance cost | 🟢 Low | Smart waits and retries reduce flakiness. |
| CI friendliness | 🟢 Good | CLI-based, easy in GitHub Actions. |
| Team fit | 🟢 Good | Readable by non-Android engineers. |

**Best for:** Fast UI smoke tests, onboarding/gallery flows, permission dialogs.

**Not suitable alone for:** Media injection, offline testing, DB/cloud verification.

### 2. Espresso + UI Automator

Google’s official Android testing stack. Espresso runs in-process for the app under test; UI Automator runs at the system level.

| Criterion | Rating | Notes |
|-----------|--------|-------|
| Android native support | ✅ Excellent | First-party, integrates with Android Studio and Gradle. |
| Cross-app support | ✅ Yes (with UI Automator) | UI Automator handles system UI and other apps. |
| Media injection | ⚠️ Partial | Test code can write to `MediaStore` directly, but it lives inside the test APK. |
| Network control | ⚠️ Partial | Can execute `adb` commands from a host-side test runner. |
| Cloud assertion | ❌ No | Needs external harness. |
| DB inspection | ✅ Yes | Can read Room/SQLite DB from the test process. |
| Setup cost | 🟡 Medium | Requires Gradle, test APK, and IdlingResources. |
| Maintenance cost | 🟡 Medium | More code than Maestro; selector fragility if UI changes. |
| CI friendliness | 🟡 Medium | Native to Android CI but slower to build and run. |
| Team fit | 🟢 Excellent | Pure Kotlin/JUnit, matches the app stack. |

**Best for:** Deep Android-native integration tests, DB assertions, complex UI state.

**Trade-off:** More powerful than Maestro but slower to author and maintain.

### 3. Appium

Industry-standard cross-platform automation using the WebDriver protocol.

| Criterion | Rating | Notes |
|-----------|--------|-------|
| Android native support | ✅ Good | Uses UIAutomator2 driver under the hood. |
| Cross-app support | ✅ Yes | Inherits UIAutomator2 capabilities. |
| Media injection | ⚠️ Partial | Can push files via `adb` but not inject into `MediaStore` automatically. |
| Network control | ⚠️ Partial | Can execute `adb` commands from the test runner. |
| Cloud assertion | ❌ No | Needs external harness. |
| DB inspection | ❌ No | No direct DB access from Appium. |
| Setup cost | 🔴 High | Server, drivers, desired capabilities, locator strategy. |
| Maintenance cost | 🔴 High | Slower tests, more flakiness, more configuration. |
| CI friendliness | 🟡 Medium | Supported everywhere but heavy. |
| Team fit | 🟡 Medium | Requires WebDriver/Appium expertise; team is Android-native. |

**Best for:** Cross-platform teams (Android + iOS) with dedicated automation engineers.

**Trade-off:** Overkill for an Android-only app. Slower and more complex than the alternatives.

### 4. Kaspresso

Kotlin DSL wrapper around Espresso and UI Automator with built-in logging and screenshot capture.

| Criterion | Rating | Notes |
|-----------|--------|-------|
| Android native support | ✅ Excellent | Built on Espresso/UI Automator. |
| Cross-app support | ✅ Yes | Uses UI Automator for system interactions. |
| Media injection | ⚠️ Partial | Same as Espresso/UI Automator. |
| Network control | ⚠️ Partial | Same as Espresso/UI Automator. |
| Cloud assertion | ❌ No | Needs external harness. |
| DB inspection | ✅ Yes | Same as Espresso. |
| Setup cost | 🟡 Medium | Slightly more setup than raw Espresso but cleaner DSL. |
| Maintenance cost | 🟡 Medium | Less boilerplate than raw UI Automator. |
| CI friendliness | 🟡 Medium | Same as Espresso. |
| Team fit | 🟢 Excellent | Kotlin-first, matches the app. |

**Best for:** Teams that want a cleaner Kotlin API than raw Espresso/UI Automator.

**Trade-off:** Less ecosystem support than Maestro; more code than Maestro.

### 5. Detox

Grey-box E2E framework popular in the React Native community.

| Criterion | Rating | Notes |
|-----------|--------|-------|
| Android native support | ⚠️ Limited | Designed for React Native; native Android support exists but is second-class. |
| Cross-app support | ❌ No | Focused on in-app flows. |
| Media injection | ❌ No | Not a strength. |
| Network control | ❌ No | Not a strength. |
| Cloud assertion | ❌ No | Needs external harness. |
| DB inspection | ❌ No | Not a strength. |
| Setup cost | 🟡 Medium | Requires Espresso test orchestrator. |
| Maintenance cost | 🟡 Medium | RN-centric docs and community. |
| CI friendliness | 🟡 Medium | Works but not optimized for native apps. |
| Team fit | 🔴 Poor | Team is not using React Native. |

**Verdict:** Not a fit for this project.

## Summary Matrix

| Framework | UI drive | Cross-app | Media injection | Network control | Cloud assert | DB inspect | Setup | Maintenance |
|-----------|----------|-----------|-----------------|-----------------|--------------|------------|-------|-------------|
| Maestro | 🟢 | 🟡 | ❌ | ❌ | ❌ | ❌ | 🟢 | 🟢 |
| Espresso + UI Automator | 🟢 | 🟢 | 🟡 | 🟡 | ❌ | 🟢 | 🟡 | 🟡 |
| Appium | 🟢 | 🟢 | 🟡 | 🟡 | ❌ | ❌ | 🔴 | 🔴 |
| Kaspresso | 🟢 | 🟢 | 🟡 | 🟡 | ❌ | 🟢 | 🟡 | 🟡 |
| Detox | 🟡 | ❌ | ❌ | ❌ | ❌ | ❌ | 🟡 | 🟡 |

Legend: 🟢 strong, 🟡 possible with extra work, ❌ not supported or poor fit.

## Recommended Approach

No single framework covers all the requirements. The recommended approach is a **hybrid stack**:

- **Maestro** for UI driving.
- **Companion Android app** for MediaStore injection and cleanup.
- **`adb` shell commands** for network control and DB extraction.
- **Cloud harness (Python/Kotlin)** for bucket verification.
- **Orchestration runner** to sequence everything.

This gives the team the speed and readability of Maestro for UI tests while using the right tool for media, network, and cloud assertions.

## When to Reconsider

- If the UI flows become highly dynamic or conditional, migrate them to **Kaspresso** or **Espresso + UI Automator**.
- If the project later adds iOS and wants a single test codebase, evaluate **Appium** or **Maestro Cloud** for cross-platform coverage.
- If the team grows a dedicated automation engineer, a pure **Espresso + UI Automator + cloud harness** suite may be worth the extra maintenance for deeper control.

## Sources

- [Maestro on GitHub](https://github.com/mobile-dev-inc/maestro)
- [Maestro documentation](https://docs.maestro.dev/)
- [Maestro CLI reference — DeviceLab](https://devicelab.dev/blog/maestro-cli-complete-reference)
- [Maestro vs Appium — Revyl](https://revyl.com/blog/maestro-vs-appium/)
- [Best Android testing tools 2026 — Drizz](https://www.drizz.dev/post/best-android-testing-tools)
- [Best mobile test automation frameworks 2026 — Drizz](https://www.drizz.dev/post/best-mobile-test-automation-frameworks-2026-when-to-choose-drizz)
- [Best mobile testing tools 2025 roundup — Maestro](https://maestro.dev/insights/best-mobile-testing-tools-2025-roundup)
- [GitHub issue: running `adb shell` from Maestro](https://github.com/mobile-dev-inc/maestro/issues/1507)
