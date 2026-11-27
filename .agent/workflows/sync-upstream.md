---
description: Check mikecann/tandem for upstream updates, assess CLT build risk, and (when approved) merge, rebuild, and update README_ANN.md.
---

# Tandem Upstream Sync Workflow

Use this workflow to check `mikecann/tandem` (`upstream/main`) against `ann-codes/tandem` (`origin/main`), assess risks for Ann's standalone Command Line Tools (CLT) setup, and sync when approved.

---

## 1. Quick Check & Early Exit (Read-Only)

Fetch both remotes without touching the working tree:

```bash
git fetch origin main --quiet && git fetch upstream main --quiet
git rev-list --count origin/main..upstream/main
```

- If the count is `0`: stop immediately and reply `Tandem is up to date with upstream/main (0 new commits).`
- Do not read other docs, run builds, or edit files on a `0`-commit check.

---

## 2. Inspect New Commits & CLT Risks (Read-Only)

Only run this section when `origin/main..upstream/main` has 1 or more commits:

1. Read `README_ANN.md` (Sections 2 and 3) for the last-synced commit and the three documented CLT build workarounds.
2. Run these read-only checks:
   ```bash
   git log --oneline origin/main..upstream/main
   git diff --stat origin/main...upstream/main -- build-app.sh setup_mac.sh install.sh Package.swift Package.resolved README_ANN.md
   git merge-tree --write-tree origin/main upstream/main >/dev/null 2>&1 && echo "MERGE_CLEAN" || echo "MERGE_CONFLICTS"
   ```
3. If any CLT-sensitive file changed (`build-app.sh`, `setup_mac.sh`, `install.sh`, `Package.swift`, `Package.resolved`), inspect its diff (`git diff origin/main...upstream/main -- <file>`) to see if it affects:
   - The `PackageDescription.swiftmodule` ManifestAPI workaround in `build-app.sh`
   - The `lottie-ios` `@Entry` macro patch in `build-app.sh`
   - The debug build default (`TANDEM_BUILD_CONFIGURATION=debug`) in `setup_mac.sh`
   - Swift tools version or new macro dependencies in `Package.swift` / `Package.resolved`

---

## 3. Report Findings (Read-Only Mode)

Present a concise report in chat:

1. **Summary:** Number of new upstream commits, date range, and main user-facing changes.
2. **Merge & CLT Risk Check:**
   - Whether `git merge-tree` reports a clean merge or conflicts.
   - Whether any CLT-sensitive files changed and what impact (if any) they have on Ann's local build workarounds.
3. **Next Step:** Stop here in read-only mode. Wait for Ann to confirm before merging, rebuilding, committing, or pushing.

---

## 4. Applying the Sync (Only When Approved)

Before making any changes, verify the checkout location and state:

1. **Root checkout check:** Confirm `pwd` is `/Users/annmini/code/tandem` on branch `main` with a clean working tree (`git status --short`).
   - Do **not** run `bash install.sh` from a temporary T3 worktree, because it symlinks `~/.local/bin/tandem` to the current repo path.
2. **Merge upstream:**
   ```bash
   git merge upstream/main
   ```
   - Preserve the CLT workarounds in `build-app.sh` and `setup_mac.sh`.
   - Do not force `TANDEM_BUILD_CONFIGURATION=release` unless full Xcode is installed or the Swift 6 SIL `CopyPropagation` crash is verified fixed.
3. **Rebuild, reinstall, and smoke-test:**
   ```bash
   bash setup_mac.sh
   bash install.sh
   ~/.local/bin/tandem help > /dev/null && echo "CLI OK"
   ls -la ~/Applications/Tandem.app/Contents/MacOS/
   ```
   - Note: `setup_mac.sh` replaces `~/Applications/Tandem.app`. Warn or pause if the build fails rather than leaving a broken state.
4. **Update `README_ANN.md`:**
   - Update the **Last synced** line in Section 2 with today's date, the latest `upstream/main` short hash and commit subject, and a short note on whether any build scripts or `Package.swift` changed.
5. **Commit & push (when requested):**
   - Commit the `README_ANN.md` update (and any conflict-resolution changes) and push `main` to `origin`.
   - Report the merged commit count, build/CLI verification result, and pushed commit range.
