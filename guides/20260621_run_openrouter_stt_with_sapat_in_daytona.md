---
title: 'Run OpenRouter STT with Sapat in Daytona'
description:
  'Use OpenRouter as a speech-to-text model router from a reproducible Sapat
  workspace in Daytona.'
date: 2026-06-21
author: 'Shuimao'
tags: ['daytona', 'speech-to-text', 'python', 'openrouter']
---

# Run OpenRouter STT with Sapat in Daytona

# Introduction

Most speech-to-text integrations are written against one provider at a time.
That works until the team wants to compare cost, latency, language coverage,
or output quality across multiple models. Switching from one API to another
often means changing authentication, request bodies, file upload formats, and
response parsing before the transcript can even be reviewed.

This guide shows how to run OpenRouter's dedicated speech-to-text endpoint
through Sapat inside a Daytona workspace. Sapat gives the workflow a consistent
CLI. Daytona gives the workflow a clean environment. OpenRouter acts as the
model routing layer, letting the same Sapat provider reach models such as
Whisper, GPT-4o Transcribe, Voxtral, Qwen ASR, Parakeet, and MAI Transcribe.

This is a current implementation path, not the older rejected OpenRouter
attempt. A previous Sapat PR was closed because OpenRouter did not expose the
claimed transcription endpoint at that time. OpenRouter now documents a
dedicated `POST /api/v1/audio/transcriptions` endpoint that accepts a
[base64 audio transcription request](/definitions/20260621_definition_base64_audio_transcription_request.md)
and returns text plus usage metadata. The companion PR implements that current
JSON/base64 API shape in Sapat's provider plugin architecture.

## TL;DR

- Use Daytona to keep Python, `ffmpeg`, and Sapat reproducible.
- Store `OPENROUTER_API_KEY` in the workspace environment, not in Git.
- Run Sapat with `--provider openrouter` and a model alias such as `whisper`,
  `gpt-4o-mini`, `voxtral-mini`, `qwen-flash`, or `parakeet`.
- Use `--language auto` when you want OpenRouter to auto-detect language.
- Keep a small validation set so model comparisons are based on the same audio
  and the same review criteria.

## How the workflow fits together

![OpenRouter STT workflow with Sapat in Daytona](/assets/20260621_run_openrouter_stt_with_sapat_in_daytona_workflow.svg)

The workflow has six parts:

- Daytona creates the workspace.
- Sapat converts source media into an audio file.
- The OpenRouter provider reads the audio bytes and base64-encodes them.
- OpenRouter receives a JSON request with `input_audio`, `model`, language, and
  temperature.
- OpenRouter routes the request to the selected STT model.
- Sapat writes the returned transcript next to the input file.

This keeps the command line stable while model choice stays flexible. The same
workspace can test a fast inexpensive model today and a higher-accuracy model
tomorrow without rewriting the transcription pipeline.

## Prerequisites

You need:

- A Daytona workspace or another clean Linux development environment.
- Python 3.8 or newer.
- `ffmpeg` for media conversion.
- An OpenRouter API key with access to STT models.
- A short sample recording you are allowed to process.

If you are testing before the companion Sapat PR is merged, install from the
provider branch:

```bash
git clone https://github.com/shuimaoiko/sapat.git
cd sapat
git checkout codex/openrouter-stt-provider
python3 -m venv .venv
source .venv/bin/activate
pip install -e '.[dev]'
```

