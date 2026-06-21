---
title: 'Run SiliconFlow ASR with Sapat in Daytona'
description:
  'Build a reproducible Sapat workflow for SiliconFlow SenseVoice transcription
  inside a Daytona workspace.'
date: 2026-06-21
author: 'Shuimao'
tags: ['daytona', 'speech-to-text', 'python', 'siliconflow']
---

# Run SiliconFlow ASR with Sapat in Daytona

# Introduction

Speech-to-text experiments often start as one-off scripts. Someone exports a
meeting clip, tries a provider, saves a transcript, and only later discovers
that the exact setup is hard to repeat. The API key lived in a shell history,
the audio conversion settings were not recorded, or the provider accepted a
slightly different multipart form than the code assumed.

This guide shows how to run SiliconFlow automatic speech recognition through
Sapat inside a Daytona workspace. Sapat provides a small Python CLI for routing
media through speech-to-text providers. Daytona gives the workflow a clean,
reproducible development environment. The companion Sapat provider adds a
`siliconflow` route that sends audio to SiliconFlow's
[OpenAI-compatible transcription endpoint](/definitions/20260621_definition_openai_compatible_transcription_endpoint.md)
while keeping the request body aligned with SiliconFlow's documented `file` and
`model` fields.

The workflow is useful for AI engineers who want a practical transcription
path for Mandarin, Cantonese, English, Japanese, Korean, or mixed multilingual
recordings. SiliconFlow exposes models such as `FunAudioLLM/SenseVoiceSmall`
and `TeleAI/TeleSpeechASR` through a simple HTTP API. SenseVoice is especially
interesting when a team needs Chinese-language or multilingual recognition but
still wants a lightweight CLI workflow that can be tested without shipping real
audio or secrets in the repository.

## TL;DR

- Use a Daytona workspace so Python, `ffmpeg`, and Sapat are installed the same
  way for every run.
- Store `SILICONFLOW_API_KEY` in the workspace environment or a local ignored
  `.env` file.
- Use `sapat --provider siliconflow --model sensevoice` for the default
  SiliconFlow SenseVoice path.
- Keep short sample clips for validation before running customer recordings or
  long meetings.
- The companion Sapat PR includes mock tests, so the request shape can be
  validated without a real SiliconFlow key.

## How the workflow fits together

![SiliconFlow ASR workflow with Sapat in Daytona](/assets/20260621_run_siliconflow_asr_with_sapat_in_daytona_workflow.svg)

The flow has five parts:

- Daytona creates a reproducible workspace with Python and system tools.
- Sapat converts media into the provider's preferred audio format.
- The SiliconFlow provider sends a multipart request with the audio file and
  model name.
- SiliconFlow returns transcript text.
- Sapat writes a `.txt` file next to the source media for review, handoff, or
  downstream processing.

This separation keeps provider details out of the operator's daily command.
The person running transcription should not need to remember the endpoint URL,
which model names are supported, or which request fields are safe to send. That
belongs in provider code and tests.

## Prerequisites

You need:

- A Daytona workspace or another clean Linux development environment.
- Python 3.8 or newer.
- `ffmpeg`, because Sapat can convert source videos before transcription.
- A SiliconFlow API key.
- A short audio or video sample that you are allowed to process.

If you are testing before the companion Sapat PR is merged, install from the
fork branch:

```bash
git clone https://github.com/shuimaoiko/sapat.git
cd sapat
git checkout codex/siliconflow-sapat-provider
python3 -m venv .venv
source .venv/bin/activate
pip install -e '.[dev]'
```

