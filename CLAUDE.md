# video-use — Claude Code Configuration

## What This Repo Is

A Claude Code skill for conversation-driven video editing. Users drop raw footage in a folder, then chat with an agent to produce a final edited video. The skill reads video through transcripts and visual composites — it never watches footage directly.

**Repo role:** This is a *skill definition*, not an application. The entire repo is symlinked into `~/.claude/skills/video-use/` so helpers remain co-located with docs.

## Quick Setup Check

```powershell
# Verify prerequisites
ffmpeg -version
python --version          # needs 3.10+
python -m pip show requests librosa matplotlib pillow numpy

# Check API key
cat .env                  # must contain ELEVENLABS_API_KEY=sk-...

# Smoke test helpers
python helpers/timeline_view.py --help
python helpers/transcribe.py --help
```

## Environment

- **Python**: 3.10+ (managed with `uv`)
- **ffmpeg**: Must be on PATH — used for all video I/O
- **ElevenLabs Scribe API**: Key in `.env` as `ELEVENLABS_API_KEY`
- **Optional**: `manim` for mathematical animations (`uv pip install -e ".[animations]"`)

```powershell
# Install deps
uv pip install -e .
# or
pip install -e .
```

## Project Layout

```
helpers/
  transcribe.py         # ElevenLabs Scribe call, word-level timestamps, cached per source
  transcribe_batch.py   # 4-worker parallel transcription for multi-take projects
  pack_transcripts.py   # Raw Scribe JSON → phrase-level markdown (takes_packed.md)
  timeline_view.py      # Filmstrip + waveform PNG for [start, end] ranges
  render.py             # Full render pipeline: extract → grade → fade → concat → subtitles
  grade.py              # ffmpeg color grade: presets or auto-analysis mode

skills/
  manim-video/          # Nested skill for 3Blue1Brown-style animations
```

## Key Conventions

- **Outputs live in `<videos_dir>/edit/`** — never write artifacts into the skill repo itself
- **Transcripts are cached** — `transcribe.py` skips re-transcription if JSON already exists
- **Subtitles are applied LAST** in the ffmpeg filter chain — never before color grade or overlays
- **30ms audio fades** at every segment boundary — no exceptions
- **Never cut inside a word** — cut boundaries must align to inter-word silence
- **Session state** persists in `<videos_dir>/edit/project.md` for continuity across sessions
- **EDL format**: `edl.json` is the canonical cut decision document — render from it, not from ad-hoc flags

## Helper Quick Reference

| Helper | What it does | Key args |
|---|---|---|
| `transcribe.py` | Transcribe one source via Scribe | `--input <file> --output <json>` |
| `transcribe_batch.py` | Transcribe all takes in parallel | `--dir <videos_dir>` |
| `pack_transcripts.py` | Build `takes_packed.md` | `--dir <videos_dir>` |
| `timeline_view.py` | Filmstrip PNG for a time range | `--input <file> --start <s> --end <s>` |
| `render.py` | Render final video from EDL | `--edl <edl.json> --output <final.mp4>` |
| `grade.py` | Grade a clip | `--input <file> --preset warm_cinematic` |

Color grade presets: `warm_cinematic`, `neutral_punch`, `subtle`.

## Working in This Repo

- **SKILL.md** is the production bible — read it before changing any helper behavior
- **README.md** is the public-facing overview and setup prompt for agents
- **install.md** is the first-time user setup guide
- When modifying a helper, verify the render pipeline end-to-end with a short test clip
- Manim animations in `skills/manim-video/` have their own SKILL.md and setup script

## Do Not

- Write output files into the skill repo directory
- Remove the 30ms audio fade logic — it prevents audible pops at cuts
- Apply subtitles before grading or overlays in the ffmpeg chain
- Re-transcribe files that already have cached JSON output
- Commit `.env` (it is gitignored)
