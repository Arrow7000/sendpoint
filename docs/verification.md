# Verification

GitHub Actions **Verify** (`.github/workflows/verify.yml`) is the authoritative automated gate for the reviewed revision. It runs on pull requests, pushes to `main`, and manual dispatch. All three independent jobs must pass; configure branch protection to require `behavior`, `screenshots`, and `packaging` if enforcement is wanted.

The workflow uses standard `macos-15` Apple Silicon runners, explicitly selects `/Applications/Xcode_26.2.app`, and verifies arm64 and Swift 6.2. GitHub's [runner reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners) maps `macos-15` to arm64 (`macos-15-intel` is separate); the [image inventory](https://github.com/actions/runner-images/blob/main/images/macos/macos-15-arm64-Readme.md) lists Xcode 26.2. Images can change: a missing toolchain is a setup failure, not permission to silently use a different version.

## Review a run

Open **Actions → Verify → run**, confirm the revision (PR runs normally test GitHub's merge revision), and review each job and its artifacts:

- **behavior-evidence**: `check.json` stage outcomes/test counts and `check.log` from `./check.sh`, including verification-script tests, Swift build, and behavior tests. Screenshot tests may be skipped here by design; the separate rendering job covers them.
- **screenshots-evidence**: `shots.json`, `shots.log`, and all rendered light/dark PNGs from `./shots.sh`. Inspect the screens affected by the change, including both appearances. This is a fresh render, **not a baseline comparison**: the JSON comparison stage should report skipped because recording was requested. A green render alone does not establish visual correctness.
- **packaging-evidence**: toolchain, build, bundle-validation, linked-library, and signature logs plus `Sendpoint-ci.zip` when assembly and validation succeeded. CI compiles Metal (downloading Apple's Metal toolchain if missing), builds release with `CODESIGN_IDENTITY=-`, validates the plist, executable architecture, resources, Sparkle framework/symlinks, and deep signatures, then uses `ditto` to archive. It extracts and verifies the archive again to check that framework symlinks and signatures survive.

Every job attempts evidence upload even after failure. Missing evidence or an interrupted/cancelled job is not a pass; cancellation or infrastructure failure can prevent upload. Artifacts expire after 14 days. The app is inside a `ditto` ZIP rather than uploaded as a loose bundle so symlinks and executable metadata are preserved. It is an untrusted CI build, not a published release; do not install or launch it on a developer Mac without explicit approval.

## Rerun remotely

For an unchanged revision, use the run's **Re-run failed jobs** or **Re-run all jobs** menu. For a new revision, push the fix to the PR and review the new run. For manual runs, use **Actions → Verify → Run workflow**, selecting the intended branch; the workflow must first exist on the default branch for manual dispatch to be available in the UI. Confirm the resulting revision rather than treating an older green run as acceptance.

No local build, tests, screenshots, install, or launch is required. Local diagnostics are optional and explicitly requested. CI does not call `install.sh` or `ship.sh` and does not launch the app; packaging success is not a launch-smoke result.

## Manual acceptance and release limits

Deterministic tests and isolated snapshot renders are not full-app end-to-end testing. With explicit approval on a real Mac, relevant changes still need checks of microphone permission and recording, Accessibility grants and selection from other apps, pasted export/focus behavior, actual downloaded-model transcription, and the updater. Report which manual checks were performed and which remain outstanding.

The CI bundle is ad-hoc signed, with no signing secrets, notarization, DMG, Sparkle feed signing, publishing, or release/updater acceptance. Release procedures remain separate (`release.sh`).
