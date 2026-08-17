# Audio Text Transcriptor — showcase

Turn any recording into searchable text, entirely on your own machine.

A local-first transcription app: drop in an audio or video file and get a
timestamped transcript, attributed to each speaker, that you can read, copy or
export as `.txt`, `.srt`, `.vtt` or `.json`. Speech recognition and speaker
identification both run on your own CPU — no cloud service, no API key, and no
audio ever leaves the device.

> This repository is a showcase. It documents what the app does and how it is
> put together; the implementation lives in a private repository.

![Transcribing an interview and labelling both speakers](assets/demo.gif)

---

## Why I built it

Transcribing a recording usually means uploading it to somebody else's server.
That is a poor trade for anything sensitive — a performance review, a doctor's
appointment, a call with a client. Whisper is good enough to run locally now, so
the upload is no longer a necessary cost. This app makes the local path the easy
one: one command to start, drag a file in, read the transcript.

## What it does

- **Drag and drop** any audio or video file — `.mp3`, `.m4a`, `.wav`, `.aac`,
  `.flac`, `.ogg`, `.opus` and more, plus `.mp4`, `.mov`, `.mkv` and `.webm`
  video. Video needs no conversion; the audio track is extracted automatically.
- **Speaker identification.** An interview comes back as `Speaker 1:` /
  `Speaker 2:`, numbered by who talks first, with the count either detected or
  set by hand.
- **Live transcript.** Lines appear as they are decoded rather than all at the
  end, so you can start reading a long recording immediately.
- **Five Whisper models**, from `tiny` (8× real time) to `large-v3` (best
  accuracy), chosen per job.
- **99 languages**, auto-detected, with optional translation into English.
- **Timestamped output**, toggleable between timestamped lines and plain prose.
- **Five export formats** — plain text, timestamped text, SRT and VTT subtitles,
  and JSON with per-segment timings. Speaker labels appear in all of them.
- **Silence-aware.** Voice-activity detection skips quiet stretches, which
  speeds up decoding and stops Whisper inventing text where nobody is speaking.
- **Genuinely offline.** After the one-time model download, the app makes no
  outbound network requests at all.

## Screenshots

| Drop a file | Pick model, language and speakers |
|---|---|
| ![Empty state with a drop zone](assets/01-drop-a-file.jpg) | ![An MP4 selected with speaker identification enabled](assets/02-choose-options.jpg) |

**Speaker-attributed transcript, with stats and export buttons**

![A transcript with Speaker 1 and Speaker 2 labels, statistics and download buttons](assets/03-speaker-transcript.jpg)

The stats row reports what actually happened: detected language, recording
length, word and line counts, speakers found, wall-clock time, and throughput
relative to real time.

## How it works

```
Browser (drag & drop)
      │  POST /api/transcribe          multipart upload, streamed to disk
      ▼
FastAPI                                validates, creates a job, returns 202
      │
      ▼
Worker thread                          faster-whisper: VAD → decode → segments
      │                                then diarisation labels the speakers
      │
      │  GET /api/jobs/{id}/stream     server-sent events, one per new segment
      ▼
Browser renders each line as it arrives
      │
      │  GET /api/jobs/{id}/download/{fmt}
      ▼
txt · timestamped txt · srt · vtt · json
```

Transcription is CPU-bound and blocking, so each job runs on its own worker
thread while the web layer reads job state under a lock. The browser follows
progress over server-sent events instead of polling — that is what makes lines
appear mid-transcription. Speaker identification runs after the transcript is
complete, so reading can start before the labels arrive.

There is a longer write-up of the design decisions in
[docs/architecture.md](docs/architecture.md).

## Stack

| Layer | Choice | Why |
|---|---|---|
| Speech recognition | [faster-whisper](https://github.com/SYSTRAN/faster-whisper) (CTranslate2) | 4× faster than reference Whisper on CPU and needs no PyTorch — the whole environment is ~230 MB instead of several GB |
| Speaker diarisation | [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) — pyannote segmentation + WeSpeaker embeddings | Runs on the onnxruntime already present for voice-activity detection, so ~31 MB of models buys diarisation with no new heavy dependency |
| Audio decoding | PyAV | Bundles its own ffmpeg, so there is no system dependency to install |
| Server | FastAPI + Uvicorn | Async streaming responses for SSE, with typed request validation |
| Frontend | Vanilla HTML/CSS/JS | No build step and no framework — the UI is three static files |
| Environment | uv + standalone CPython 3.12 | Self-contained, reproducible, installs nothing globally |

## Performance

Measured on an Apple M-series CPU with int8 quantisation. "2.5× real time" means
a 10-minute recording finishes in about 4 minutes.

| Model | Size | Throughput |
|---|---|---|
| `tiny` | 75 MB | ~8× real time |
| `base` | 142 MB | ~2.5× real time |
| `small` | 466 MB | ~1.1× real time |
| `medium` | 1.5 GB | ~0.4× real time |
| `large-v3` | 2.9 GB | ~0.2× real time |

Speaker identification adds roughly half the recording's length again. The demo
above is `small` with speakers on: 35 seconds of two-person interview,
transcribed and attributed in 43 seconds, word-for-word correct.

## Design notes

A few decisions worth calling out:

**Attribute words, not lines.** Whisper decides where to break a line from the
audio alone, so a line frequently spans a speaker change — the end of a question
and the start of the answer. The first version labelled whole lines by majority
overlap, which confidently put words in the wrong mouth. Attributing each word
and splitting the line where the speaker changes fixed it.

**Auto-detection was tuned against ground truth, not guessed.** The clustering
threshold was measured on known recordings across WAV, MP3 and AAC. The initial
value collapsed a two-speaker interview into one speaker at low bitrate; a lower
threshold was correct everywhere without splitting a single-speaker recording in
two. Setting the count by hand remains the more reliable option, and the UI says
so.

**A diarisation failure must not cost you the transcript.** Speaker
identification is a second pass over completed work, so its errors are reported
next to the transcript rather than replacing it.

**Streaming beats waiting.** The first version returned the whole transcript in
one response. On a 40-minute recording that is a blank screen for 20 minutes.
Publishing each segment as it is decoded turned the same wait into something you
can read as it fills in.

**Isolation was a requirement, not a nicety.** The app keeps its own Python
interpreter, its own model cache — including the Hugging Face download cache —
and its own port, and installs nothing globally. Deleting the folder removes it
completely, which also means it can never break a neighbouring project.

**int8 over float32.** Quantised inference is roughly 3–4× faster on CPU with no
accuracy difference that matters for speech. It is the reason `small` runs faster
than real time on a laptop.

## A note on the demo

The recording in the GIF and screenshots is synthetic — a scripted mock
interview rendered with two macOS text-to-speech voices. No real person's audio
appears anywhere in this repository.

## Licence

The application is MIT licensed. This showcase repository contains documentation
and media only.

Speech recognition by [faster-whisper](https://github.com/SYSTRAN/faster-whisper),
a CTranslate2 reimplementation of [OpenAI Whisper](https://github.com/openai/whisper).
Diarisation by [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) using
[pyannote](https://github.com/pyannote/pyannote-audio) segmentation and
[WeSpeaker](https://github.com/wenet-e2e/wespeaker) embedding models.