The companion provider implementation is
[nibzard/sapat#69](https://github.com/nibzard/sapat/pull/69). After it is
merged, use the normal Sapat install path.

## Step 1: Configure the workspace

Check Python and `ffmpeg`:

```bash
python3 --version
ffmpeg -version
```

Install `ffmpeg` if the image does not already include it:

```bash
sudo apt-get update
sudo apt-get install -y ffmpeg
```

Create a simple folder layout:

```bash
mkdir -p samples reviewed scorecards
```

Copy one short file into `samples/`. Start with one file, not a whole folder.
Base64 JSON requests are larger than the raw audio bytes, so short validation
clips are easier to debug and cheaper to compare.

## Step 2: Add OpenRouter credentials

Export your key in the workspace shell:

```bash
export OPENROUTER_API_KEY="your-openrouter-api-key"
```

If you use a local `.env` file, keep it ignored:

```bash
printf 'OPENROUTER_API_KEY=your-openrouter-api-key\n' > .env
```

Do not commit the key, generated transcripts from private recordings, sample
media, or billing screenshots. The provider implementation is covered by mock
tests, so reviewers can verify request construction without any real key.

## Step 3: Pick an STT model alias

The provider includes aliases for common OpenRouter STT models:

| Sapat model | OpenRouter model |
| --- | --- |
| `whisper` | `openai/whisper-large-v3` |
| `whisper-turbo` | `openai/whisper-large-v3-turbo` |
| `gpt-4o` | `openai/gpt-4o-transcribe` |
| `gpt-4o-mini` | `openai/gpt-4o-mini-transcribe` |
| `voxtral-mini` | `mistralai/voxtral-mini-transcribe` |
| `qwen-flash` | `qwen/qwen3-asr-flash-2026-02-10` |
| `parakeet` | `nvidia/parakeet-tdt-0.6b-v3` |
| `mai` | `microsoft/mai-transcribe-1.5` |

Start with Whisper for a baseline:

```bash
sapat samples/demo.mp3 \
  --provider openrouter \
  --model whisper \
  --language auto \
  --quality M
```

Then run a second model against the same clip:

```bash
sapat samples/demo.mp3 \
  --provider openrouter \
  --model gpt-4o-mini \
  --language auto \
  --quality M
```

Using `--language auto` matters. Sapat's CLI default is `en`, which is useful
for English-only recordings but not for multilingual comparison. When the
provider receives `auto`, it omits the `language` field so OpenRouter can use
auto-detection.

## Step 4: Compare transcripts with a scorecard

Model routing is only valuable if the comparison is consistent. Keep the same
source clip and review criteria for each run.

Create a scorecard:

```markdown
## OpenRouter STT scorecard

- Source file:
- Sapat provider: openrouter
- Model alias:
- Resolved model slug:
- Audio quality flag:
- Language setting:
- Terms missed:
- Names or numbers corrected:
- Latency notes:
- Cost notes from usage metadata:
- Keep, retry, or reject:
```

After each run, copy the transcript into the `reviewed/` folder with the model
in the filename:

```bash
cp samples/demo.txt reviewed/demo.openrouter.whisper.txt
```

Then rerun with another model and copy the next output:

```bash
sapat samples/demo.mp3 --provider openrouter --model qwen-flash --language auto
cp samples/demo.txt reviewed/demo.openrouter.qwen-flash.txt
```

The OpenRouter response can include usage metadata such as seconds, token
counts, and cost. The companion Sapat provider stores the raw response on the
`TranscriptionResult` for tests and future callers. The CLI still writes the
plain transcript text, which keeps the main workflow simple.

## Step 5: Handle large recordings carefully

OpenRouter's STT documentation recommends splitting very long recordings,
because upstream provider timeouts can happen with large files. Sapat already
has chunking behavior based on the provider's file size limit. The OpenRouter
provider keeps the default maximum at 25 MB to avoid creating huge base64 JSON
bodies.

For long meetings:

- Split the meeting by agenda section when possible.
- Keep the original source file outside Git.
- Name the chunks predictably, such as `meeting-01-intro.mp3`.
- Review chunk boundaries manually, because words around cuts are easy to lose.
- Record which model produced each transcript.

For browser recordings or voice notes, MP3 is a practical default because it
keeps the encoded request smaller than WAV. For short, high-value clips where
quality matters more than payload size, test WAV and compare the result.

## Step 6: Validate the provider without a real key

The companion Sapat PR includes mock tests that verify:

- `OPENROUTER_API_KEY` controls provider availability.
- The request URL is OpenRouter's STT endpoint.
- Audio bytes are base64-encoded without a data URI prefix.
- The payload includes `input_audio.data`, `input_audio.format`, model, and
  temperature.
- `--language auto` omits the language field for model auto-detection.
- Model aliases resolve to current OpenRouter STT model slugs.
- API errors raise a clear runtime error.

Run targeted tests:

```bash
python -m pytest tests/providers/test_openrouter.py tests/test_registry.py -q
```

Run the full validation set before opening a PR:

```bash
python -m pytest -q
python -m black --check sapat/providers/openrouter.py tests/providers/test_openrouter.py
python -m compileall sapat tests/providers/test_openrouter.py
git diff --check
```

These checks prove the request shape without uploading any private audio or
using a real API key during review.

## Common issues and troubleshooting

**Problem:** Sapat says `openrouter` is not available.

**Solution:** Check that `OPENROUTER_API_KEY` is exported in the same shell.
Provider discovery skips providers with missing required environment variables.

**Problem:** OpenRouter says the model was not found.

**Solution:** Use one of the aliases above or check OpenRouter's models API
with `output_modalities=transcription` for current model slugs.

**Problem:** A multilingual clip is treated like English.

**Solution:** Pass `--language auto` so the provider omits the language field.
If you know the language, pass the ISO-639-1 code instead, such as `ja` or
`en`.

**Problem:** Requests time out on long audio.

**Solution:** Split the recording into shorter sections. Base64 JSON bodies are
larger than raw files, and upstream providers can time out before processing a
large clip.

## Conclusion

You now have a repeatable OpenRouter STT workflow in Daytona: install the Sapat
branch, export `OPENROUTER_API_KEY`, choose a model alias, run `sapat
--provider openrouter`, and review the transcript output. OpenRouter gives the
workflow a routing layer across several STT models, while Sapat keeps the local
command stable and testable.

This pattern is especially useful for model comparison. Keep the same sample
clips, review with the same scorecard, and decide based on transcript quality,
cost, latency, and language behavior rather than API convenience alone.

## References

- [OpenRouter Speech-to-Text guide](https://openrouter.ai/docs/guides/overview/multimodal/stt)
- [OpenRouter create transcription API reference](https://openrouter.ai/docs/api/api-reference/transcriptions/create-audio-transcriptions)
- [OpenRouter speech-to-text model collection](https://openrouter.ai/collections/speech-to-text-models)
- [Sapat OpenRouter provider pull request](https://github.com/nibzard/sapat/pull/69)
