# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Read this first

**`DESIGN.md` is the source of truth.** It records every architectural decision,
the reasoning, and — importantly — the alternatives that were already evaluated
and rejected. Read it in full before proposing anything. Do not re-litigate the
rejected options (DWM thumbnails, `PrintWindow`, C#/WPF, Tauri/Electron, a Rust
GUI framework, `WH_KEYBOARD_LL`) without genuinely new information.

**`STATUS.md`** says where the build stands, what is verified, and what is next.
Read it before starting work.

## What this is

**DashPick** — a centered, hotkey-summoned grid of everything worth switching to on
this PC: running windows, pinned apps, pinned Explorer folders, browser tabs,
pinned bookmarks. Uniform, configurable tile size, scaling to 50+ items. Click to
launch or focus.

## Session context

This project was designed in a prior session running under WSL, and deliberately
handed off to **PowerShell** because the toolchain, the GUI process, and the
packaging commands are all Windows-native. Work from pwsh, not WSL.

- Repo: `R:\dev\dashpick`
- Toolchain: Rust `stable-x86_64-pc-windows-msvc` (1.89.0), already installed
- Target: **Windows 11 only** (dev machine is Win11 Pro 22631 / 23H2)
- Status: **Milestones 1, 2 and 5 done, plus type-to-filter.** Activation works.
  Taskbar pins, sections, and the general parsing-name target model landed with
  M2. Drag-to-reorder, the right-click menu, unpinning and `IDropTarget` landed
  with M5, ahead of order. Typing filters the grid, and arrows plus Enter came
  with it. Milestone 4 is part done: browser tabs and favicons arrive over a
  loopback WebSocket from the MV3 extension in `extension/`, and clicking a tab
  switches to it. Verified in real Chrome. Bookmarks are not built.
- **There is no edit mode.** One was built and removed the same session. See
  DESIGN.md, "Edit mode: built, then removed". Do not reintroduce a mode without
  new information; a drag threshold plus a context menu is the settled answer.
- **The pushpin and drop-to-pin are slated for removal.** Roku's call. Read the
  first two steps in `STATUS.md` before touching either.

## Stack

Rust + the `windows` crate + Windows.UI.Composition + Windows.Graphics.Capture.
**No GUI framework** — the UI is a uniform grid of identical tiles, and a
framework would fight the D3D11 capture-texture pipeline. See `DESIGN.md` for the
full rationale.

## Build order

Milestones 1, 2 and 5 are done. See `STATUS.md` for where things stand and what
is next; `DESIGN.md` has the full milestone list and the reasoning.

`dry_run` was ON through Milestone 1 and defaults OFF since Milestone 2. It stays
in config as a debugging switch: clicks log what they would do and do nothing.

Remaining: bookmarks over the same channel → capture previews. Type-to-filter was pulled ahead of both, because 40 tabs would
flood the grid without it.

## Safety rules — non-negotiable

The user explicitly asked that this not break anything on their PC. These are not
suggestions:

1. **Never request elevation.** Manifest `asInvoker`, always. The app needs no
   privileged operation.
2. **Portable single exe.** Config beside the binary, cache in
   `%LOCALAPPDATA%\dashpick`. No installer, no scattered state.
3. **The browser socket is off by default and fails closed.** Loopback only, and
   it refuses every connection unless the `Origin` is allowlisted *and* the
   token matches. The origin check is what stops a web page enumerating open
   tabs; the token is what stops a local process. Never weaken either, never
   ship a default allowlist. See `DESIGN.md`, "Who is allowed on the socket".
4. **Read-only toward everything else.** Never write to a browser profile or
   another app's data. Only writes are DashPick's own config and cache. DashPick does
   write `dashpick.toml` now (tray pickers add pins); that goes through `toml_edit`
   so hand-written comments and formatting survive.
5. **No `WH_KEYBOARD_LL`.** Use `RegisterHotKey` — process-scoped and released by
   the OS even on crash.
6. **Never block the UI thread on a shell call.** `IShellItemImageFactory` and
   `SHGetFileInfo` can block for seconds. Workers with timeouts, always.
7. **Panic hook + watchdog** that destroy/hide the window. An invisible topmost
   window swallowing clicks is the failure mode that feels like a broken PC.

Total persistent system footprint must stay limited to the four reversible items
listed in `DESIGN.md`. Anything that would add a fifth needs the user's sign-off.

## Building

```powershell
cd R:\dev\dashpick
cargo build
```

Do not use Windows Sandbox or a VM for testing — see `DESIGN.md`. The app's job is
enumerating *real* windows; a sandbox has nothing to enumerate.

## Settled

- Hotkey: `alt+``. `ctrl+alt+space` was picked first but is already registered by
  something else on this machine.
- Grid: fixed tile size, panel grows from center, caps at 80% of the work area,
  scrolls past that.
- Sections stack with text headers, configured in `dashpick.toml`. A section
  takes a list of sources; the default is two, `["windows", "tabs"]` over
  `["taskbar", "manual"]`. A header plus a whole row per section was most of the
  panel on a quiet machine. Split them back out with `match`.
- Pins and running windows are shown separately, never merged. See `DESIGN.md`.
- Filtering hides, it never reorders, and the panel's width is frozen while a
  query is live. Both are about keeping tile positions stable — the same rule
  that fixed the tile size. See `DESIGN.md`, "Type-to-filter".

## Open questions for the user

- Bookmarks arrive with the extension (Milestone 4); whether they get their own
  section or merge into an existing one is still open. Tabs are settled: they
  share the `Active` section with the windows.
