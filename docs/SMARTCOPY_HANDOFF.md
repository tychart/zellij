# SmartCopy Handoff

## Purpose

This document is a handoff for continuing development of the `SmartCopy` feature on top of current `main` in the rebased branch `pr/GoodbyeNJN/4369`.

Target behavior:

- If there is **no active mouse selection**, `Ctrl+C` should behave normally and send ETX / `SIGINT` to the focused pane.
- If there **is an active Zellij mouse selection**, `Ctrl+C` should copy the selected text to the system clipboard.
- In the selection case, `Ctrl+C` should **not** be forwarded to the pane.
- Selection should **not** auto-copy on highlight. Copy should happen only when `SmartCopy` is triggered.

## Current Branch State

- Branch: `pr/GoodbyeNJN/4369`
- Default branch base after rebase: `main` at commit `ff0925f2`
- Rebased SmartCopy commits currently on the branch:
  - `cba6eb54` `feat(panes): add has_selection and reset_selection methods`
  - `c07b514b` `feat(actions): add SmartCopy action and update configration parsing logic`
  - `a0cb13eb` `feat(screen): add SmartCopy instruction and update related routing logic`
  - `e940c51f` `feat(example): update config for binding SmartCopy action in keybindings`

Important: the branch history contains the SmartCopy port, but there are still **unstaged working tree changes** that should not be lost. See "Unstaged Local Changes" below.

## Design Summary

The port was adapted to current `main` rather than reviving the old routing model from the original PR base.

The key architectural decision is:

- `SmartCopy` is a regular `Action`.
- It needs access to the original keyboard payload so that, when no selection exists, it can forward the exact original `Ctrl+C` bytes to the active pane.
- Because of that, `route_action(...)` was extended to accept:
  - `key_with_modifier: Option<KeyWithModifier>`
  - `raw_bytes: Option<Vec<u8>>`
  - `is_kitty_keyboard_protocol: Option<bool>`
- Only key-triggered action dispatch passes those values through.
- Manual action paths, tests, and plugin-triggered routes pass `None, None, None`.

## Core File Changes

### 1. Action definition and config parsing

`SmartCopy` is exposed as a bindable action.

Files:

- `zellij-utils/src/input/actions.rs`
  - Added `Action::SmartCopy`.
- `zellij-utils/src/kdl/mod.rs`
  - Added parsing support for `"SmartCopy"`.
- `zellij-utils/src/plugin_api/action.rs`
  - Serialization/deserialization support includes `Action::SmartCopy`.
- `zellij-utils/src/errors.rs`
  - `SmartCopy` was added to the screen error/context enum used for diagnostics.

### 2. Routing changes

`route_action(...)` on current `main` returns `Result<(bool, Option<ActionCompletionResult>)>` and participates in newer session/watcher behavior. The SmartCopy port preserves that.

Files:

- `zellij-server/src/route.rs`
  - `route_action(...)` signature was extended with the original key payload arguments.
  - `Action::SmartCopy` now routes to `ScreenInstruction::SmartCopy(...)`.
  - The `ClientToServerMsg::Key { ... }` dispatch path now preserves and passes:
    - `key`
    - `raw_bytes`
    - `is_kitty_keyboard_protocol`
  - Non-key/manual action paths pass `None, None, None`.
- `zellij-server/src/plugins/zellij_exports.rs`
  - `route_action(...)` call sites were updated to match the new signature.
- `zellij-server/src/unit/screen_tests.rs`
  - The local test helper `send_cli_action_to_server(...)` also needs the extra final `None` for the rebased signature.

### 3. Screen-level behavior

The actual SmartCopy behavior is implemented in screen instruction handling.

Files:

- `zellij-server/src/screen.rs`
  - Added `ScreenInstruction::SmartCopy(Option<KeyWithModifier>, Option<Vec<u8>>, Option<bool>, ClientId)`.
  - Added handling logic:
    - if the active tab has a selection for the client:
      - copy selection to clipboard
      - reset selection
    - otherwise:
      - forward original key bytes to the active pane or synced panes

Current behavior in `screen.rs` is the intended forward port and should be treated as the source of truth for the feature.

### 4. Pane / selection plumbing

SmartCopy relies on active-pane selection state being queryable independently of the legacy copy-on-select flow.

Files:

- `zellij-server/src/tab/mod.rs`
  - `Pane` trait default now includes `has_selection(...)`.
  - `Tab` has SmartCopy-facing helpers:
    - `reset_selection(client_id)`
    - `has_selection(client_id)`
    - `copy_selection(client_id)`
- `zellij-server/src/panes/grid.rs`
  - Existing grid selection helpers are reused:
    - `reset_selection()`
    - `has_selection()`
    - `get_selected_text()`
- `zellij-server/src/panes/terminal_pane.rs`
  - Delegates selection helpers to the terminal grid.
- `zellij-server/src/panes/plugin_pane.rs`
  - Delegates per-client selection helpers to the plugin grid.

## Example Config Changes

The feature is intentionally documented as an optional binding, not enabled by default.

Files:

- `example/config.kdl`
- `example/default.kdl`

These now contain commented examples such as:

```kdl
// bind "Ctrl c" { SmartCopy; }
```

This is only documentation/example wiring. It does not change default runtime behavior by itself.

## Rebase Notes

The original PR branch was based on an older commit line and had to be replayed onto current `main`.

Important rebase details:

- The current implementation intentionally preserves modern `main` concepts such as:
  - watcher-client handling
  - `ActionCompletionResult`
  - `NotificationEnd`
  - the current `ClientToServerMsg::Key { key, raw_bytes, is_kitty_keyboard_protocol }` contract
- The port does **not** try to restore the pre-current-main route/screen architecture.

