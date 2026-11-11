# BhashaOS

**Local-first, offline note-taking with Kannada speech-to-text.**

Speak in Kannada → an on-device ASR model transcribes → text lands in a rich editor.
No cloud, no telemetry; your audio never leaves the machine.

**Flow:** mic → WebSocket (chunked audio) → transcription pipeline (chunk state
machine) → ONNX Runtime (CPU, isolated from the event loop) → notes in SQLite.

## Why

Commercial ASR speaks the "standard" register of a language. Real Kannada — from
Bellary to Mangalore, Dharwad to Gulbarga — does not. BhashaOS is built on a
research hypothesis: **jointly training ASR with dialect identification (DID)
improves recognition for underrepresented dialects.**

The bundled model is a Conformer-CTC student distilled from an IndicWav2Vec
teacher, carrying a 5-way Kannada dialect head trained jointly with CTC
(λ_ctc = 1.0, λ_did = 0.5).

### Model card (summary)

| | |
|---|---|
| Task | Kannada CTC ASR (beam-10 prefix search / greedy) + 5-way dialect ID |
| Student | Conformer-CTC encoder (16 layers, 176-d) + 176→256 projection |
| Teacher | IndicWav2Vec (95M) + linear + 8-block Conformer (256-d) |
| Dialects | D1 Central (Bellary) · D2 Coastal (Mangalore) · D3 Dharwad · D4 NE (Gulbarga) · D5 Mysore |
| Export | ONNX opset 17, static graph, 30 s window @ 16 kHz mono, float32 |
| Test | CER 7.75 · WER 37.63 · DID accuracy 78.25 (macro-F1 78.18) |

Full provenance (vocab, blank id, dialect order, export validation, sha256):
[`backend/models/kannada_asr/metadata.json`](backend/models/kannada_asr/metadata.json).
`samples/` holds golden fixtures — real audio with reference transcripts — used by tests.

## Stack

- **Frontend:** React · TypeScript · Tailwind CSS · Tiptap editor
- **Backend:** Python · FastAPI (uvicorn ASGI) · Pydantic v2
- **Persistence:** SQLite · SQLAlchemy 2.0 async (aiosqlite) · Alembic migrations
- **Inference:** ONNX Runtime (CPU, quantized, isolated from the ASGI event loop)

## Repository layout

```text
backend/
  models/kannada_asr/     model contracts + golden fixtures (weights hosted on HuggingFace)
  src/bhasha/             api · schemas · services · inference · db · core
  tests/
frontend/                 React + TypeScript app
bhashaos_data/            runtime: SQLite DB, audio blobs, exports (git-ignored)
```

## Getting the model weights

Model weights are open-sourced on the HuggingFace Hub — one model repo per
language — and kept **out of git** (binaries don't belong in git history). This
repo carries the *contracts*: each model's `metadata.json` (vocab, dialect map,
expected sha256) and golden fixtures. A fetch script downloads pinned revisions
and verifies hashes before the app ever loads a model.

- Kannada ASR: `<huggingface repo URL>` *(link added once published)*

## Status

**M0 — bootstrap.** Roadmap: M1 database layer → M2 API skeleton → M3 file
transcription → M4 streaming pipeline → M5 editor UI → M6 Docker + polish.
