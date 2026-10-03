---
title: "Real-time STT"
description: "Real-time speech-to-text transcription via WebSocket."
slug: "stt-realtime"
breadcrumb: "Speech"
---

# Real-time STT

Real-time speech-to-text transcription via WebSocket.

## Overview

Real-time STT is available through the **Voice Agent WebSocket** pipeline. Audio is streamed as raw PCM s16le 16 kHz mono, and a transcript is returned for each utterance as the user speaks.

There is no standalone real-time STT WebSocket endpoint — real-time transcription is part of the full Voice Agent pipeline (STT → LLM → TTS).

For file-based transcription, use the [Speech to Text](/docs/speech-to-text) REST API.

## Via Voice Agent

Connect to `WS /ws/voice-agent`, send a `config` message, then stream raw audio. Each finished user utterance comes back as a `transcript` message:

```json
{"type": "transcript", "text": "Hello, how are you?"}
```

The socket also runs the agent's reply (LLM + speech), so you receive `llm_response`, `agent_text`, `audio_start` / `audio_end` and binary MP3 audio as well. A transcript-only client can ignore those.

## Authentication

Pass your API key as a WebSocket subprotocol: `Sec-WebSocket-Protocol: token, cm_your_key`. That is a request header, so the key stays out of access logs and proxy history, which a query string does not. In the browser the constructor's second argument sets it, and the order matters: the literal `token` first, then the key.

```javascript
new WebSocket(url, ["token", "cm_your_key"]);
```

Clients that can set headers may send `Authorization: Bearer cm_your_key` instead.

The `?key=cm_your_key` query parameter is **deprecated**. It still works so existing integrations keep connecting, but prefer the subprotocol for anything new.

The key needs the `stt`, `tts` **and** `llm` permissions, because the socket runs all three.

## Audio format

Send audio as **binary** frames of raw PCM: signed 16-bit little-endian, 16 kHz, mono, with no container header. Keep frames small (tens to hundreds of milliseconds of audio); oversized frames are dropped. Compressed audio (WebM/Opus from `MediaRecorder`, MP3) is not decoded.

## Config message

The first text frame must be the config, sent within 10 seconds of connecting. Every field is optional. Send another `config` later to change settings mid-session; the server answers each one with `{"type": "ready"}`.

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `"config"` |
| `language` | string | Speech locale, default `en-IN`. One of `bn-IN`, `en-IN`, `gu-IN`, `hi-IN`, `kn-IN`, `ml-IN`, `mr-IN`, `od-IN`, `pa-IN`, `ta-IN`, `te-IN` (a bare `hi` is read as `hi-IN`); anything else falls back to `en-IN` |
| `voice` | string | A `bulbul:v3` voice from [Voices](/docs/tts-voices), default `shubh`. An unknown voice falls back to the language's default voice |
| `system_prompt` | string | Instructions for the agent's replies |
| `llm_model` | string | Chat model ID for the replies, default `gemma-4-31b`. Must be a chat model available on your plan |

Send `{"type": "clear_history"}` to reset the conversation memory.

## Close codes

| Code | Meaning |
|------|---------|
| `4001` | Missing or invalid API key |
| `4003` | Key lacks `stt`/`tts`/`llm` permission, or `llm_model` is not allowed |
| `4008` | Plan usage limit reached, or insufficient credits |
| `4009` | Config message too large |
| `1013` | Too many connections — retry later |

## Example

:::tabs
```javascript [JavaScript]
const ws = new WebSocket(
  "wss://api.callmissed.com/ws/voice-agent",
  ["token", "cm_your_key"]
);
ws.binaryType = "arraybuffer";

ws.onopen = async () => {
  ws.send(JSON.stringify({ type: "config", language: "hi-IN", voice: "shubh" }));

  // Capture the microphone at 16 kHz and send raw 16-bit PCM.
  const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
  const ctx = new AudioContext({ sampleRate: 16000 });
  const source = ctx.createMediaStreamSource(stream);
  const processor = ctx.createScriptProcessor(4096, 1, 1);
  processor.onaudioprocess = (e) => {
    const f32 = e.inputBuffer.getChannelData(0);
    const i16 = new Int16Array(f32.length);
    for (let i = 0; i < f32.length; i++) {
      i16[i] = Math.max(-1, Math.min(1, f32[i])) * 0x7fff;
    }
    ws.send(i16.buffer);
  };
  source.connect(processor);
  processor.connect(ctx.destination);
};

ws.onmessage = (event) => {
  if (typeof event.data === "string") {
    const msg = JSON.parse(event.data);
    if (msg.type === "transcript") console.log("User said:", msg.text);
    if (msg.type === "error") console.error(msg.code, msg.message);
  }
  // Binary frames are the agent's MP3 reply audio.
};
```
```python [Python]
import asyncio
import json
import wave

import websockets

async def realtime_stt():
    uri = "wss://api.callmissed.com/ws/voice-agent"
    async with websockets.connect(
        uri, subprotocols=["token", "cm_your_key"]
    ) as ws:
        await ws.send(json.dumps({"type": "config", "language": "hi-IN"}))

        async def listen():
            async for message in ws:
                if isinstance(message, str):
                    data = json.loads(message)
                    if data["type"] == "transcript":
                        print(f"Transcript: {data['text']}")

        listener = asyncio.create_task(listen())

        # recording.wav must be 16 kHz, mono, 16-bit PCM.
        with wave.open("recording.wav", "rb") as wav:
            while chunk := wav.readframes(8000):  # 0.5 s of raw PCM
                await ws.send(chunk)
                await asyncio.sleep(0.5)

        await asyncio.sleep(5)  # let the last transcript arrive
        listener.cancel()

asyncio.run(realtime_stt())
```
:::

See the [Voice Agent](/docs/voice-agent) page for the full WebSocket protocol and all message types.
