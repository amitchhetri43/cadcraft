# CADCraft local-only build: security audit

Audit date: 2026-11-14. Source: fork `amitchhetri43/cadcraft` at `ddfa833` (identical to
`storytold/cadcraft` main at audit time). Branch with changes: `local-only/remove-updater`.

## Verdict

CADCraft is a low-risk app. It is pure Rust with `unsafe_code = "forbid"` workspace-wide, no
telemetry, no analytics, no crash upload and no cloud accounts. The **only** code that contacted
the internet on its own was the built-in self-updater. That has been removed on this branch, along
with the entire HTTPS/TLS stack it pulled in. A 25-second socket capture of the rebuilt app showed
only `127.0.0.1` (the optional control port) and no outbound connections.

## Findings

| # | Area | Finding | Risk | Action |
|---|------|---------|------|--------|
| 1 | Self-updater (`craft-update`) | On every launch it called `api.github.com` (User-Agent `CADCraft/<version>`), and could download a release archive and replace the app binary. Exposes IP and version to GitHub; is a code-replacement path. | Medium | **Removed**: dependency, startup check, self-update helper, Help ▸ Check for Updates… |
| 2 | HTTPS stack | `ureq`, `rustls`, `ring`, certificate verifiers, `zip`, `tar` existed only for the updater. | Low | **Removed** (47 crates dropped from `Cargo.lock`). |
| 3 | `rustix` 1.1.5 | RUSTSEC-2026-0486: xattr functions mark unwritten bytes initialised on a zero-length buffer. | Low | **Bumped to 1.1.6.** |
| 4 | `zbus_names`, `zvariant`, `zvariant_derive` | RUSTSEC-2026-0370/0371/0372: unmaintained notices. | None on macOS | Linux/BSD-only dependency (`cfg(all(unix, not(target_os = "macos")))`); not compiled into the Mac app. |
| 5 | Control channel (`--control PORT`) | Binds `127.0.0.1` only; every request needs a per-session token from an owner-only file (`~/.cache/cadcraft-control-<port>.token`, mode 600). Off unless started with `--control`. | Low | None. Note: anything holding the token can read/write any file you can (upstream known limit). |
| 6 | MCP bridge (`crates/mcp`) | Connects only to `127.0.0.1:<port>` and checks the peer address before sending the token. | Low | None. |
| 7 | AutoLISP | Can read/write files you can access; cannot run programs or use the network. `acad.lsp`/`acaddoc.lsp` load only from `<settings>/support`, never from a drawing's folder (tested). | Low | None. Only load LISP you trust. |
| 8 | Drawing files (DXF/DWG) | Treated as hostile input; no `unsafe`; panics are guarded. DWG is parsed in a separate worker process (the app's own binary), which has no memory cap yet. | Low | None. A crafted DWG can use a lot of memory for that one open. |
| 9 | External links | Discord / getartcraft.com / contributor GitHub links open in your browser **only when clicked**. | Informational | Kept. Remove `native_menu.rs` / `dialogs.rs` / `credits.rs` links if you want zero outbound links. |
| 10 | Local files | Autosave folder mode 700; token file 600; logs and `support` are 755 inside `~/Library` (which macOS keeps private). Crash reports are written locally and never uploaded. | Low | None. |
| 11 | Process launches | Only the DWG worker (own binary) and `open -a CADCraft` from the MCP bridge. | Low | None. |

## What was checked

- Every network-capable crate in `Cargo.lock` traced to its dependent (`ureq` → `craft-update`;
  `webbrowser` → egui link clicks; `tokio` → wasm build only; `open` → tests only).
- All non-test Rust for sockets, `bind`, URL opening, process spawning, `unsafe`, upload/download
  and telemetry keywords.
- 558 crates.io dependencies against the OSV advisory database.
- Runtime: rebuilt app launched in an isolated profile, driven over the control channel (draw,
  inspect, render, menu list), sockets sampled every second for 25 s.

## Not covered

- Git dependencies from `storytold/craft-libs`, `storytold/egui` and `storytold/winit` were not
  line-by-line reviewed; they were checked for network crates only (none apart from `craft-update`).
- No fuzzing of the DXF/DWG parsers beyond the project's own hostile-input tests.
- The upstream release binary is signed by "Learning Machines LLC"; the local build is ad-hoc signed.

## Staying local

- Updates are now manual: pull upstream into the fork, review, rebuild.
- Rebuild: `cargo build --release -p cadcraft`, then replace
  `~/Applications/CADCraft.app/Contents/MacOS/CADCraft` and re-sign:
  `codesign --force --sign - --options runtime --entitlements packaging/macos/entitlements.plist ~/Applications/CADCraft.app`.
