# Headless vs GUI split — design & plan

Goal: the fork supports two run modes — a **headless capture agent** (no GUI; records + OCRs +
serves `:3030`; the cross-platform fleet agent) and the **GUI** (desktop preview). This doc is the
investigation result + recommendation. **No heavy build has been kicked off** — plan first.

## Key finding (corrects a prior agent-file gotcha)

The fork **already ships a headless binary**: `crates/screenpipe-engine/src/bin/screenpipe-engine.rs`,
declared in `crates/screenpipe-engine/Cargo.toml` as `[[bin]] name = "screenpipe"`. It is the full
CLI — `record` (captures, OCRs, and serves `:3030` via `SCServer`), `status`, `search`, `sync`,
`team`, `pipe`, `audio`, `vision`, `mcp`. CLI defs live in `crates/screenpipe-engine/src/cli/`.

The Tauri GUI (`apps/screenpipe-app-tauri/src-tauri`) does **not** shell out to that binary. It
depends on `screenpipe-engine` as a **library** and runs the recorder + `SCServer` **in-process**
via `recording::spawn_screenpipe`. `tauri.conf.json` has `externalBin: []` — literal: no bundled
sidecar. The word "sidecar" in the GUI source is legacy naming for the in-process engine.

**Consequence:** a fork-version headless agent is essentially **already implemented**. The work is
to *build* the `screenpipe` bin (it's only ever been built as a library inside the GUI so far),
verify `record` serves + captures headless on Linux + Windows, and trim it to a screen-only profile.
This also fixes the schema-incompatibility root cause: a fork-built agent writes the *same* schema
(`...20260520180000`) as the GUI and our read tooling, unlike the older published npm CLI 0.3.350
which can't open it.

## Option A — build the fork's `screenpipe` bin as the headless agent  ✅ RECOMMENDED

Build the existing `[[bin]] screenpipe` target, run `screenpipe record --port 3030 --data-dir …`
headless as the fleet agent.

**Crates/APIs it already uses** (from the bin's `use` block): `screenpipe_engine::{SCServer,
ResourceMonitor, vision_manager::VisionManager, start_ui_recording, ...}`, `screenpipe_db::DatabaseManager`,
`screenpipe_screen::monitor::list_monitors`, and the `screenpipe_engine::cli::*` handlers. The
`record` path constructs the DB, starts capture/vision, and starts `SCServer` (axum) on `--port`.

**Build implications on `linux-build-fixes`:**
- Engine default features = `["qwen3-asr", "parakeet", "redact-onnx-cpu"]`. `qwen3-asr` is exactly
  what triggers the `liblibopenblas` symlink workaround (LINUX_BUILD_NOTES §3) and downloads large
  ASR models on first run. `redact-onnx-cpu` is *forced* in (engine unconditionally imports
  `screenpipe_redact::adapters::onnx`).
- For a **screen-only agent** audio/ASR is dead weight (employees run audio-disabled). Two levers,
  in order of effort:
  1. **Cheapest, do first:** build with defaults (eat the openblas symlink), run
     `screenpipe record --disable-audio …`. ASR code compiles in but never executes. Proves the
     agent end-to-end with zero source changes.
  2. **Then trim:** add a `screen-only` Cargo feature on `screenpipe-engine` that drops
     `qwen3-asr` + `parakeet` (keep `redact-onnx-cpu` since the engine import forces it, unless we
     also gate that import). Smaller binary, no openblas symlink, no ASR model download. Requires
     verifying the engine compiles without the audio ASR features — the `audio` CLI subcommand and
     any unconditional ASR calls would need `#[cfg]` gating. Medium effort; do only after mode 1
     proves the loop.
- The `find_tools.js` Bun patch is GUI-build-only; irrelevant to the Rust bin.
- Build command (fast profile): `cargo build -p screenpipe-engine --bin screenpipe --profile release-dev`.
  First build is slow (whole dep tree incl. ffmpeg/onnx); subsequent are incremental.

**Windows:** the bin is cross-platform (the `#[cfg(target_os="macos")]` blocks degrade cleanly).
Windows MSVC link is noted as clean in `Cargo.toml` (bswap_shim). Verify in M3, not now.

**Pros:** true fork-schema headless agent; schema-matched to GUI + read tooling (kills the silent-
hang landmine); one source of truth (the engine); minimal/no new code for mode 1.
**Cons:** first build is heavy; trimming audio cleanly (mode 2) needs `#[cfg]` work.

## Option B — drive the Tauri app headless via a flag

Run the GUI binary in a no-window background mode (skip `WebviewWindow` creation, still call
`spawn_screenpipe`), GUI preview when launched normally.

**Pros:** reuses the GUI's settings/store and `spawn_screenpipe` orchestration.
**Cons:** drags the entire Tauri/webkit/GTK runtime onto every employee PC as a headless service
(heavy, fragile under systemd/no-session); the GUI is excluded from the workspace and built with
`bun tauri build` (the find_tools Bun patch, signing, bundling) — a far heavier and less portable
artifact than a plain Rust bin; "headless Tauri" fights the framework. Worse fit for a fleet agent.

## Recommendation

**Option A.** The headless binary already exists; we just build and harden it. Path:
1. Build `screenpipe` bin with defaults on `linux-build-fixes` (`release-dev`), with the openblas
   symlink in place.
2. Run `screenpipe record --disable-audio --data-dir <fresh dir> --port 3030` on the Berlin desktop;
   verify `:3030/health` = 200 AND `screenpipe status --data-dir <dir>` frame count climbs (close M0
   on a fork-built agent instead of the published npm CLI).
3. Once stable, add a `screen-only` feature to drop audio/ASR (mode 2) for a lean fleet binary.
4. Package per M3 (Win + Linux, autostart, token enrollment).

> Do NOT kick off the first full `cargo build` without Manar's go-ahead — it's a long heavy build
> (full dep tree incl. ffmpeg/onnx). Surface this plan first.

## How M2 (central-sync shim) hooks in — same either way

The shim is **independent of the run mode** because it reads the agent's local state, not the engine
internals:

- **Read side:** open the agent's `db.sqlite` read-only (WAL, while the agent writes) OR call the
  local `:3030` API. The DB cursor is `frames.id` (autoincrement). Tables: `frames`
  (`id, video_chunk_id, offset_index, timestamp`), `ocr_text` (`frame_id, text`). The `search` CLI
  subcommand documents itself as reading `~/.screenpipe/db.sqlite` read-side via WAL with no daemon —
  proof the read-side is safe concurrent.
- **Push side:** track last-synced `frames.id`; batch new rows (OCR text + a thumbnail per frame, NOT
  raw video); POST over HTTPS with a per-agent token to the M1 central ingest; advance the cursor on
  ack (resumable).
- **Implement as a separate small process** (Rust or a script) under `product/` — keep it OUT of the
  engine so it doesn't couple to the fork's release cycle or the heavy build. It only needs the
  schema (which is stable per migration) and the token/endpoint config.

## Open questions for Manar

1. **Audio: drop entirely?** Plan assumes screen-only (`--disable-audio`, later a `screen-only`
   feature). Confirm no audio/meeting transcription is wanted on employee PCs (also the cleaner
   legal posture).
2. **PII redaction:** `redact-onnx-cpu` is on by default (downloads ~280MB ONNX model on first run,
   redacts PII in frames/text). Keep it on the agent (privacy-by-default, good legal optics) or do
   redaction centrally? Affects build trim and first-run footprint.
3. **Read side for M2:** prefer reading the local SQLite directly (lighter, no daemon dependency) or
   going through the `:3030` API (cleaner contract, survives schema changes)? Leaning direct-SQLite.
4. **Bin rename:** the bin is literally named `screenpipe`. For the fleet do we ship/rename it (e.g.
   `mz-agent`) to avoid colliding with any installed npm `screenpipe` on dev machines? Cosmetic, M3.
