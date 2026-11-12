# Hacivat patches on zed

This branch, `hacivat/912159c`, is
`zed-industries/zed@912159c7ad77ce6296a93be77efc298b0b00fd50` plus the patches listed below.
The Hacivat project carries them for its use of the Apache-2.0 GPUI crates (`gpui`,
`gpui_platform` and the platform crates they pull in). Hacivat pins this branch by commit.

Nothing outside `crates/gpui*` is changed, apart from this file. Zed's GPL-3.0 crates are
neither modified nor used by Hacivat.

## License notice (Apache-2.0, section 4)

- **4(a):** `LICENSE-APACHE` is kept unchanged at the repository root. At this base, zed has
  no `NOTICE` file. If a later base adds one, it is carried unchanged (4(d)).
- **4(b):** every modified file carries a header comment saying that the Hacivat project
  modified it, what changed and which zed commit the original work comes from.
- **4(c):** all copyright, patent, trademark and attribution notices of the original files
  are kept.
- The modified files stay under Apache-2.0. The Hacivat changes are contributed under the
  same license.

## Patches

### 1. `gpui_macos`: discard marked text on `FocusLost`

- **Commit:** "gpui_macos: discard marked text on FocusLost (Hacivat carry)".
- **File:** `crates/gpui_macos/src/window.rs` (header comment added, section 4(b)).
- **Origin:** `crates/ui/hacivat-ui-core/patches/gpui-discard-marked-text-on-focus-lost.patch`
  in the Hacivat repository.
- **What it does:** it implements `PlatformWindow::text_input_state_changed` for
  `MacWindow`. At this base, macOS uses the trait's default no-op. On
  `TextInputStateChange::FocusLost` only, it calls
  `-[[view inputContext] discardMarkedText]`. The call is deferred to the foreground
  executor, so it runs after the current draw's focus listeners.
- **Why:** when focus moves from one text input to another inside a window, the app commits
  its marked (IME composition) text in `on_blur`. Without this patch the macOS input method
  keeps its own copy of the composition and replays it into the next input. AppKit's
  contract is that the client unmarks first and then tells the input method to drop its
  copy. The deferral gives that order.
- **What it does not do:**
  - It never acts on `ContentChanged` or `SelectionChanged`. GPUI sends those after every
    IME edit, so acting on them would end every composition.
  - An app switch keeps GPUI focus and sends no `FocusLost`, so the input method keeps
    owning the composition there, as intended.
  - A focus move while the window is inactive is not covered. GPUI does not deliver the
    blur, or send `FocusLost`, for an inactive window. That needs a change in `gpui`
    itself; Hacivat schedules it for a later patch on this branch.
  - On its own it does not cover a direct move from one text input to another. Patch 2
    makes GPUI send `FocusLost` for that move.
- **Changes from the carried patch file:** none in behaviour. The `use gpui::{…}` import
  list was reflowed by `rustfmt`, because the patch file's hunk was not rustfmt-formatted.
- **Verified:** `cargo check -p gpui_macos` and `cargo clippy -p gpui_macos`, both without
  warnings, with Rust 1.98.1 (this base's `rust-toolchain.toml` channel) on
  aarch64-apple-darwin.

### 2. `gpui`: report `FocusLost` then `FocusGained` on a move between text inputs

- **Commit:** "gpui: report FocusLost then FocusGained on a text input to text input move
  (Hacivat carry)".
- **Files:**
  - `crates/gpui/src/window.rs` (header comment added, section 4(b)). The behaviour change
    is in `Window::draw`.
  - `crates/gpui/src/input.rs` (header comment added). Tests only.
  - `crates/gpui_macos/src/window.rs`. A comment in patch 1's `text_input_state_changed`
    now names this case. No code change.
- **Origin:** `crates/ui/hacivat-ui-core/patches/gpui-focus-lost-on-input-to-input-move.patch`
  in the Hacivat repository.
- **What it does:** at this base, `Window::draw` calls
  `PlatformWindow::text_input_state_changed` only when the window's focused text input goes
  from Some to None (`FocusLost`) or from None to Some (`FocusGained`). When focus moves
  straight from one text input to a different one within a frame (Some(a) to Some(b)), it
  sent nothing. This patch sends `FocusLost` and then `FocusGained` for that move.
  - Some to None and None to Some behave as before: one notification each.
  - Some(a) to Some(a), a redraw or a refocus of the same input, still sends nothing.
- **Why:** without it, patch 1 never ran for an input-to-input move. Example: a
  programmatic focus move from a settings search box, mid-composition, to the command
  palette's input. The macOS input method kept the composition and its candidate window
  stayed open over the new input.
- **Effect on other platforms at this base:** Linux and Windows use the trait's default
  no-op. `gpui_web` and `gpui_ios` now end the old input's session and start the new one's,
  which matches what they already do for a Some to None to Some sequence across two frames.
  Hacivat does not build either.
- **Tests:** `input::tests::focus_move_between_text_inputs_reports_loss_then_gain` is new.
  `input::tests::text_notifications_require_a_focused_input_and_track_field_changes` moves
  focus between two text inputs, so its expected sequences now include the
  `FocusLost`, `FocusGained` pair.
- **Verified:** `cargo test -p gpui --features test-support --lib` (423 passed, 2 ignored)
  and `cargo clippy -p gpui -p gpui_macos --features gpui/test-support --tests`, without
  warnings, with Rust 1.98.1 on aarch64-apple-darwin. Without the `window.rs` change, both
  tests above fail.
