# Thin-client / fat-host architecture (revised 2026-05-30)

> Supersedes the OCR-on-agent assumption baked into M1/M2 and `headless-split.md`.
> Decision owner: Manar (2026-05-30). The agent **captures only**; **all** heavy
> processing (OCR, ASR/transcription, embedding, indexing, PII redaction) runs on
> the self-hosted central host.

## Why this changed

The deployed reference recorder on the Berlin workstation
(`screenpipe-recorder.service`, fork bin, data dir `/mnt/data/screenpipe-agent`,
monitors 596+445, audio on) burned **~74% of one core sustained + ~2.1 GB RSS for
4 h**. Root-caused on the live process:

- **Sustained burn = on-client audio.** `--audio-transcription-engine parakeet`
  (default) ran VAD + segmentation + ASR continuously over a gappy Focusrite
  Scarlett 2i2 output stream. Health showed `chunks_received: 1723`,
  `vad_rejected: 1723` (every chunk non-speech), `transcriptions_completed: 0`,
  `total_words: 0` — i.e. it produced **nothing useful** while pegging a core, and
  the `source_buffer` gap-filler spammed `large gap … inserting silence` every
  ~170 ms.
- **OCR = bursty, minor.** Tesseract ran on nearly every captured frame
  (`ocr_text` ≈ `frames`, because Linux AT-SPI a11y coverage is thin so the
  `has_accessibility_text` fast-path rarely fires), but at `capture_fps_actual ≈
  0.06` (one frame every ~15 s) it is NOT the sustained cost.

Both are pure waste on the client when the product goal is central search: the
employee PC should be a dumb capturer.

## Immediate relief applied (2026-05-30) — ASR off, capture kept

