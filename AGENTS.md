# AGENTS.md

## Purpose

This fork tracks official Zellij releases while carrying the SmartCopy feature. The goal is for the fork to remain interchangeable with upstream for deployment: the version, release tag, musl archive names, and checksum format should match the official release.

Current state:

- Upstream base: `v0.45.1`
- Fork release: `https://github.com/tychart/zellij/releases/tag/v0.45.1`
- Fork commit: `7476d0433`
- Main binary asset currently published: `zellij-x86_64-unknown-linux-musl.tar.gz`
- Matching checksum asset: `zellij-x86_64-unknown-linux-musl.sha256sum`

## What SmartCopy does

SmartCopy is an optional bindable action:

- If the focused pane has an active Zellij mouse selection, it copies the selection and clears it.
- If there is no selection, it forwards the original `Ctrl+C` bytes to the pane, preserving normal interrupt behavior.
- It is not enabled by default. Example configuration is documented in `README.md`, `example/config.kdl`, and `example/default.kdl`.

## Updating to a new upstream release

Do not begin by merging blindly into the existing fork branch. The fork has historically contained an experimental/rebased SmartCopy history and may have local worktree changes. Use this sequence:

1. **Inspect and preserve state.**

   ```bash
   git status --short --branch
   git remote -v
   git log --oneline --decorate -20
   git diff --binary > /tmp/zellij-local.patch
   git branch backup/pre-release-update-<version>
   ```

   Never discard local changes without confirming what they contain. SmartCopy-related changes in `route.rs`, `screen.rs`, and `unit/screen_tests.rs` may include completion plumbing and regression tests.

2. **Fetch the official tag.** If a local tag has the same name, fetch the tag under a temporary name to avoid clobbering it:

   ```bash
   git fetch upstream --tags --prune
   git fetch upstream refs/tags/v<version>:refs/tags/upstream-v<version>
   git show -s --format='%H %P %s' upstream-v<version>
   ```

3. **Start from the official release tag.** This keeps unrelated fork history out of the release:

   ```bash
   git reset --hard upstream-v<version>
   ```

4. **Replay only the feature commits.** The original SmartCopy implementation commits are:

   ```text
   cba6eb54  feat(panes): add has_selection and reset_selection methods
   c07b514b  feat(actions): add SmartCopy action and update configration parsing logic
   a0cb13eb  feat(screen): add SmartCopy instruction and update related routing logic
   e940c51f  feat(example): update config for binding SmartCopy action in keybindings
   ```

   Cherry-pick them individually so conflicts remain reviewable:

   ```bash
   git cherry-pick cba6eb54 c07b514b a0cb13eb e940c51f
   ```

   **Important:** Do not blindly cherry-pick `a493229c` or the old merge commit. That historical port included accidental large deletions/additions, stale generated assets, and a development handoff document. In the 0.45.1 update, the safe approach was to restore upstream versions of `zellij-server/src/tab/mod.rs`, `zellij-server/src/panes/grid.rs`, and `zellij-server/src/unit/screen_tests.rs`, then reapply only the intended SmartCopy methods, routing changes, and tests.

5. **Adapt to upstream API changes.** Check compiler errors rather than forcing old conflict resolutions. Common required adaptations include:

   - Preserve the current upstream `route_action` arguments and add SmartCopy's original key payload only where needed.
   - Add `None` for `os_input` in plugin/manual route calls when required by the current signature.
   - Keep passthrough dispatch tuples internally consistent when adding key/raw-byte fields.
   - Do not remove newer upstream `Screen::new` arguments or tests.
   - Avoid duplicate `Grid::has_selection` implementations if upstream has since added one.
   - Avoid restoring old mouse-handler methods that have moved into upstream's mouse handler module.
   - Keep the generated IPC Rust file synchronized with `common_types.proto`; use the current unused protobuf field number, not an old number that may now be occupied.

6. **Restore intentional local tests.** If saved local changes were present, reapply them carefully:

   ```bash
   git apply --3way /tmp/zellij-local.patch
   ```

   Resolve conflicts by preserving the current upstream file and manually adding only the completion plumbing and SmartCopy regression tests. Never accept an entire old `screen_tests.rs` conflict side because that can revert newer upstream test fixes.

7. **Format and validate.** Recommended order:

   ```bash
   cargo fmt --all -- --check
   cargo test -p zellij-utils --lib
   cargo test -p zellij-server smart_copy --lib
   cargo test -p zellij-server --lib
   ```

   The full server test suite has a timing-sensitive snapshot test (`send_cli_scroll_up_action`) that may fail once and pass on an immediate rerun. Record such a rerun explicitly; do not update snapshots unless the behavior change is intentional.

   The RHEL build environment may need:

   ```bash
   sudo dnf install gcc openssl-devel perl-IPC-Cmd perl-File-Compare perl-Time-Piece
   ```

   `protobuf-compiler` was unavailable in the RHEL 10 repositories used here. The checked-in generated protobuf sources allow the focused Rust tests to run without `protoc`; the full code-generation workflow may still require obtaining `protoc` separately.

