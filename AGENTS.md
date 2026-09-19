# Podscript Agent Guide

## Repository shape

- The CLI entry point is the flat `podscript.py` module.
- `pyproject.toml` must keep `py-modules = ["podscript"]` so the editable console script can import that module.
- Work from the repository root: `/home/catsandcode/dev/projects/podscript`.
- `.venv/` and generated transcript `.md` files are local artifacts. Preserve them unless the user explicitly asks for cleanup.

## Environment

Use the existing virtual environment when available:

```bash
source .venv/bin/activate
podscript --help
```

The local transcription stack requires `faster-whisper`, `pyannote.audio`, and `torch`. YouTube support additionally requires `yt-dlp`, `ffmpeg`, and `ffprobe`. `imageio-ffmpeg` provides only `ffmpeg`; it does not satisfy yt-dlp's `ffprobe` requirement.

Hosted providers use the existing `requests` dependency and do not require provider SDKs. `ASSEMBLYAI_API_KEY`, `OPENAI_API_KEY`, and `ELEVENLABS_API_KEY` must never be printed or committed.

The `.env` file may set `PODSCRIPT_PROVIDER`, `PODSCRIPT_MODEL`, `PODSCRIPT_COOKIES_FROM_BROWSER`, `PODSCRIPT_JS_RUNTIME`, and `PODSCRIPT_OUTPUT`. Empty values preserve defaults. CLI flags override these settings. Model names are provider-specific.

On Linux, ctranslate2/faster-whisper may require CUDA 12 libraries even when the installed PyTorch wheel uses CUDA 13. In the validated working environment, `nvidia-cublas-cu12` and `nvidia-cuda-runtime-cu12` were installed manually, and `_configure_cuda_library_path()` exposes virtual-environment CUDA libraries before native Whisper libraries load. Fresh installs may need those packages when GPU loading reports a missing `libcublas.so.12`. Do not remove that setup without testing both CPU and GPU paths.

## YouTube lessons

The YouTube path first asks yt-dlp for metadata, then downloads and converts audio to MP3 before transcription. Failures during metadata lookup are YouTube/yt-dlp failures, not Whisper failures. Before pyannote diarization, convert the downloaded audio to temporary mono 16 kHz PCM WAV; passing MP3 directly can produce one-sample boundary mismatches on 10-second chunks. Release the Whisper model and clear the CUDA cache before pyannote runs; both models cannot reliably coexist on smaller GPUs.

For long videos or GPUs with limited VRAM, prefer `--provider assemblyai`. AssemblyAI handles hosted transcription and speaker diarization without loading local Whisper or pyannote. The OpenAI provider avoids local GPU use and chunks audio into ten-minute uploads, but standard OpenAI Whisper output has no speaker labels. Hosted provider calls should remain lazy so local-only users do not need provider SDKs.

Some videos trigger HTTP 429 or JavaScript challenges while others work without authentication. For protected videos, use a signed-in browser session:

```bash
podscript "https://www.youtube.com/watch?v=..." \
  --local \
  --cookies-from-browser chrome \
  --js-runtime bun \
  --output transcript.md
```

- Close the browser before yt-dlp reads its cookie database.
- Supported browser names include `chrome`, `chromium`, `firefox`, `brave`, and `edge`.
- Podscript passes `--remote-components ejs:github` automatically for YouTube so yt-dlp can fetch its official EJS challenge solver.
- The Hugging Face token is only for pyannote speaker diarization. It does not solve YouTube authentication.
- Never print, commit, or place browser cookies, Hugging Face tokens, or API keys in source, logs, docs, or test fixtures.

## Validation

For source or packaging changes, run the narrow checks first:

```bash
python -m py_compile podscript.py
podscript --help
git diff --check
```

When network access is available, a YouTube smoke test should use a public video and write output outside the repository:

```bash
podscript "https://www.youtube.com/watch?v=..." \
  --local --model tiny --output /tmp/podscript-smoke-test.md
```

Confirm the output is non-empty and contains a title, duration, timestamps, and transcript text. Do not use a user's private cookies or tokens for a smoke test.

## Documentation and generated output

- Keep README commands aligned with the actual CLI flags.
- Explain YouTube cookie and JavaScript-runtime recovery separately from Hugging Face diarization.
- Do not commit generated transcripts unless the user explicitly requests them.
- Report network-dependent validation separately from local syntax and CLI checks.
