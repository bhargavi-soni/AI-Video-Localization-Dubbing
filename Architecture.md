# Architecture.md — AI Video Localization & Dubbing Pipeline

> Working name: **DubFlow** (change freely)
> Goal: upload a video (MP4 or YouTube URL) → get back a dubbed video + subtitles in target languages, plus a QA page to compare original vs dubbed.
> Constraint: **100% free tools**, deployable on free cloud tiers.

---

## 1. Reality check: changes from the original brief

The original brief is a great idea, but a few choices will break on free hosting. These adjustments keep the same story while making the demo reliable. Judges care that it works live.

| Original idea | Problem on free tier | What we do instead |
|---|---|---|
| Coqui TTS / Bark on Render | Both need a GPU or lots of RAM. Render free = 512 MB RAM, no GPU, sleeps when idle. Bark is very slow on CPU. | **Primary TTS: `edge-tts`** (free, no API key, CPU-light, 40+ languages, natural neural voices). **Optional "premium" mode: Coqui XTTS** run locally / on Hugging Face Space only if time remains (voice cloning showcase). |
| Groq Whisper does diarization | Whisper gives timestamps but **not** speaker labels. | Start with **segment-level timestamps** + a simple speaker assignment (single speaker default; optional 2-speaker via LLM-based turn labelling or `pyannote` later). Keep diarization behind a feature flag. |
| Dynamic time warping for sync | DTW is for aligning feature sequences, overkill here. | **Per-segment time fitting**: generate TTS per segment, measure duration, then time-stretch with `ffmpeg atempo` (0.8x–1.25x) and/or ask the LLM to shorten the translation if too long. Same outcome, far simpler. |
| Render for processing | Free web services spin down, have a ~512 MB limit and short request timeouts. | **Hugging Face Spaces (Docker, free CPU 16 GB RAM)** or Render free with a background worker pattern. See section 8. |
| YouTube URL download | `yt-dlp` is frequently blocked from cloud/datacenter IPs. | Support it, but **MP4 upload is the primary path**. Show a friendly fallback message if YouTube blocks. Demo with an uploaded file. |

---

## 2. High-level system diagram

```
                    ┌──────────────────────────┐
                    │   Streamlit Frontend     │
                    │ (Upload, Jobs, QA view)  │
                    └───────────┬──────────────┘
                                │ calls Python modules / job runner
                                ▼
┌────────────────────────────────────────────────────────────┐
│                    Pipeline Orchestrator                   │
│  (runs stages in order, updates job status in Neon)        │
└──┬─────────┬──────────┬───────────┬───────────┬───────────┘
   ▼         ▼          ▼           ▼           ▼
 Ingest   Transcribe  Translate    TTS Dub    Assemble
 (yt-dlp  (Groq       (Groq        (edge-tts  (ffmpeg /
 /upload  Whisper)    LLaMA)       + fit)     MoviePy)
 + ffmpeg)                                         │
   │         │          │           │              ▼
   └─────────┴──────────┴───────────┴──────► Supabase Storage
                                              (videos, audio, SRT)
                                                   │
                  Neon Postgres ◄── job metadata, segments, status
                  Supabase Auth ◄── login
```

---

## 3. Pipeline stages (the heart of the project)

Each stage is a **pure Python function**: takes inputs, returns outputs, no hidden state. This makes each one testable on its own.

### Stage 1 — Ingestion
- **Input:** uploaded MP4 or YouTube URL.
- **Do:** save the file; use `yt-dlp` for URLs; extract audio with ffmpeg → 16 kHz mono WAV/FLAC; read duration.
- **Output:** `video.mp4`, `audio.wav`, `duration_sec`.
- **Guardrails:** max length for free tier (e.g. 10 min for demo; configurable), max file size.

### Stage 2 — Transcription (+ speakers)
- **Tool:** Groq `whisper-large-v3-turbo` (or `whisper-large-v3`) with `response_format="verbose_json"` and segment timestamps.
- **Chunking:** Groq limits upload size (about 25 MB on free tier). Split audio into ~5-minute chunks with ffmpeg, transcribe each, then offset timestamps and merge.
- **Output:** list of segments: `{id, start, end, text, speaker}`.
- **Speakers:** default `speaker = "SPEAKER_1"`. Optional step: LLM labels likely speaker changes from text cues. Mark as experimental in the UI.

