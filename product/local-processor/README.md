# local-processor

Self-throttling worker that drains the recorder's local SQLite buffer. The thin-client recorder
captures frames + audio but does **no** processing (OCR/ASR are off on the agent for CPU reasons);
this decoupled process does the heavy work and writes the resulting text back into the same DB.

Three tasks, one process:

| task | what | cost |
|------|------|------|
| `a11y_harvest` | copy `frames.full_text` (accessibility text the recorder already captured) → `ocr_text` | **free** (no CPU/GPU, no network) |
| `audio` | `audio_chunks` → GPU faster-whisper (`whisper_url`) → `audio_transcriptions`; silence is size-screened locally so the GPU only sees real speech | GPU, rate-capped |
| `ocr` | frames with no text → tesseract: live JPEG if the snapshot survives, else extract the frame from its compacted `video_chunk` by `offset_index` | CPU, capped; video-extract gated to catch-up |

## In-code caps (no systemd dependency)
- per-task **concurrency** + token-bucket **RPS**
- **CPU guard**: pauses any task while `os.getloadavg()[0]` exceeds the profile cap — hard
  non-exhaustion guarantee independent of the rate caps
- tesseract/ffmpeg pinned to 1 thread + `nice -n 19` + `ionice -c3`
- **scheduled decap**: `trickle` profile by default; switches to `catchup` during the night window
  *and* automatically when the backlog exceeds `catchup_if_backlog_over`. The CPU guard still bounds
  catch-up, so "decapped" never means runaway.

All knobs live in `processor.config.json` — edit and restart, no rebuild.

## Reliability
Single-instance `flock`, fully resumable (every cursor is DB state: `transcription_status`,
presence of an `ocr_text` row, and a side `state.db` for OCR attempt caps), retry caps, graceful
SIGTERM (finish batch, commit, exit), heartbeat `processor.progress.json`, structured log.
Safe to run live alongside the recorder (WAL + `busy_timeout`; only writes tables the recorder
doesn't: `ocr_text`, `audio_transcriptions`).

## Run
```bash
python3 processor.py                 # persistent
python3 processor.py --once          # one pass of each task (testing)
python3 processor.py --once --task audio   # a single task
```

## Dependencies
- `curl`, `ffmpeg`, `tesseract` (lang `eng`; install `tesseract-ocr-deu` for German screens)
- a reachable GPU whisper server at `whisper_url` (OpenAI-compatible `/v1/audio/transcriptions`)

## Not in git
Runtime artifacts (`*.log`, `*.progress.json`, `state.db`, `*.lock`) and any rescued transcripts
are deliberately git-ignored / kept out of the repo.

## Open
- Ship the processed text up to the central store (transcript table + text-ingest endpoint).
- Wrap as a systemd unit for auto-start/restart (deployment concern).