8. **Review the diff before committing.** In particular, compare against the upstream tag, not only `HEAD`:

   ```bash
   git diff --check
   git diff upstream-v<version> --stat
   git diff upstream-v<version> --name-status
   ```

   The final diff should contain SmartCopy, its tests/documentation, and compatibility adaptations—not broad rewrites of panes, tests, generated plugin assets, or unrelated upstream code.

9. **Commit and update the fork.** After tests pass:

   ```bash
   git add -A
   git commit -m "feat: update v<version> with SmartCopy"
   git push --force-with-lease origin main
   ```

   A force push is expected when replacing the old fork branch with a clean upstream-release base. Verify the remote before pushing and use the saved backup branch for recovery.

## Building release assets

Official releases use exact target-based names. For the primary Linux deployment target:

```bash
cargo build --release --target x86_64-unknown-linux-musl

mkdir -p /tmp/zellij-release-v<version>
cp target/x86_64-unknown-linux-musl/release/zellij /tmp/zellij-release-v<version>/zellij
file /tmp/zellij-release-v<version>/zellij
(
  cd /tmp/zellij-release-v<version>
  tar -czf zellij-x86_64-unknown-linux-musl.tar.gz zellij
  sha256sum zellij > zellij-x86_64-unknown-linux-musl.sha256sum
)
```

To match the official checksum file exactly, the checksum line should name the build path:

```bash
printf '%s  target/x86_64-unknown-linux-musl/release/zellij\n' \
  "$(sha256sum /tmp/zellij-release-v<version>/zellij | awk '{print $1}')" \
  > /tmp/zellij-release-v<version>/zellij-x86_64-unknown-linux-musl.sha256sum
```

The musl binary should report as statically linked, for example:

```text
ELF 64-bit LSB pie executable, x86-64, ... static-pie linked
```

Upload assets to the existing release:

```bash
gh release upload v<version> --repo tychart/zellij \
  /tmp/zellij-release-v<version>/zellij-x86_64-unknown-linux-musl.tar.gz \
  /tmp/zellij-release-v<version>/zellij-x86_64-unknown-linux-musl.sha256sum \
  --clobber
```

Verify both the GitHub asset listing and a real download:

```bash
gh release view v<version> --repo tychart/zellij --json assets,url
curl -fsSL -o /tmp/zellij.tar.gz \
  https://github.com/tychart/zellij/releases/download/v<version>/zellij-x86_64-unknown-linux-musl.tar.gz
tar -tzf /tmp/zellij.tar.gz
```

The official workflow also publishes aarch64 Linux, macOS, Windows, and `no-web` variants. Add those targets only when the deployment system needs them; preserve the exact official naming convention.

## Creating the GitHub release

For a new version, push the tag after the commit:

```bash
git tag v<version>
git push origin v<version>
```

Create or update the release with `gh`:

```bash
gh release create v<version> --repo tychart/zellij \
  --title "Zellij <version> with SmartCopy" \
  --notes-file /tmp/zellij-release-notes.md
```

Upload the binary assets afterward if the fork's Actions release workflow does not run automatically. A release without the target-named archives is incomplete for deployment.

## Upstream PR prospects

A PR to `zellij-org/zellij` is realistic, but the current fork implementation should be treated as a working prototype rather than submitted unchanged. Before opening a PR:

1. Rebase the feature onto the current upstream `main`, not an old release.
2. Split the work into focused commits: pane selection API, action/config/IPC plumbing, screen routing/behavior, tests, and documentation.
3. Remove fork-specific handoff material, generated binary plugin assets, and unrelated conflict-resolution changes.
4. Ask upstream maintainers whether they prefer a generic action name/behavior, since changing `Ctrl+C` semantics can affect terminal users and clipboard expectations.
5. Add focused tests for selection copy, no-selection SIGINT passthrough, synced panes, kitty keyboard protocol, plugin/config round trips, and client completion behavior.
6. Run the upstream CI matrix and address formatting, clippy, MSRV, Linux/macOS/Windows, and no-web concerns.

The hardest parts are not the basic behavior; they are agreeing on the user-facing semantics and keeping the action compatible with Zellij's evolving routing, IPC, and selection architecture. A well-scoped PR with tests and no unrelated history should be manageable, but expect review on naming, default/config behavior, key payload propagation, and whether SmartCopy belongs in core or should be implemented as a plugin/config pattern.

Keep this file current whenever the release process, target matrix, dependency requirements, or upstream PR status changes.
