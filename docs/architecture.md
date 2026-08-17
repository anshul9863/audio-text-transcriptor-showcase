# Architecture

How Audio Text Transcriptor is put together, and why it is built this way.

## The shape of the problem

Transcription has an awkward performance profile for a web app. A single request
can occupy a CPU for minutes, produces output gradually, and cannot be
meaningfully parallelised within one file. Three consequences follow, and they
drive most of the design:

1. The work cannot happen inside the request handler, or the server stops
   answering anything else.
2. The client must not wait for a single response, or the user stares at nothing
   for minutes.
3. Progress must be derived from the audio itself, because there is no
   step count to divide by.

## Request flow

```
┌──────────────────────────────────────────────────────────────────┐
│ Browser — three static files, no build step                       │
│                                                                   │
│  index.html   drop zone, model/language pickers, transcript view   │
│  app.js       upload, EventSource subscription, rendering          │
│  styles.css   token-based palette, light and dark                  │
└───────┬───────────────────────────────────────────▲───────────────┘
        │ 1. POST /api/transcribe                   │ 3. SSE frames
        │    (multipart)                            │
        ▼                                           │
┌──────────────────────────────────────────────────────────────────┐
│ FastAPI — server.py                                              │
│                                                                   │
│  · validates extension, model, language, task                     │
│  · streams the upload to disk in 1 MB chunks, enforcing a limit    │
│  · creates a Job, returns 202 with a job id immediately            │
│  · /stream polls job state at 4 Hz and emits only what changed     │
│  · /download/{fmt} renders on demand from stored segments          │
└───────┬───────────────────────────────────────────▲───────────────┘
        │ 2. spawn worker thread                    │ reads under lock
        ▼                                           │
┌──────────────────────────────────────────────────────────────────┐
│ Worker thread — transcriber.py                                   │
│                                                                   │
│  Job.run():                                                       │
│    load_model()          cached per (size, compute_type)           │
│    model.transcribe()    returns a generator, not a list           │
│    for segment in …:     append under lock, update progress        │
│                                                                   │
│  JobStore: id → Job, evicts oldest finished jobs and their uploads │
└──────────────────────────────────────────────────────────────────┘
```

## Decisions

### Threads, not async, for the transcription itself

`faster-whisper` is synchronous CPU-bound C++ behind a Python wrapper. Wrapping
it in `async def` would block the event loop and freeze every other request, so
each job gets a real thread. The GIL is not a bottleneck here because CTranslate2
releases it during inference. FastAPI's async nature is still doing useful work —
it is what makes hundreds of concurrent SSE connections cheap.

### Server-sent events, not WebSockets or polling

The data flows one way: server to client, until the job ends. SSE is the exact
shape of that problem — it is a plain HTTP response, it reconnects on its own,
and it needs no protocol upgrade. A WebSocket would add a bidirectional channel
nothing uses.

The `/stream` endpoint polls the job's own state at 4 Hz rather than having the
worker push into a queue. That choice is deliberate: state is the single source
of truth, so a client that reconnects mid-job simply asks for segments from index
*n* and catches up. There is no replay buffer to keep consistent.

### Progress from audio position

`model.transcribe()` returns a generator, and each segment carries its `end`
timestamp. Dividing that by the total duration Whisper reports gives honest
progress — it tracks how much *audio* is done, not how many function calls have
returned. It is capped at 99.9% until the generator is actually exhausted, so the
bar never sits at 100% while work continues.

### Segments stream, files render on demand

Only segments are stored. Every export format is rendered from them when
requested, so adding a format is a pure function and nothing needs regenerating.
The five formats — plain text, timestamped text, SRT, VTT and JSON — are about
sixty lines of code in total.

### The model cache is a dictionary with a lock

Loading `small` takes a couple of seconds; doing it per job would dominate short
files. Models are cached by `(size, compute_type)` and shared across jobs, which
is safe because CTranslate2 models are thread-safe for inference. The double
-checked pattern around the lock keeps the fast path lock-free after warm-up.

### Isolation as a hard constraint

The app must never disturb anything else on the machine, so:

| Concern | Approach |
|---|---|
| Python | Standalone CPython 3.12 in a project-local `.venv` via uv |
| Model weights | Cached in `./models`, not shared `~/.cache/huggingface` |
| Port | 8770, well clear of 3000/5173/8000; startup aborts if taken |
| System packages | None — PyAV bundles ffmpeg, so nothing to install |
| Network exposure | Binds `127.0.0.1` only |

Choosing faster-whisper over reference Whisper matters here too: no PyTorch means
the entire environment is 194 MB rather than several gigabytes.

### Bounded memory

`JobStore` keeps the 40 most recent jobs and deletes the uploaded audio of
finished ones as they are evicted. A long-running server therefore has a ceiling
on both memory and disk, without needing a cleanup daemon.

## Failure handling

| Failure | Behaviour |
|---|---|
| Unsupported file type | 400 before anything is written to disk |
| File over the size limit | 413 mid-stream; the partial file is deleted |
| Empty upload | 400, partial file deleted |
| Decode or inference error | Job moves to `error`, message surfaced in the UI |
| No speech found | Job completes with zero segments; the UI says so plainly |
| Client disconnects | Job keeps running; reconnecting resumes from the last segment |
| Job evicted before download | 404 with a clear message rather than an empty file |

Nothing is swallowed — every failure either becomes an HTTP status with a
readable message or lands on the job as an error the UI displays.

## What I would add next

- **Speaker labels.** Diarisation would turn a two-person conversation from one
  block of prose into an attributed transcript. It is the single biggest
  usability gain still on the table.
- **Batch mode.** A queue for a folder of recordings rather than one file at a
  time.
- **In-page playback.** Clicking a timestamp to hear that moment, with the
  transcript following along.
- **Word-level timings.** `faster-whisper` can emit them; they would make search
  and subtitle alignment sharper.
