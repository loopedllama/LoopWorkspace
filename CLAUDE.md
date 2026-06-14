# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

LoopWorkspace is the **umbrella Xcode workspace** for **Loop**, a DIY automated insulin delivery (AID) system for iOS/watchOS. This repo contains almost no source code of its own — it is a thin shell that aggregates the Loop app and its dependencies as **git submodules** and provides build/signing/release automation around them.

This checkout is a fork under the `loopedllama` org. The browser-build automation (see below) is designed to periodically sync the fork's branches from upstream `LoopKit/LoopWorkspace`.

> **Platform note:** This is an iOS/watchOS Xcode project. It can only be built, run, or tested on **macOS with Xcode** (CI uses Xcode 26.2 on `macos-26`). It cannot be compiled in a Windows/WSL/Linux environment — in those environments you can edit source, run git/submodule operations, and reason about the code, but not build or run tests.

## Submodule architecture — read this before editing code

All real code lives in submodules (all from the `LoopKit` org; see `.gitmodules`). The workspace pins each one to a **specific commit in detached-HEAD state**.

- **`Loop`** — the main app (UI, app lifecycle, the loop algorithm orchestration). Built on top of LoopKit.
- **`LoopKit`** — core frameworks: data storage/retrieval/calculation plus shared view controllers. Most other modules depend on it.
- **Device drivers** — `CGMBLEKit` + `G7SensorKit` (Dexcom CGM), `LibreTransmitter` (Libre CGM), `OmniBLE` + `OmniKit` (Omnipod pumps), `MinimedKit` + `RileyLinkKit` (Medtronic pumps, via RileyLink), `dexcom-share-client-swift`.
- **Services** — `NightscoutService` + `NightscoutRemoteCGM`, `TidepoolService`, `AmplitudeService`, `LogglyService`, `MixpanelService`.
- **Support / misc** — `LoopOnboarding`, `LoopSupport`, `Minizip`, `TrueTime.swift`.

**Editing workflow for submodule changes:** a code change almost always lives *inside* a submodule. To persist it you must (1) commit and push within that submodule's own repo/branch, then (2) commit the updated submodule pointer here in LoopWorkspace. Editing files in a submodule and committing only at the workspace level does **not** capture the code change. Submodule branch conventions are defined in `Scripts/define_common.sh` and `Scripts/sync.swift` (most modules track `dev`; device kits like `G7SensorKit`/`OmniKit`/`MinimedKit`/`LibreTransmitter` track `main`).

After switching branches or pulling, resync submodules with:
```
git submodule update --init --recursive
```

## Mac/Xcode build (developer path)

```
xed .                       # open LoopWorkspace.xcworkspace in Xcode
```

- **Always build the `LoopWorkspace` scheme**, never the `Loop` scheme — only the workspace scheme links all submodules correctly.
- Building to a simulator works with no signing changes. For a real device, set your Apple Developer team: in `LoopConfigOverride.xcconfig`, uncomment `LOOP_DEVELOPMENT_TEAM` and set your team id.
- Command line (macOS only) — build, test, and run a single test:
  ```
  xcodebuild build -workspace LoopWorkspace.xcworkspace -scheme LoopWorkspace \
    -destination 'platform=iOS Simulator,name=iPhone 16'

  xcodebuild test -workspace LoopWorkspace.xcworkspace -scheme LoopWorkspace \
    -destination 'platform=iOS Simulator,name=iPhone 16'

  xcodebuild test -workspace LoopWorkspace.xcworkspace -scheme LoopWorkspace \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    -only-testing:LoopTests/<TestClass>/<testMethod>
  ```
- Test targets live in the `Loop` submodule: `LoopTests` and `DoseMathTests` (each module also ships its own test target). Individual submodules have their own `*.xcworkspace`/schemes if you want to work on one in isolation.

## Build-time configuration (`.xcconfig`)

