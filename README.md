# Podscript

*Podcast → Transcript!*

Transcribe any podcast episode or YouTube video from the command line. Generates clean markdown with speaker labels and timestamps. Choose a hosted provider such as [AssemblyAI](https://www.assemblyai.com/), [OpenAI](https://platform.openai.com/), or [ElevenLabs](https://elevenlabs.io/), or run **fully locally** with [Whisper](https://github.com/SYSTRAN/faster-whisper).

<div align="center">
    <img src="./assets/image.png" alt="Podscript - Podcast to Transcript" width="600" />
</div>


## Installation

```bash
# Local transcription (free, no API key needed)
pip install podscript[local]

# Or use ElevenLabs API
pip install podscript
podscript --setup  # paste your ElevenLabs API key
```

For local mode, just add `--local` to any command. For ElevenLabs, you'll need an [API key](https://elevenlabs.io/app/settings/api-keys).

For YouTube support, also install [yt-dlp](https://github.com/yt-dlp/yt-dlp) and both `ffmpeg` and `ffprobe` from [FFmpeg](https://ffmpeg.org/). `imageio-ffmpeg` alone is not sufficient because it does not provide `ffprobe`.

### Installing from this repository

```bash
cd /path/to/podscript
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e ".[local]"
python -m pip install yt-dlp
cp .env.example .env
```

On Ubuntu or Debian, install FFmpeg system-wide:

```bash
sudo apt install ffmpeg
```

After activation, verify that the command resolves to this environment:

```bash
which podscript
podscript --help
```

If `podscript` is not found, run `source .venv/bin/activate` from the repository root.

`.env` is loaded automatically from the repository directory. Fill in only the API key for the provider you plan to use. Keep `.env` private; it is ignored by Git.

## Usage

```bash
# Transcribe a podcast from an Apple Podcasts link
podscript "https://podcasts.apple.com/us/podcast/huberman-lab/id1545953110?i=1000690"

# Transcribe a YouTube video
podscript "https://www.youtube.com/watch?v=dQw4w9WgXcQ"

# Use AssemblyAI for long videos and hosted speaker diarization
podscript "https://www.youtube.com/watch?v=..." --provider assemblyai

# Use OpenAI hosted transcription without local GPU memory
podscript "https://www.youtube.com/watch?v=..." --provider openai

# Use an RSS feed directly
podscript https://feeds.simplecast.com/JGE3yC0V

# Browse episodes first
podscript https://feeds.simplecast.com/JGE3yC0V --list

# Search for a specific episode
podscript https://feeds.simplecast.com/JGE3yC0V --search "AI"

# Pick episode #3 from the list
podscript https://feeds.simplecast.com/JGE3yC0V --episode 3

# Custom output filename
podscript https://feeds.simplecast.com/JGE3yC0V --latest --output transcript.md
```

Without any flags, the default behavior is to transcribe the most recent episode.

## Transcription providers

Provider API keys are read from environment variables. The application does not require provider SDKs; it uses their HTTPS APIs through the existing `requests` dependency.

| Provider | Flag | Environment variable | Speaker diarization | Long-video behavior |
|---|---|---|---|---|
| AssemblyAI | `--provider assemblyai` | `ASSEMBLYAI_API_KEY` | Yes, hosted | Uploads the full audio and polls for completion |
| OpenAI | `--provider openai` | `OPENAI_API_KEY` | No with the standard Whisper endpoint | Splits audio into ten-minute chunks |
| ElevenLabs | `--provider elevenlabs` | `ELEVENLABS_API_KEY` | Yes | Hosted transcription |
| Local Whisper | `--provider local` or `--local` | Optional `HF_TOKEN` for pyannote | Optional, local pyannote | Uses local GPU/CPU memory |

Set the key for the provider you choose:

```bash
export ASSEMBLYAI_API_KEY="..."
# or:
export OPENAI_API_KEY="..."
# or:
export ELEVENLABS_API_KEY="..."
```

For a long YouTube video on a small GPU, prefer AssemblyAI:

```bash
source .venv/bin/activate
export ASSEMBLYAI_API_KEY="..."
podscript "https://www.youtube.com/watch?v=..." \
  --provider assemblyai \
  --cookies-from-browser chrome \
  --js-runtime bun \
  --output transcript.md
```

This path does not load Whisper or pyannote, so the local GPU out-of-memory failure cannot occur. AssemblyAI's speaker labels are normalized into Podscript's `Speaker 1`, `Speaker 2`, and so on.

OpenAI's hosted Whisper path avoids GPU use and automatically chunks audio before upload:

```bash
export OPENAI_API_KEY="..."
podscript "https://www.youtube.com/watch?v=..." \
  --provider openai \
  --output transcript.md
```

OpenAI's standard Whisper endpoint returns timestamps but not speaker identities, so output is labeled `Speaker 1`.

## Output

Generates a markdown file with speaker labels and timestamps:

```markdown
# The Economics of Carbon Removal

**Podcast:** a16z Podcast
**Date:** 2/10/2026
**Duration:** 1:04:23

---

## Speaker 1
[0:00] Welcome back to the show. Today we're talking about...

## Speaker 2
[0:15] Thanks for having me. So the key challenge with carbon removal is...

## Speaker 1
[2:41] That's fascinating. How does the economics actually work at scale?
```

## Local Transcription

You can transcribe with a local Whisper model — no API key required:

```bash
pip install podscript[local]
```

This installs `faster-whisper`, `pyannote.audio`, and `torch`.

### Usage

```bash
# Basic local transcription (uses "base" model, no speaker diarization)
podscript "https://www.youtube.com/watch?v=..." --provider local

# Use a larger model for better accuracy
podscript "https://www.youtube.com/watch?v=..." --provider local --model medium

# If YouTube asks you to sign in or returns HTTP 429, use browser cookies
# and a JavaScript runtime. Close the browser first.
podscript "https://www.youtube.com/watch?v=..." --local \
  --cookies-from-browser chrome --js-runtime bun

# Enable speaker diarization with a HuggingFace token
podscript "https://feeds.example.com/rss" --local --hf-token hf_xxxxx

# Or set the token as an environment variable once
export HF_TOKEN=hf_xxxxx
podscript "https://feeds.example.com/rss" --local
```

### Model Sizes

| Model | Speed | Quality | VRAM |
|-------|-------|---------|------|
| `tiny` | Fastest | Lower | ~1 GB |
| `base` | Fast | Good (default) | ~1 GB |
| `small` | Moderate | Better | ~2 GB |
| `medium` | Slower | Great | ~5 GB |
| `large-v2` | Slowest | Best | ~10 GB |
| `large-v3` | Slowest | Best | ~10 GB |

CPU mode uses `int8` quantization automatically. GPU (CUDA) uses `float16`.

### Speaker Diarization

Speaker diarization (identifying who said what) requires a free [HuggingFace](https://huggingface.co/) token:

1. Create an account at [huggingface.co](https://huggingface.co/join)
2. Accept the terms for the diarization models:
   - [pyannote/speaker-diarization-3.1](https://huggingface.co/pyannote/speaker-diarization-3.1)
   - [pyannote/segmentation-3.0](https://huggingface.co/pyannote/segmentation-3.0)
   - [pyannote/speaker-diarization-community-1](https://huggingface.co/pyannote/speaker-diarization-community-1)
3. Create a token at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens)
4. Pass it via `--hf-token hf_xxxxx` or set `HF_TOKEN` in your environment

Without a token, all speech is attributed to "Speaker 1" — still useful for single-speaker content.

## YouTube troubleshooting

Some YouTube videos are more aggressively protected than others. A `429`, `Sign in to confirm you're not a bot`, or `Only images are available` error comes from YouTube/yt-dlp before Whisper runs.

Use a browser session that is signed in to YouTube:

```bash
source .venv/bin/activate
podscript "https://www.youtube.com/watch?v=..." \
  --local \
  --cookies-from-browser chrome \
  --js-runtime bun \
  --output transcript.md
```

Replace `chrome` with `chromium`, `firefox`, `brave`, or `edge` as appropriate. Close the browser before running the command so yt-dlp can read its cookie database. Never paste cookie contents or access tokens into an issue or commit.

Recent yt-dlp versions also use an external JavaScript challenge solver. Podscript automatically enables the official `ejs:github` remote component for YouTube downloads. If you invoke yt-dlp directly, add `--remote-components ejs:github`.

The Hugging Face token only controls speaker diarization. It does not authenticate YouTube, and valid Hugging Face tokens use the `hf_...` prefix.

If GPU transcription fails with a missing `libcublas.so.12`, ctranslate2/faster-whisper needs the CUDA 12 runtime libraries even when PyTorch installed CUDA 13 libraries. Install `nvidia-cublas-cu12` and `nvidia-cuda-runtime-cu12` in the virtual environment. Podscript configures the virtual-environment CUDA library path before loading Whisper.

If diarization reports that a requested 10-second MP3 chunk contains a few samples too many or too few, this is an audio-decoder boundary issue. Podscript converts the audio to temporary mono 16 kHz PCM WAV before pyannote runs; make sure `ffmpeg` is available on `PATH`.

On smaller GPUs, Podscript releases the Whisper model and clears the CUDA cache before running pyannote. If diarization still reports CUDA out-of-memory, rerun with `--model tiny` or omit `--hf-token` to skip diarization.

## License

MIT
