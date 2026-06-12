---
title: "Real-time STT"
description: "Real-time speech-to-text transcription via WebSocket."
slug: "stt-realtime"
breadcrumb: "Speech to Text"
---

# Real-time STT

Real-time speech-to-text transcription via WebSocket.

## Overview

Real-time STT is available through the **Voice Agent WebSocket** pipeline. Audio is streamed as PCM s16le 16kHz mono, and transcripts are returned in real-time as the user speaks.

There is no standalone real-time STT WebSocket endpoint — real-time transcription is part of the full Voice Agent pipeline (STT → LLM → TTS).

For file-based transcription, use the [Speech to Text](/docs/speech-to-text) REST API.

## Via Voice Agent

Connect to `WS /ws/voice-agent`, send audio chunks, and receive `transcript` messages:

```json
{"type": "transcript", "text": "Hello, how are you?", "is_final": true}
```

## Example

:::tabs
```javascript [JavaScript]
const ws = new WebSocket(
  "wss://api.callmissed.com/ws/voice-agent?key=cm_your_key"
);

ws.onopen = () => {
  // Send configuration
  ws.send(JSON.stringify({
    type: "config",
    bot_id: "your-bot-id",
    stt_language: "hi-IN",
    tts_voice: "shubh",
  }));

  // Stream audio from microphone
  navigator.mediaDevices.getUserMedia({ audio: true }).then((stream) => {
    const recorder = new MediaRecorder(stream, { mimeType: "audio/webm" });
    recorder.ondataavailable = (e) => ws.send(e.data);
    recorder.start(250); // send chunks every 250ms
  });
};

ws.onmessage = (event) => {
  if (typeof event.data === "string") {
    const msg = JSON.parse(event.data);
    if (msg.type === "transcript") {
      console.log("User said:", msg.text);
    } else if (msg.type === "llm_token") {
      process.stdout.write(msg.token);
    }
  } else {
    // Binary data = TTS audio chunk (MP3)
    playAudio(event.data);
  }
};
```
```python [Python]
import asyncio
import websockets
import json

async def realtime_stt():
    uri = "wss://api.callmissed.com/ws/voice-agent?key=cm_your_key"
    async with websockets.connect(uri) as ws:
        # Send config
        await ws.send(json.dumps({
            "type": "config",
            "bot_id": "your-bot-id",
            "stt_language": "hi-IN",
        }))

        # Send audio file as chunks
        with open("recording.wav", "rb") as f:
            while chunk := f.read(16000):  # 0.5s chunks at 16kHz
                await ws.send(chunk)
                await asyncio.sleep(0.25)

        # Listen for transcripts
        async for message in ws:
            if isinstance(message, str):
                data = json.loads(message)
                if data["type"] == "transcript":
                    print(f"Transcript: {data['text']}")

asyncio.run(realtime_stt())
```
:::

See the [Voice Agent](/docs/voice-agent) page for the full WebSocket protocol and all message types.