- **`LoopConfigOverride.xcconfig`** — local overrides: signing team, app name/bundle id, and feature flags via `SWIFT_ACTIVE_COMPILATION_CONDITIONS`. Default flags enable `EXPERIMENTAL_FEATURES_ENABLED`, `SIMULATORS_ENABLED`, `ALLOW_ALGORITHM_EXPERIMENTS`, `DEBUG_FEATURES_ENABLED`. This file is meant to be edited locally and includes `../../LoopConfigOverride.xcconfig` if present (so a parent dir can override).
- **`VersionOverride.xcconfig`** — `LOOP_MARKETING_VERSION` and `CURRENT_PROJECT_VERSION` for DIY builds.

## Browser / TestFlight build (Fastlane + GitHub Actions)

The non-developer path builds and ships to TestFlight entirely via GitHub Actions — there is no local Mac involved. This is the flow documented in `fastlane/testflight.md` (and LoopDocs "Browser Build"). The workflows live in `.github/workflows/` (`build_loop.yml`, `create_certs.yml`, `add_identifiers.yml`, `validate_secrets.yml`).

Fastlane lanes (`fastlane/Fastfile`), run with `bundle install` then `bundle exec fastlane <lane>`:

- `build_loop` — set team, bump build number, sign via `match`, archive `Loop.ipa`.
- `release` — upload the IPA to TestFlight.
- `identifiers` — create App Store Connect bundle IDs + capabilities.
- `certs` / `validate_secrets` / `nuke_certs` / `check_and_renew_certificates` — `match`-based code-signing cert management.

These lanes are driven by repository **secrets/variables**, not local config: `TEAMID`, `GH_PAT`, `FASTLANE_KEY_ID`, `FASTLANE_ISSUER_ID`, `FASTLANE_KEY`, `MATCH_PASSWORD`. Signing certs/profiles are stored in a separate `Match-Secrets` repo accessed via `GH_PAT`.

`build_loop.yml` also: (1) syncs the target branch from upstream `LoopKit/LoopWorkspace` (skipped when the owner is `LoopKit`), and (2) runs scheduled builds (monthly, or when upstream has new commits) gated by the `SCHEDULED_BUILD` / `SCHEDULED_SYNC` repo variables.

## Fork customizations via patches

`patches/` is applied at CI build time (`git apply ./patches/*`) before building — this is how a fork customizes Loop **without** forking every submodule. The `build_loop.yml` "Customize Loop" step also pulls optional named customizations from `loopandlearn/lnl-scripts` (`CustomizationSelect.sh`). To add a customization, drop a `.patch` in `patches/` or edit that workflow step.

## Localization

`Scripts/` holds the translation pipeline (Lokalise-based): `manual_*` scripts orchestrate export/upload/download/import/finalize, sharing config from `define_common.sh`. See `Scripts/LocalizationInstructions.md`. Loop's strings are managed as `.xcstrings` catalogs in the `Loop` submodule.

## Loop app architecture (in the `Loop` submodule)

The app is a coordination layer over LoopKit. Key managers in `Loop/Loop/Managers/`:

- **`LoopAppManager`** — app bootstrapping and lifecycle/state machine.
- **`LoopDataManager`** — the heart of the system: combines glucose, carb, and insulin histories to run the loop and produce dose recommendations.
- **`DeviceDataManager`** — owns and coordinates the active pump (`PumpManager`) and CGM (`CGMManager`) plugins; `DoseEnactor` issues doses.
- **`ServicesManager` / `RemoteDataServicesManager`** — optional external services (Nightscout, Tidepool, analytics, remote commands).
- **`SettingsManager`, `AlertManager`/`AlertMuter`, `NotificationManager`, `OnboardingManager`** — settings, alerting, notifications, onboarding.

Targets: `Loop` (app), `Loop Status Extension`, `Loop Widget Extension`, `Loop Intent Extension` (Siri/Shortcuts), `WatchApp` + `WatchApp Extension`, with shared `LoopCore` and `LoopUI` frameworks.

⚠️ This software controls insulin dosing. It is experimental and not approved for therapy. Treat any change touching dosing math, device communication, or safety limits with corresponding caution.
