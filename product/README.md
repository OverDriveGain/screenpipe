# product/ — Manar's screen-activity monitoring product

This dir holds OUR product code built on the vendored screenpipe fork (the rest of this repo).
Kept separate so it never tangles with upstream crates and survives `git pull` of the fork.

Layout:

- `docs/` — design notes.
  - `headless-split.md` — headless agent vs GUI run-mode design + plan (M0).
- `sync-shim/` — **M2 (built)** agent-side pusher. Standalone Rust binary (own workspace,
  excluded from the fork workspace). Reads the agent's local `db.sqlite` read-only (cursor =
  `frames.id`), downscales each frame's on-disk snapshot JPEG to a thumbnail, batches OCR text +
  thumbnail + metadata, pushes over HTTPS+token to the central ingest. See `sync-shim/README.md`.
- `central-server/` — **M1 (built)** self-hosted ingest API + Postgres(+pgvector) + thumbnail
  object store + full-text/vector/hybrid search. FastAPI + docker-compose. See
  `central-server/README.md`.
- `viewer/` — M4 per-person web UI over the central store (not started — WAVE 2).

## Key data-model finding (2026-05-30)

The agent stores each captured frame as a **per-frame JPEG snapshot on disk**
(`frames.snapshot_path`), NOT as video — `video_chunks` is empty in a screen-only
run. So the shim never decodes video: it reads the JPEG, downscales, ships a
thumbnail. This is exactly the "never ship raw video" constraint and makes M2
much simpler than a video-extraction approach.

## End-to-end proven (2026-05-30, WAVE 1)

Live Berlin agent SQLite (`/mnt/data/screenpipe-m0-final`, 26 frames) → sync-shim →
`POST /ingest` → Postgres (26 frames, 26 OCR, 25 embeddings) → search returns
ranked hits (fts + vector + hybrid) with thumbnail URLs. Idempotent re-run =
0 accepted / 26 duplicates. See incidents in the home base.

The fleet **agent itself** is the fork's `screenpipe` binary
(`crates/screenpipe-engine` bin target), built from source — not separate code here. See
`docs/headless-split.md`.

Roadmap and locked decisions: see the top section of the repo-root `CLAUDE.md`.