Lever: `record --audio-transcription-engine disabled` ("audio capture only, no
speech-to-text"). Wired into the env role
(`roles/services/screenpipe-app/`) via `recorder_transcription_engine: disabled`
in `vars/main.yml`, rendered in `screenpipe-recorder.service.j2`. Deployed with
`./ansible.sh -l desktop -t screenpipe-app` (recorder restarted).

**Measured before/after on the live process:**

| | before (parakeet) | after (disabled) |
|---|---|---|
| steady-state CPU (60 s window) | ~74% of a core | **8.7% of a core** |
| RSS | ~2.1 GB | ~0.55 GB |

**Capture confirmed intact after the change:**
- Frames still captured: `frames` 953→1001, `frames_db_written` climbing,
  `frame_status: ok`, `last_frame_timestamp` current.
- **Raw audio still captured + stored on disk:** `audio_chunks` 1713→1734, mp4
  audio files 1828→1857, `chunks_received` climbing, `last_audio_timestamp`
  current. `--audio-transcription-engine disabled` keeps the recording path
  (chunks written to disk) and only short-circuits VAD-heavy STT
  (`reconciliation.rs` returns early when the engine is `Disabled`).

This is the THIN-CLIENT audio behavior: capture + store raw audio, zero on-client
transcription.

## Capture-only feasibility verdict

**Audio: clean.** `--audio-transcription-engine disabled` is a first-class CLI
value. Raw `.mp4`/wav audio chunks are still captured and written to disk +
`audio_chunks` table; only STT is skipped. Verified live. No source change, no
rebuild.

**Screen frames: feasible today, but OCR cannot be cleanly turned off by a flag.**
- The JPEG snapshot is written to disk **before and independent of** OCR
  (`paired_capture.rs`: snapshot `write()` at the top; OCR is a later, separate
  step). So frames are always retained.
- OCR gating is **a11y-driven, not flag-driven**: OCR runs when
  `!has_accessibility_text || a11y_is_thin` (or for terminal apps). There is **no
  CLI flag and no env var** to globally skip OCR while keeping frame capture
  (confirmed by grepping the whole tree: no `DISABLE_OCR`/`SKIP_OCR`/`NO_OCR`).
- The only existing screenshot-skip path is `screenshot_disabled` (power-profile
  FullPause/AudioPaused) which ALSO stops writing the JPEG — not what we want.

**Consequence for "zero OCR on client":** at the current capture cadence
(~0.06 fps) Tesseract OCR is a minor bursty cost, so for the immediate relief we
**leave OCR as-is** (the audio fix already reclaimed ~65 points of CPU). To get
**true zero-OCR on the client** (the stated thin-client goal) requires a small
source change — add a `screen-only` / "raw-capture" gate in `screenpipe-engine`
that, when set, skips the OCR block in `paired_capture` and the a11y tree walk,
storing the JPEG + minimal metadata (timestamp, monitor, app/window if cheap)
only. This is a Cargo feature or a runtime flag (preferred: a runtime
`--ocr-engine off` style flag or `SCREENPIPE_DISABLE_OCR=1` env, gated in
`paired_capture::paired_capture`). Medium effort, no new crate. **Recommend doing
this when M2 is reworked to ship raw frames** (below) — until then OCR on the
client is harmless overhead, and its output is simply ignored/re-done centrally.

## Revised data flow (thin client → fat host)

```
EMPLOYEE PC (thin)                         CENTRAL HOST (fat, self-hosted)
─────────────────                          ──────────────────────────────
screenpipe record                          ingest API (per-agent token, HTTPS)
  • screen → JPEG snapshots on disk   ──▶     • store raw JPEG (object store)
  • audio  → mp4/wav chunks on disk   ──▶     • store raw audio chunk
  • NO OCR (target; flag-gated TODO)         processing workers (server-side):
  • NO ASR (done: engine=disabled)             • OCR (tesseract / better engine)
  • NO embedding                               • ASR (whisper-server we already
  • NO PII redaction                             run on desktop:9000, or central)
                                                • PII redaction (text + image)
sync-shim (raw pusher)                          • embedding (e5-small, 384-dim)
  • cursor = frames.id + audio_chunk id         • FTS + vector index
  • ships RAW frame JPEG + RAW audio       ◀── ack → advance cursor
    bytes (+ minimal metadata), NOT
    OCR text, NOT thumbnails-only
                                           Postgres(+pgvector) + object store
                                           viewer (per-person web UI) reads here
```

## How M1 (central) changes

The central server stops being a "store the agent's OCR" sink and becomes the
**processing host**. Concretely:

1. **Ingest accepts raw artifacts, not derived text.**
   - New/extended `POST /ingest` payload per frame: `frame_id` (agent cursor),
     `timestamp`, `monitor`, `app_name/window_name/browser_url` (cheap metadata
     the recorder already has without OCR), and the **full-resolution JPEG bytes**
     (base64 or multipart) — NOT `ocr_text`, NOT a pre-downscaled thumbnail.
   - New audio ingest: `audio_chunk_id`, `timestamp`, device, and the **raw audio
     chunk bytes**. (Audio was never ingested before — this is net-new on
     central.)
2. **Server-side processing workers** (async, off the ingest hot path — enqueue on
   ingest, process in a worker):
   - **OCR worker:** run tesseract (or a better engine — we control the host now,
     can use GPU/larger models) over each stored JPEG → `ocr_text`. The desktop
     already runs a GPU box; OCR can be heavier/better than on-client Tesseract.
   - **ASR worker:** transcribe each audio chunk. We **already self-host
     faster-whisper** at `desktop:9000` (`/v1/audio/transcriptions`, OpenAI-compat,
     GPU, multilingual) — the central pipeline can POST chunks to it (or run its
     own whisper). Reuse, don't rebuild.
   - **PII redaction worker:** redact text (and optionally black out image PII)
     server-side, before indexing/serving. This is where Manar's "redaction is
     central" decision lands. Strong legal posture: raw lands on OUR host, redacted
     before it's searchable/viewable.
   - **Embedding worker:** e5-small (already in central), `passage:` prefix, store
     vector. Unchanged except now fed by the server-side OCR/ASR output.
   - **Thumbnail derivation:** central now generates the viewer thumbnail from the
     stored full-res JPEG (downscale 960px q60) — this moved off the agent.
3. **Storage:** object store now holds **full-res frame JPEGs + raw audio chunks**
   (much larger than thumbnails-only). Retention TTL (`CENTRAL_RETENTION_DAYS`)
   becomes more important — and may want a tiered policy (keep raw N days, keep
   derived text/thumbnail longer). Sizing + object-store backend (local FS vs
   S3/minio) is an environment-lane decision; flag it.
4. **Search/viewer endpoints** are largely unchanged — they read `ocr_text`,
   embeddings, and thumbnails, all of which now exist because the server produced
   them. The viewer gains audio-transcript surfacing once ASR runs centrally.

## How M2 (sync-shim) changes

The shim stops being an "OCR-text + thumbnail" pusher and becomes a **raw artifact
pusher**:

- **Two cursors:** `frames.id` (screen) AND the audio chunk id (audio is now
  shipped too — previously the shim ignored audio entirely).
- **Ships raw bytes:** read `frames.snapshot_path` JPEG and send it **full-res**
  (central downscales); read the audio chunk file and send it raw. NO on-agent
  downscale, NO on-agent OCR read (the `ocr_text` table may be empty once the
  client OCR is gated off — the shim must not depend on it).
- **Bandwidth/cost note:** shipping full-res JPEGs + raw audio is materially more
  network + central storage than the old "thumbnail + text". Two mitigations to
  decide with Manar: (a) ship a moderately-compressed JPEG (not thumbnail, not
  lossless) — the recorder already writes ~q-balanced JPEGs; (b) audio chunking
  cadence/codec. The current 30s audio chunks as mp4 are already compact.
- Still resumable (advance cursor on ack), batched, HTTPS + per-agent token —
  those mechanics carry over from the existing shim.

## Open questions for Manar

1. **True zero-OCR on the client?** Audio ASR is off (big win, done). OCR is still
   running on the client (minor at 0.06 fps). Do you want me to add the
   source-side flag (`SCREENPIPE_DISABLE_OCR` / `screen-only` feature) to make the
   client truly OCR-free, or is "leave the harmless OCR, just don't use its
   output" fine until M2 raw-frame rework? (Recommend: add the flag when we rework
   M2, not as an urgent rebuild.)
2. **Raw storage + bandwidth.** Shipping full-res frames + raw audio is much
   heavier than the old thumbnail+text. OK to ship moderately-compressed JPEG
   (e.g. existing recorder JPEG, ~q-balanced) rather than full lossless? Audio:
   keep 30s mp4 chunks as-is? And the central retention TTL for RAW vs DERIVED —
   one TTL or tiered?
3. **Central ASR engine.** Reuse the existing self-hosted faster-whisper at
   `desktop:9000` from the central pipeline, or run a separate whisper on the
   central box? (Reuse is least work and already GPU-backed.)
4. **Central OCR engine.** Stick with Tesseract server-side, or use something
   better now that we control the host (GPU OCR / a VLM)? Affects quality + cost.
5. **Where does central run?** This reshapes infra sizing (now CPU/GPU-heavy:
   OCR+ASR+embedding for the whole fleet on one host). That's environment's lane —
   but the thin-client decision means the central box needs real compute, not just
   storage. Flag to environment.
```