## Unstaged Local Changes

There are currently unstaged local changes that matter for the handoff.

### A. `zellij-server/src/tab/mod.rs`

The rebased commit history currently contains an accidental large block of mouse-handling methods that was pulled in during conflict resolution.

The working tree currently removes that block.

What the local unstaged diff does:

- Deletes a large accidental block beginning right after `handle_mouse_event(...)`, including:
  - `handle_right_click(...)`
  - `handle_middle_click(...)`
  - `handle_mouse_no_click(...)`
  - `unselectable_pane_at_position(...)`
  - `focus_pane_at(...)`
  - `handle_right_mouse_release(...)`
  - `handle_middle_mouse_release(...)`
- Keeps only the intended SmartCopy-related additions in that area:
  - `reset_selection(client_id)`
  - `has_selection(client_id)`
  - `copy_selection(client_id)`

Interpretation:

- `git diff main...HEAD` still shows a large `tab/mod.rs` delta because that accidental block is in the rebased commit history.
- `git diff HEAD` shows the local cleanup that removes it.
- If this branch is moved elsewhere, this cleanup must be preserved or recommitted there.

### B. `zellij-server/src/unit/screen_tests.rs`

There are also local unstaged test additions that are not present in the rebased commit history.

The local changes do three things:

1. Update the local helper `send_cli_action_to_server(...)` to pass the extra trailing `None` required by the rebased `route_action(...)` signature.
2. Add a helper:

```rust
fn pty_write_instructions(log: &[PtyWriteInstruction]) -> Vec<PtyWriteInstruction>
```

3. Add two focused SmartCopy regression tests:

- `smart_copy_sends_ctrl_c_when_no_selection_exists()`
- `smart_copy_copies_selection_once_then_falls_back_to_ctrl_c()`

These tests validate the intended behavioral contract:

- Without a selection, SmartCopy forwards `Ctrl+C` to the PTY.
- With a selection, the first SmartCopy consumes the selection and clears it.
- The next SmartCopy falls back to forwarding `Ctrl+C` to the PTY.

Interpretation:

- These tests are important and should be carried forward.
- They are currently in the working tree, not in the rebased commit history.

## Validation State

### What was checked

- File-level diagnostics were checked during conflict resolution on the touched server files.
- After restoring the local SmartCopy test edits, file diagnostics were again clean.

### Environment blockers encountered during validation

The following environment issues were encountered locally:

1. `openssl-sys` initially failed because the machine did not have OpenSSL development files.
   - Resolved by installing `openssl-devel` and ensuring `pkg-config` could find OpenSSL.
2. `cargo fmt` was initially unavailable because the machine was using a distro Rust install without the repo-pinned components.
   - Resolved by installing and using `rustup`, then letting the repo toolchain activate.
3. `cargo xtask test` later failed because `protoc` was missing.
   - This is documented in `CONTRIBUTING.md`.
   - Install `protobuf-compiler` on Fedora to resolve it.

### Current expected environment for validation

At minimum, this environment should exist before rerunning validation:

```bash
sudo dnf install gcc openssl-devel pkgconf-pkg-config protobuf-compiler
source "$HOME/.cargo/env"
hash -r
```

The repo also expects the toolchain in `rust-toolchain.toml`:

- Rust `1.92.0`
- `rustfmt`
- `clippy`
- target `wasm32-wasip1`
- target `x86_64-unknown-linux-musl`

## Recommended Next Steps For The Next Agent

1. Preserve or commit the current unstaged cleanup in `zellij-server/src/tab/mod.rs`.
   - This is not optional if you want the branch to reflect only the intended SmartCopy delta.
2. Preserve or commit the current unstaged test additions in `zellij-server/src/unit/screen_tests.rs`.
3. Install `protobuf-compiler` locally if validation is being rerun on the same machine.
4. Run the narrowest useful validation first:

```bash
cargo test -p zellij-server smart_copy --lib
```

5. After that, if the environment is ready, run broader validation:

```bash
cargo xtask test
```

6. If this branch is being transplanted elsewhere, compare:

```bash
git diff main...HEAD
git diff HEAD
```

Why both matter:

- `main...HEAD` shows the rebased committed SmartCopy branch delta.
- `HEAD` shows the still-uncommitted cleanup and regression tests.

## Suggested Minimal Commit Plan

If continuing on this branch, the cleanest next step would be one follow-up commit containing only:

- the `tab/mod.rs` cleanup that removes the accidental mouse-handler block
- the `screen_tests.rs` helper signature fix
- the `pty_write_instructions(...)` helper
- the two SmartCopy regression tests

That would make the SmartCopy port self-contained and reduce the risk of losing the important local state.

## Quick File Checklist

Main committed SmartCopy port:

- `zellij-utils/src/input/actions.rs`
- `zellij-utils/src/kdl/mod.rs`
- `zellij-utils/src/plugin_api/action.rs`
- `zellij-utils/src/errors.rs`
- `zellij-server/src/route.rs`
- `zellij-server/src/screen.rs`
- `zellij-server/src/panes/grid.rs`
- `zellij-server/src/panes/terminal_pane.rs`
- `zellij-server/src/panes/plugin_pane.rs`
- `zellij-server/src/tab/mod.rs`
- `zellij-server/src/plugins/zellij_exports.rs`
- `example/config.kdl`
- `example/default.kdl`

Important local-only follow-up edits:

- `zellij-server/src/tab/mod.rs`
- `zellij-server/src/unit/screen_tests.rs`

## One-Sentence Summary

The SmartCopy feature is successfully forward-ported onto current `main`, but another agent should treat the current working tree, not just the commit history, as the authoritative state because the working tree contains the essential `tab/mod.rs` cleanup and the SmartCopy regression tests.
