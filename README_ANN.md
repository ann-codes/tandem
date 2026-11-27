# README_ANN.md — Ann's Personal Setup & Sync Guide

This document captures custom environment configurations, troubleshooting history, build workarounds, and maintenance instructions for this fork of [Tandem](https://github.com/mikecann/tandem). It serves as a persistent reference for Ann and any AI coding agents working in this repository.

---

## 1. Repository & Remotes Overview

- **Fork (`origin`):** `https://github.com/ann-codes/tandem.git`
- **Upstream (`upstream`):** `https://github.com/mikecann/tandem.git`
- **Default Branch:** `main`

To inspect remotes at any time:
```bash
git remote -v
```

---

## 2. Syncing Fork with Upstream

**Last synced:** 27 Nov 2026, at upstream `30ef844` ("drag: keep privacy file lookups across drag previews, match owners in linear time"). Upstream's build scripts were unchanged; `Package.swift` gained a debug `-O` setting for TandemCore (a drag-lag fix), which needs no changes to the workarounds below.

To bring this fork up to date with the latest changes from Mike Cann's upstream repository:

### Step 1: Fetch upstream commits
```bash
git fetch upstream
```

### Step 2: Check commits ahead / behind
```bash
# See what upstream has that local does not:
git log --oneline HEAD..upstream/main

# See what local has that upstream does not:
git log --oneline upstream/main..HEAD
```

### Step 3: Ensure local changes are saved
```bash
git status
# If there are changes to keep, commit them first:
git add build-app.sh setup_mac.sh
git commit -m "Preserve CLT build fixes"
```

### Step 4: Merge upstream changes
```bash
git merge upstream/main
```

### Step 5: Push to your GitHub fork
```bash
git push origin main
```

### Step 6: Rebuild and reinstall
```bash
bash setup_mac.sh
bash install.sh
```

---

## 3. macOS Environment & Known Build Quirks

### Environment Specifications
- **Operating System:** macOS (Apple Silicon / arm64)
- **Developer Tools:** Apple Command Line Tools (`/Library/Developer/CommandLineTools`) without a full Xcode.app installation.
- **Swift Version:** Swift 6.3.3 (Apple Swift version 6.3.3.1.3 clang-2100.1.1.101).

### Issue 1: Obsolete `.private.swiftinterface` in Command Line Tools
- **Symptom:** `Invalid manifest` errors when running `swift build` or `setup_mac.sh`:
  - `extra argument 'swiftLanguageModes' in call`
  - `reference to member 'v15' cannot be resolved without a contextual type`
  - `extra argument 'traits' in call`
- **Root Cause:** A stale Swift 5.10 interface file (`arm64-apple-macos.private.swiftinterface`) remained in `/Library/Developer/CommandLineTools/usr/lib/swift/pm/ManifestAPI/PackageDescription.swiftmodule/`. SwiftPM prioritizes `*.private.swiftinterface` over public `*.swiftinterface`, incorrectly resolving the Package Description manifest API as Swift 5.10.
- **Workaround (automated in `build-app.sh`):**
  `build-app.sh` detects if the private interface exists. If so, it pre-compiles `PackageDescription.swiftmodule` directly from the valid Swift 6 interface into `.build/toolchain/ManifestAPI/` and exposes a wrapper `swiftc` that injects `-I "$TOOL_DIR/ManifestAPI"`.

### Issue 2: Missing `SwiftUIMacros` Plugin in Standalone CLT
- **Symptom:** Build fails when compiling `lottie-ios` with:
  ```text
  error: external macro implementation type 'SwiftUIMacros.EntryMacro' could not be found for macro 'Entry()'; plugin for module 'SwiftUIMacros' not found
  ```
- **Root Cause:** The `@Entry` macro implementation (`SwiftUIMacros`) is proprietary Apple software bundled exclusively inside `Xcode.app`, not in standalone Command Line Tools.
- **Workaround (automated in `build-app.sh`):**
  `build-app.sh` automatically patches the two `lottie-ios` files (`EpoxySwiftUIIntrinsicContentSizeInvalidator.swift` and `EpoxySwiftUILayoutMargins.swift`) to expand `@Entry` into standard SwiftUI `EnvironmentKey` definitions.

### Issue 3: Swift 6.3.3 Compiler Optimizer Assertion Failure
- **Symptom:** In release builds (`-c release`), the compiler crashes in the `CopyPropagation` SIL pass:
  ```text
  note: Invalid lifetime of value whose availability boundary is being visited.
  While running pass #2366342 SILFunctionTransform "CopyPropagation" on SILFunction ... workOutDrop ... TimelineLanesView.swift:1388:13
  ```
- **Workaround (configured in `setup_mac.sh`):**
  `setup_mac.sh` defaults to debug configuration:
  ```bash
  TANDEM_BUILD_CONFIGURATION="${TANDEM_BUILD_CONFIGURATION:-debug}" bash "$SCRIPT_DIR/build-app.sh"
  ```
  Debug configuration skips this specific SIL optimization pass and completes successfully.

---

## 4. Building, Installing & Running

### Full Build & Installation
```bash
# 1. Build the desktop app:
bash setup_mac.sh

# 2. Link CLI to ~/.local/bin and agent skills to ~/.claude/skills & ~/.agents/skills:
bash install.sh
```

### Launching Tandem
- **Desktop Application:**
  ```bash
  tandem app
  ```
  Or launch from Spotlight: `Tandem.app` (located at `~/Applications/Tandem.app`).

- **CLI Commands:**
  ```bash
  tandem help
  tandem status
  ```

- **MCP Server for AI Agents:**
  ```bash
  claude mcp add tandem -- ~/Applications/Tandem.app/Contents/MacOS/tandem mcp
  ```

---

## 5. Agent Instructions for this Codebase

When an AI agent works on this codebase:
1. Always preserve the CLT workarounds in [build-app.sh](/Users/annmini/code/tandem/build-app.sh) and [setup_mac.sh](/Users/annmini/code/tandem/setup_mac.sh).
2. Do not attempt to force `TANDEM_BUILD_CONFIGURATION=release` unless Xcode is installed or the Swift 6 compiler issue has been verified resolved in an updated toolchain.
3. When pulling from upstream (`mikecann/tandem`), check whether [build-app.sh](/Users/annmini/code/tandem/build-app.sh) or dependencies changed, and test the build with `bash setup_mac.sh`.
