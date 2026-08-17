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
│    _identify_speakers()  optional second pass — see below          │
│                                                                   │
│  JobStore: id → Job, evicts oldest finished jobs and their uploads │
└──────────────────────────────────────────────────────────────────┘
        │ when speaker identification is on
        ▼
┌──────────────────────────────────────────────────────────────────┐
│ Diarisation — diarizer.py                                        │
│                                                                   │
│  find_turns()      pyannote segmentation → WeSpeaker embeddings    │
│                    → clustering → speech turns with cluster ids    │
│  split_and_label() attribute each WORD, split lines where the      │
│                    speaker changes, number by who speaks first     │
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

### Diarisation without PyTorch

The obvious way to add speaker labels is pyannote.audio, which means PyTorch —
several gigabytes, plus a gated model download that needs an account. That is a
lot of weight for one feature in an app whose whole premise is running easily on
a laptop.

sherpa-onnx runs the same pyannote segmentation network, plus a WeSpeaker
embedding network, as ONNX models on onnxruntime — which faster-whisper already
depends on for voice-activity detection. Two models totalling ~31 MB, no new
heavy dependency, and no account required.

### Attributing words, not lines

This was the most interesting correctness problem in the build.

Transcription and diarisation are independent passes over the same audio, and
they disagree about where boundaries are. Whisper decides where to break a line
from the audio alone, so a line frequently runs straight across a speaker change:
the tail of one person's sentence and the beginning of the reply, in one segment.

The first implementation labelled each line by whichever speaker overlapped it
most. The result looked plausible and was wrong — a line reading
`Speaker 1: …transactions a day. That sounds significant.` attributes the first
half to the wrong person, with no visible sign that anything is off. That is the
worst kind of bug in a transcript: confidently misattributed quotes.

The fix is to work at word resolution. Whisper can emit per-word timings, so
each word is attributed to whichever speaker turn covers it, and a line is split
wherever the attribution changes. Word timings are requested only when
diarising, since they cost extra decoding work, and they are dropped from the API
payload once the labelling is done.

One refinement on top: a run of one or two words attributed to a different
speaker in the middle of a sentence is far more likely to be a diarisation wobble
than a real interjection, so runs shorter than 0.6 s that are sandwiched between
two runs of the same speaker are folded into their neighbour.

### Tuning auto-detection against ground truth

The clustering threshold decides how readily two voices are treated as one
person, and it is the single value that determines whether auto-detection works.
Rather than accept a default, it was measured: a known two-speaker interview and
a known single-speaker recording, each encoded as WAV, MP3 and AAC, scored
against the real turn-taking.

The initial threshold merged the two-speaker interview into one speaker once the
bitrate dropped, while a lower value was correct on every realistic encoding and
still did not split the single-speaker recording in two. The measurement also
found the real limit: at around 24 kbps the voice embeddings degrade enough to be
unreliable at any threshold. That is documented as a limitation rather than
papered over, and the UI recommends setting the count by hand when it is known.

### Isolation as a hard constraint

The app must never disturb anything else on the machine, so:

| Concern | Approach |
|---|---|
| Python | Standalone CPython 3.12 in a project-local `.venv` via uv |
| Model weights | Cached in `./models`, including the Hugging Face download cache, rather than shared `~/.cache/huggingface` |
| Port | 8770, well clear of 3000/5173/8000; startup aborts if taken |
| System packages | None — PyAV bundles ffmpeg and onnxruntime ships its own binaries |
| Network exposure | Binds `127.0.0.1` only |

Choosing faster-whisper over reference Whisper, and ONNX diarisation over
pyannote.audio, matters here too: with no PyTorch anywhere the entire environment
is around 230 MB rather than several gigabytes.

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
| Diarisation failure | Transcript is kept; the error is reported beside it |
| Speaker models missing | Downloaded on first use; a failure there is reported, not fatal |
| No speech found | Job completes with zero segments; the UI says so plainly |
| Client disconnects | Job keeps running; reconnecting resumes from the last segment |
| Job evicted before download | 404 with a clear message rather than an empty file |

Nothing is swallowed — every failure either becomes an HTTP status with a
readable message or lands on the job as an error the UI displays.

## What I would add next

- **Named speakers.** Letting you rename "Speaker 1" to a real name once, and
  having it apply throughout the transcript and every export.
- **Batch mode.** A queue for a folder of recordings rather than one file at a
  time.
- **In-page playback.** Clicking a timestamp to hear that moment, with the
  transcript following along.
- **Voice profiles.** Remembering a recurring speaker's embedding between
  recordings so the same person keeps the same label across files.