### Stage 3 — Translation
- **Tool:** Groq LLaMA (e.g. `llama-3.3-70b-versatile`; fall back to `llama-3.1-8b-instant` if rate limited).
- **Prompt rules:** preserve tone, keep proper nouns/terminology, use natural spoken language, keep length close to source (lip/timing friendly), return **JSON** keyed by segment id.
- **Batching:** translate ~10–20 segments per request with 1–2 segments of context before/after so sentences flow.
- **Optional glossary:** user-supplied terms that must not be translated (e.g. brand names).
- **Output:** each segment gets `translated_text` per target language.

### Stage 4 — TTS dubbing
- **Tool:** `edge-tts` (async). Voice picked per target language + speaker gender (small lookup table in `config`).
- **Do:** synthesize each segment to its own audio file.
- **Output:** `seg_{id}.mp3` and its measured duration.

### Stage 5 — Timing alignment
For each segment with available time `T = end - start` and TTS duration `D`:
1. If `D <= T`: pad with silence (or leave natural gap).
2. If `D > T` and `D/T <= 1.25`: speed up with `ffmpeg atempo` (max 1.25x for natural sound).
3. If `D/T > 1.25`: ask the LLM to **shorten** the translation, re-synthesize once, then speed-up as needed.
4. Allow a segment to borrow time from the silent gap that follows it.

Then place every segment on a silent timeline at its `start` and mix into one dubbed audio track.

### Stage 6 — Video assembly
- Replace original audio with dubbed track (ffmpeg or MoviePy).
- Option: keep original background audio at low volume (simple version: mix original at 10–15% under the dub). Only attempt if time remains.
- Optionally burn subtitles in, or ship `.srt` alongside.
- **Output:** `dubbed_{lang}.mp4`.

### Stage 7 — Subtitles
- Build `.srt` from segment timestamps + translated text, one file per language (and the original-language one).

### Stage 8 — QA dashboard
- Streamlit page: side-by-side players (original vs dubbed), synchronized transcript table (original text | translation | audio clip), per-segment "regenerate" button with edit box, download buttons.

---

## 4. Tech stack (all free)

| Layer | Tool | Notes |
|---|---|---|
| Speech-to-text | Groq Whisper | Free tier with rate limits; chunk audio |
| Translation | Groq LLaMA | JSON output mode |
| TTS | edge-tts (main), Coqui XTTS (optional, local) | Bark dropped: too slow on CPU |
| Media | ffmpeg + MoviePy | ffmpeg for heavy lifting, MoviePy for simple composition |
| UI | Streamlit | Fast to build, good for demos |
| Auth | Supabase Auth | Email login; can skip auth in MVP |
| File storage | Supabase Storage (1 GB free) | Auto-delete old files |
| Database | Neon Postgres (free) | Jobs + segments |
| Hosting | Streamlit Community Cloud (UI) + Hugging Face Space or Render (heavy processing) | See section 8 |
| Language | Python 3.11 | |

---

## 5. Data model (Neon Postgres)

```sql
-- users come from Supabase Auth; we just store their id as text
CREATE TABLE jobs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         TEXT,
  source_type     TEXT,             -- 'upload' | 'youtube'
  source_ref      TEXT,             -- storage path or URL
  source_language TEXT,             -- detected or chosen
  target_languages TEXT[],          -- e.g. {'es','hi','fr'}
  status          TEXT,             -- queued|ingesting|transcribing|translating|dubbing|assembling|done|failed
  progress        INT DEFAULT 0,    -- 0-100
  error_message   TEXT,
  duration_sec    REAL,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE segments (
  id            SERIAL PRIMARY KEY,
  job_id        UUID REFERENCES jobs(id) ON DELETE CASCADE,
  idx           INT,               -- order in video
  start_sec     REAL,
  end_sec       REAL,
  speaker       TEXT,
  source_text   TEXT
);

CREATE TABLE translations (
  id             SERIAL PRIMARY KEY,
  segment_id     INT REFERENCES segments(id) ON DELETE CASCADE,
  language       TEXT,
  translated_text TEXT,
  audio_path     TEXT,
  audio_duration REAL,
  speed_factor   REAL,
  edited_by_user BOOLEAN DEFAULT FALSE
);

CREATE TABLE outputs (
  id         SERIAL PRIMARY KEY,
  job_id     UUID REFERENCES jobs(id) ON DELETE CASCADE,
  language   TEXT,
  video_path TEXT,
  srt_path   TEXT
);
```

