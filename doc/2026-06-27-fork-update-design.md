# 2026-06-27 Fork Update Design

## Goal

This fork keeps a small local patch set:

- The main WPF binary updates from `Mion354D/MaaAssistantArknights` fork releases.
- Game resources update from the original Maa resource source.
- Update UI is quiet: no routine update toasts, no post-update changelog window, and no external updater progress window.
- Failures remain explicit through task logs or a blocking updater failure message.

## Binary Update Flow

1. The WPF updater checks `https://api.github.com/repos/Mion354D/MaaAssistantArknights/releases`.
2. It scans recent releases and keeps the existing channel rules:
   - Stable accepts stable SemVer tags.
   - Beta accepts stable and beta tags.
   - Nightly accepts all newer tags.
3. It only selects an exact Windows OTA asset for the installed version and the target fork version.
4. If no exact OTA exists, it logs `NewVersionNoOtaPackage` and does not auto-select a full package.
5. If an OTA is found, the existing pending-update downloader and applier handle the package.

This intentionally avoids upstream's full OTA/mirror release system. The fork release must contain the exact OTA package required by the installed version.

## Resource Update Flow

When the update source is `Github` / overseas source:

1. The resource updater checks `https://raw.githubusercontent.com/MaaAssistantArknights/MaaResource/main/resource/version.json`.
2. If `last_updated` is newer than the local resource timestamp, it downloads `https://github.com/MaaAssistantArknights/MaaResource/archive/refs/heads/main.zip`.
3. It replaces the local `resource/` content from that original MaaResource package.

MirrorChyan resource updates are still available when selected, but this fork's target path is the original GitHub resource source.

## GitHub Actions Flow

The fork workflow `.github/workflows/fork-sync-build-release.yml` runs weekly and manually.

1. Checkout `dev-v2`.
2. Merge `upstream/dev-v2`.
3. Push the merged fork branch if the merge succeeds.
4. Build Windows x64 with the upstream CMake and WPF publish steps.
5. Publish a fork release with a stable date tag such as `v2026.6.27042`.
6. Generate OTA packages from recent fork releases.
7. Generate seed OTA packages from recent upstream releases in both the main upstream repository and `MaaRelease`.
8. Optionally generate one required OTA from a manually supplied `from_version`.

The upstream seed OTA step is what makes the first fork release usable without a previous fork release. The workflow derives the installed-version part of the OTA filename from the downloaded full package name, not only from the release tag. The manual `from_version` input is still available for the one-user case where the installed version is older than the automatic upstream/fork OTA history. If `from_version` is from a specific repository, set `source_package_repo` to the repository containing that full package.

## Quiet UI Policy

- First launch after update clears the update flag without opening the changelog window.
- Update-found, download-failed, resource-update, and MirrorChyan update messages are written to the task log instead of toast notifications.
- `MAA.Updater.exe` never opens the progress window.
- `MAA.Updater.exe` still shows a blocking failure message if applying an update fails.

## Owned Files

Fork-specific behavior is intentionally concentrated in:

- `.github/workflows/fork-sync-build-release.yml`
- `src/MaaWpfGui/Constants/MaaUrls.cs`
- `src/MaaWpfGui/Models/ResourceUpdater.cs`
- `src/MaaWpfGui/ViewModels/Dialogs/VersionUpdateDialogViewModel.cs`
- `src/MaaWpfGui/ViewModels/UserControl/Settings/VersionUpdateSettingsUserControlModel.cs`
- `src/MaaUpdater/main.cpp`

## Known Risks

- If upstream changes the same updater files, the scheduled merge may conflict and the workflow should fail explicitly.
- If the installed version is older than the generated fork/upstream OTA history and no manual `from_version` OTA was created, auto binary update will log that no OTA exists and skip the full package.
- The resource update tracks MaaResource `main`, so a breaking resource format change must still be handled by upstream compatibility.
