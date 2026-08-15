# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Read this first

**`DESIGN.md` is the source of truth.** It records every architectural decision,
the reasoning, and — importantly — the alternatives that were already evaluated
and rejected. Read it in full before proposing anything. Do not re-litigate the
rejected options (DWM thumbnails, `PrintWindow`, C#/WPF, Tauri/Electron, a Rust
GUI framework, `WH_KEYBOARD_LL`) without genuinely new information.

## What this is

**flick** — a centered, hotkey-summoned grid of everything worth switching to on
this PC: running windows, pinned apps, pinned Explorer folders, browser tabs,
pinned bookmarks. Uniform, configurable tile size, scaling to 50+ items. Click to
launch or focus.

## Session context

This project was designed in a prior session running under WSL, and deliberately
handed off to **PowerShell** because the toolchain, the GUI process, and the
packaging commands are all Windows-native. Work from pwsh, not WSL.

- Repo: `R:\dev\flick`
- Toolchain: Rust `stable-x86_64-pc-windows-msvc` (1.89.0), already installed
- Target: **Windows 11 only** (dev machine is Win11 Pro 22631 / 23H2)
- Status at handoff: design complete, **no code written yet**. Next step is
  Milestone 1.

## Stack

Rust + the `windows` crate + Windows.UI.Composition + Windows.Graphics.Capture.
**No GUI framework** — the UI is a uniform grid of identical tiles, and a
framework would fight the D3D11 capture-texture pipeline. See `DESIGN.md` for the
full rationale.

## Build order

Milestone 1 is **dry run**: enumerate everything, render the grid with icons, but
every click is a no-op that logs what it *would* have activated. Dry-run mode is
ON by default. This validates reading the system before any code acts on it.

Milestones 2–5: activation → capture/previews → browser extension → drag-to-pin.

## Safety rules — non-negotiable

The user explicitly asked that this not break anything on their PC. These are not
suggestions:

1. **Never request elevation.** Manifest `asInvoker`, always. The app needs no
   privileged operation.
2. **Portable single exe.** Config beside the binary, cache in
   `%LOCALAPPDATA%\flick`. No installer, no scattered state.
3. **Read-only toward everything else.** Never write to a browser profile or
   another app's data. Only writes are flick's own config and cache.
4. **No `WH_KEYBOARD_LL`.** Use `RegisterHotKey` — process-scoped and released by
   the OS even on crash.
5. **Never block the UI thread on a shell call.** `IShellItemImageFactory` and
   `SHGetFileInfo` can block for seconds. Workers with timeouts, always.
6. **Panic hook + watchdog** that destroy/hide the window. An invisible topmost
   window swallowing clicks is the failure mode that feels like a broken PC.

Total persistent system footprint must stay limited to the four reversible items
listed in `DESIGN.md`. Anything that would add a fifth needs the user's sign-off.

## Building

```powershell
cd R:\dev\flick
cargo build
```

Do not use Windows Sandbox or a VM for testing — see `DESIGN.md`. The app's job is
enumerating *real* windows; a sandbox has nothing to enumerate.

## Open questions for the user

- Hotkey binding not yet chosen.
- Whether tabs and windows share one grid section or are separated.
- Tile size defaults, and how the grid reflows past ~50 items (fixed columns vs
  fit-to-screen).