The companion provider implementation is
[nibzard/sapat#68](https://github.com/nibzard/sapat/pull/68). After that PR is
merged, replace the fork checkout with the normal Sapat install path.

## Step 1: Create a workspace for the transcription run

Start with a dedicated workspace rather than your main project folder. The goal
is to keep setup, inputs, outputs, and validation files easy to inspect.

Inside the workspace, check the tools:

```bash
python3 --version
ffmpeg -version
```

If `ffmpeg` is missing in a Debian or Ubuntu image, install it:

```bash
sudo apt-get update
sudo apt-get install -y ffmpeg
```

Create folders for input samples and reviewed transcripts:

```bash
mkdir -p samples reviewed
```

Use copies of test recordings in `samples/`. Do not start with private
customer media. A good validation set has a clean clip, a noisy clip, and one
clip with real product names or domain terms.

## Step 2: Configure SiliconFlow credentials

Set the API key as an environment variable:

```bash
export SILICONFLOW_API_KEY="your-siliconflow-api-key"
```

If you prefer a `.env` file, keep it local and ignored by Git:

```bash
printf 'SILICONFLOW_API_KEY=your-siliconflow-api-key\n' > .env
```

Never commit `.env`, transcripts from private recordings, or generated audio
artifacts. The provider tests use mocks, so code review does not require real
credentials.

SiliconFlow's transcription documentation lists a bearer token in the
`Authorization` header. The provider maps that to `SILICONFLOW_API_KEY`, then
sends:

- endpoint: `https://api.siliconflow.cn/v1/audio/transcriptions`
- auth: `Authorization: Bearer $SILICONFLOW_API_KEY`
- form field: `file`
- form field: `model`

The implementation intentionally avoids forwarding generic CLI fields such as
`language`, `prompt`, or `temperature` because they are not part of the
documented SiliconFlow transcription body. That makes the request easier to
reason about and keeps failures focused on credentials, file limits, or model
selection.

## Step 3: Choose the model alias

The provider exposes short aliases for the documented models:

| Sapat model | SiliconFlow model |
| --- | --- |
| `sensevoice` | `FunAudioLLM/SenseVoiceSmall` |
| `sensevoice-small` | `FunAudioLLM/SenseVoiceSmall` |
| `teleai` | `TeleAI/TeleSpeechASR` |
| `telespeech` | `TeleAI/TeleSpeechASR` |

Start with SenseVoice:

```bash
sapat samples/demo.wav \
  --provider siliconflow \
  --model sensevoice \
  --quality M
```

Use `TeleAI/TeleSpeechASR` when you specifically want to compare the second
SiliconFlow transcription option:

```bash
sapat samples/demo.wav \
  --provider siliconflow \
  --model teleai \
  --quality M
```

The input file can be audio or video. Sapat converts it to MP3 for this
provider, sends the converted audio, writes `samples/demo.txt`, then removes
only the temporary converted file. The companion PR also fixes a processing
edge case so a source file that is already in the preferred audio format is not
deleted as if it were temporary output.

## Step 4: Keep the first run small

SiliconFlow's documentation currently describes a maximum transcription upload
of 50 MB and a maximum duration of one hour. Those limits are generous enough
for many demos and voice notes, but the first run should still be short. A
30-second sample is easier to debug than a 45-minute recording.

Run one file:

```bash
sapat samples/design-review.wav \
  --provider siliconflow \
  --model sensevoice \
  --quality M
```

Then inspect the output:

```bash
ls -la samples
sed -n '1,120p' samples/design-review.txt
```

If the transcript is empty, check these items first:

- `SILICONFLOW_API_KEY` is present in the same shell that runs Sapat.
- The selected model resolves to a SiliconFlow model name.
- The source media can be decoded by `ffmpeg`.
- The converted file stays under the provider's file size and duration limits.
- The account has access to the selected SiliconFlow model.

## Step 5: Build a repeatable review loop

Raw transcription is only the first step. A useful workflow also records how
the transcript was produced and how it was reviewed.

For every provider comparison, keep a small scorecard:

```markdown
## Transcript review

- Source file: samples/design-review.wav
- Provider: siliconflow
- Model: sensevoice
- Audio quality flag: M
- Strong points:
- Weak points:
- Product names corrected:
- Follow-up action:
```

Review one transcript against the audio before processing a folder. Pay special
attention to:

- product names,
- mixed Chinese-English terms,
- speaker names,
- numbers and dates,
- code identifiers,
- places where background noise hides a word.

For team workflows, store reviewed transcripts separately:

```bash
cp samples/design-review.txt reviewed/design-review.siliconflow.txt
```

That gives you an audit trail without committing private audio. If the
transcript belongs in a public repository, remove private names and internal
details first.

## Step 6: Validate the provider without secrets

The companion Sapat PR includes tests for the provider and for the processing
edge case mentioned above. They verify that:

- `SILICONFLOW_API_KEY` controls provider availability.
- The provider sends `Authorization: Bearer ...`.
- The endpoint is SiliconFlow's transcription URL.
- The form body contains the resolved model name.
- Undocumented generic fields are not sent to SiliconFlow.
- API errors raise a clear runtime error.
- Existing source audio is not deleted during cleanup.

Run the targeted checks:

```bash
python -m pytest tests/providers/test_siliconflow.py tests/test_registry.py tests/test_process.py -q
```

Run the full test suite before opening a PR:

```bash
python -m pytest -q
python -m black --check sapat/providers/siliconflow.py sapat/process.py tests/providers/test_siliconflow.py tests/test_process.py
python -m compileall sapat tests/providers/test_siliconflow.py tests/test_process.py
git diff --check
```

Mock-based tests are important here. They prove the provider's request shape
without exposing a real API key or uploading private audio during review.

## Common issues and troubleshooting

**Problem:** Sapat says the provider is not available.

**Solution:** Confirm `SILICONFLOW_API_KEY` is exported in the current shell.
Provider discovery skips providers whose required environment variables are
missing.

**Problem:** The request fails with an authentication error.

**Solution:** Regenerate or re-copy the SiliconFlow key, then make sure there
are no surrounding quotes or spaces in the environment variable.

**Problem:** The transcript quality changes between files.

**Solution:** Compare audio quality first. Use the same `--quality` value,
avoid clipping, and keep a validation set with known expected terms.

**Problem:** A long meeting is slow or fails near the provider limits.

**Solution:** Split the recording into smaller sections before transcription
or keep the file below SiliconFlow's documented upload limits. Review section
boundaries manually so names and decisions are not split in confusing places.

## Conclusion

You now have a reproducible SiliconFlow transcription path inside Daytona:
create a workspace, install the Sapat branch, set `SILICONFLOW_API_KEY`, run
`sapat --provider siliconflow`, and review the generated transcript. The
provider keeps SiliconFlow-specific request details in code, while the Daytona
workspace keeps the environment easy to recreate.

This is also a good pattern for future providers. Start with the provider's
documented minimum request, add clear model aliases, write mock tests for the
HTTP request, and keep real recordings and credentials out of the repository.

## References

- [SiliconFlow Create transcription API](https://docs.siliconflow.cn/en/api-reference/audio/create-audio-transcriptions)
- [FunAudioLLM/SenseVoiceSmall model card](https://huggingface.co/FunAudioLLM/SenseVoiceSmall)
- [Sapat SiliconFlow provider pull request](https://github.com/nibzard/sapat/pull/68)