---

## 6. Folder structure

```
dubflow/
├── app.py                    # Streamlit entry (upload + jobs list)
├── pages/
│   ├── 1_New_Job.py
│   ├── 2_My_Jobs.py
│   └── 3_QA_Review.py
├── pipeline/
│   ├── __init__.py
│   ├── ingest.py             # yt-dlp, ffmpeg audio extraction
│   ├── transcribe.py         # Groq Whisper + chunking
│   ├── translate.py          # Groq LLaMA batch translation
│   ├── tts.py                # edge-tts synthesis
│   ├── align.py              # timing fit + mixing
│   ├── assemble.py           # final video + SRT
│   └── orchestrator.py       # runs all stages, updates status
├── services/
│   ├── db.py                 # Neon connection + queries
│   ├── storage.py            # Supabase storage helpers
│   └── auth.py               # Supabase auth helpers
├── config.py                 # voices table, limits, model names
├── utils/
│   ├── srt.py
│   └── ffmpeg_utils.py
├── tests/
│   ├── test_srt.py
│   ├── test_align.py
│   └── sample_data/
├── requirements.txt
├── packages.txt              # apt packages for Streamlit Cloud (ffmpeg)
├── .env.example
├── Dockerfile                # for Hugging Face Space deployment
├── Architecture.md
├── Progress.md
└── README.md
```

---

## 7. Configuration & secrets

`.env` (never commit):
```
GROQ_API_KEY=
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_KEY=
NEON_DATABASE_URL=
MAX_VIDEO_MINUTES=10
```
On Streamlit Cloud, put them in **Secrets**. On Hugging Face, use **Space secrets**.

---

## 8. Deployment plan (free)

**Recommended: single Hugging Face Space (Docker, CPU basic, free)**
- 16 GB RAM, 2 vCPU. Handles ffmpeg + edge-tts + Streamlit comfortably.
- Dockerfile installs ffmpeg, Python deps, runs `streamlit run app.py --server.port 7860`.
- Pros: simplest, no split architecture, generous RAM.
- Cons: sleeps after inactivity (wakes in ~1 min); storage is ephemeral, so outputs go to Supabase Storage.

**Alternative: Streamlit Community Cloud for UI only**
- Great for the UI but limited RAM (~1 GB) and no background processing guarantee. Use only for short clips.

**Fallback:** Render free web service running Streamlit; be aware of 512 MB RAM and cold starts.

Pick one in Phase 1 and stick with it. Default = Hugging Face Space.

---

## 9. Limits & mitigations

| Risk | Mitigation |
|---|---|
| Groq rate limits | Retry with exponential backoff; smaller model fallback; cache transcripts |
| Long videos time out | Cap length; process in a background thread; show progress bar from DB |
| Translation too long for slot | Shorten via LLM, speed-up cap 1.25x |
| YouTube blocked | MP4 upload primary; clear error message |
| Free storage fills | Delete outputs older than 7 days; cap file size |
| Cold start in demo | Pre-warm the Space before presenting; keep a pre-processed sample job |

---

## 10. What makes this impressive for judges

1. **Live end-to-end demo** on a 1–2 minute clip into 2–3 languages.
2. **Cost comparison slide:** traditional $15K–50K/hr vs. this pipeline near $0 compute.
3. **QA dashboard** with per-segment editing and re-dub.
4. **Honest engineering:** documented trade-offs (this file).
5. Multimodal story: audio → text → translation → speech → video.

---

## 11. Stretch goals (only after MVP works)
- Voice cloning with Coqui XTTS (local demo clip).
- Background music preservation (source separation with Demucs, local only).
- Lip-sync model (Wav2Lip), shown as a "future work" slide.
- Glossary / brand terminology manager.
- Batch processing and email notification.
