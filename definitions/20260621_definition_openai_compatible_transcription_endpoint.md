---
title: 'OpenAI-compatible transcription endpoint'
description:
  'An OpenAI-compatible transcription endpoint accepts audio transcription
  requests using a familiar multipart HTTP shape.'
date: 2026-06-21
author: 'Shuimao'
---

# OpenAI-compatible transcription endpoint

## Definition

An OpenAI-compatible transcription endpoint is an HTTP API that follows the
same broad request pattern as OpenAI's audio transcription API: a client sends
a multipart form request with an audio file, a model name, and authentication
headers, then receives structured text in response.

The phrase does not always mean every optional OpenAI field is supported. Some
providers accept only the common core fields, while others also accept language,
prompt, timestamp, or response-format options. Production code should follow
the provider's own documentation instead of assuming every OpenAI-style option
is portable.

## Context and Usage

For speech-to-text tooling, OpenAI-compatible transcription endpoints are useful
because one CLI can route similar audio requests to multiple providers. A tool
such as Sapat can keep a consistent command shape while each provider module
handles details such as API key names, endpoint URLs, supported models, file
size limits, and provider-specific response parsing.

The safest implementation pattern is to start with the documented minimum
request body, add aliases for common model names, and cover the request shape
with mock tests. That gives users a predictable workflow without committing API
keys, sample recordings, or provider-specific secrets to the repository.
