# Progress.md — DubFlow Build Tracker

> Update this file at the end of every working session. Paste it back into the Claude Project so Claude knows where we are.

**Last updated:** 2026-11-18
**Current phase:** Phase 0 — Setup
**Current blocker:** none

Legend: `[ ]` todo · `[~]` in progress · `[x]` done

---

## Phase 0 — Setup & accounts (Day 1)
- [ ] Install Python 3.11 and check with `python --version`
- [ ] Install VS Code (or any editor)
- [ ] Install Git, create a GitHub account
- [ ] Install ffmpeg and check with `ffmpeg -version`
- [ ] Create a Groq account, generate API key
- [ ] Create a Supabase project (storage bucket `videos` + auth)
- [ ] Create a Neon project, copy connection string
- [ ] Create a Hugging Face account (for deployment)
- [ ] Create GitHub repo `dubflow`, clone locally
- [ ] Create virtual environment, install base packages
- [ ] Create `.env` from `.env.example` (never commit it)
- [ ] Put Architecture.md, Progress.md, Project Instructions into the Claude Project

**Exit test:** `python -c "import groq, streamlit"` runs with no error.

## Phase 1 — Pipeline core, command line only (Days 2–5)
Goal: run everything from the terminal on a 1-minute test video before building any UI.
- [ ] `ingest.py`: extract audio from MP4
- [ ] `transcribe.py`: Groq Whisper, segments with timestamps
- [ ] Add audio chunking for long files
- [ ] `translate.py`: batch translate with JSON output
- [ ] `utils/srt.py`: generate SRT (with a unit test)
- [ ] `tts.py`: edge-tts per segment
- [ ] `align.py`: time fitting + mix dubbed track
- [ ] `assemble.py`: replace audio in video
- [ ] `orchestrator.py`: run all stages in one command

**Exit test:** `python -m pipeline.orchestrator sample.mp4 --lang es` produces `dubbed_es.mp4` and `es.srt`.

## Phase 2 — Database & storage (Days 6–7)
- [ ] Create tables in Neon (see Architecture.md section 5)
- [ ] `services/db.py`: create job, update status, save segments
- [ ] `services/storage.py`: upload/download via Supabase Storage
- [ ] Orchestrator writes progress to DB

**Exit test:** run a job, see rows in Neon and files in Supabase.

## Phase 3 — Streamlit UI (Days 8–10)
- [ ] Upload page (file + language multiselect)
- [ ] YouTube URL input with error handling
- [ ] Jobs page with status/progress bar
- [ ] Download buttons (video + SRT)
- [ ] Supabase login (optional for MVP)

**Exit test:** a non-technical friend can dub a clip without help.

## Phase 4 — QA dashboard (Days 11–12)
- [ ] Side-by-side original vs dubbed players
- [ ] Segment table (source text, translation, audio)
- [ ] Edit translation + regenerate single segment
- [ ] Re-assemble video after edits

## Phase 5 — Quality & polish (Days 13–14)
- [ ] Shorten-translation loop for segments that run long
- [ ] Voice selection per language / gender
- [ ] Error handling and friendly messages
- [ ] Length/size limits enforced
- [ ] Basic tests passing
- [ ] Cleanup job for old files

## Phase 6 — Deployment (Days 15–16)
- [ ] Write `Dockerfile` and `requirements.txt`
- [ ] Deploy to Hugging Face Space
- [ ] Add secrets
- [ ] Test the live URL with a real video
- [ ] Record a backup demo video in case of live failure

## Phase 7 — Competition prep (Days 17–18)
- [ ] README with screenshots and architecture diagram
- [ ] Cost comparison slide ($30K vs ~$0)
- [ ] 3-minute demo script
- [ ] Prepare 2 sample clips in 3 languages (pre-processed)
- [ ] Rehearse; pre-warm the Space before presenting

## Stretch goals
- [ ] Voice cloning (Coqui XTTS)
- [ ] Background music preservation
- [ ] Glossary manager
- [ ] Speaker diarization improvements

---

## Decisions log
| Date | Decision | Reason |
|---|---|---|
| 2026-11-18 | edge-tts instead of Coqui/Bark as main TTS | Free tiers have no GPU |
| 2026-11-18 | Per-segment time-stretch instead of DTW | Simpler, same result |
| 2026-11-18 | Hugging Face Space as default host | 16 GB RAM free |

## Known issues
_(none yet)_

## Session notes
_Write a 2–3 line summary after each session: what you did, what broke, what's next._
