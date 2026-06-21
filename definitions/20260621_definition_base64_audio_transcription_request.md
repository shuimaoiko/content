---
title: 'Base64 audio transcription request'
description:
  'A base64 audio transcription request sends encoded audio bytes inside a JSON
  body instead of uploading the file as multipart form data.'
date: 2026-06-21
author: 'Shuimao'
---

# Base64 audio transcription request

## Definition

A base64 audio transcription request is an API request that reads an audio file
as bytes, encodes those bytes as base64 text, and sends the result in a JSON
body with metadata such as the audio format, model name, language, or provider
options. The server decodes the base64 string, transcribes the audio, and
returns text plus optional usage metadata.

This differs from multipart upload APIs, where the client streams a file field
directly in a form request. Base64 JSON requests are easier to pass through
SDKs, queues, and serverless workers, but they increase payload size because
base64 adds encoding overhead.

## Context and Usage

Speech-to-text providers use base64 JSON requests when they want one request
shape across SDKs and model routers. In an AI workflow, this is useful when a
tool needs to route the same audio clip to different transcription models while
keeping authentication, payload construction, and response parsing in one
provider adapter.

For production use, the client should record the exact audio format in the
payload, avoid data URI prefixes unless the API asks for them, and keep large
recordings split into smaller sections so the encoded body does not trigger
timeouts or request-size limits.
