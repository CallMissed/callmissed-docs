---
title: "Voice SDKs"
description: "Python and JavaScript SDKs for building voice agents."
slug: "voice-sdk"
breadcrumb: "API Guides & Tutorials > Voice Agent"
---

# Voice SDKs

Python and JavaScript SDKs for building voice agents.

## Python SDK

Install:

```bash
pip install callmissed
```

### Quick Start

```python
import asyncio
from callmissed import CallMissed, SessionConfig

async def main():
    async with CallMissed(jwt_token="your_token") as client:
        # Create session
        session = await client.voice.create_session(
            SessionConfig(
                system_prompt="You are a helpful assistant.",
                voice="shubh",
                llm_model="sarvam-30b",
            )
        )

        # Connect and stream
        async with client.voice.connect(session) as ws:
            ws.on("transcript", lambda t: print(f"User: {t}"))
            ws.on("agent_text", lambda t: print(t, end=""))
            ws.on("audio", lambda chunk: play_audio(chunk))
            ws.on("error", lambda msg: print(f"Error: {msg}"))

            await ws.send_audio(pcm_bytes)
            await asyncio.sleep(60)

        # Get transcript
        turns = await client.voice.get_transcript(session.id)
        for turn in turns:
            print(f"User: {turn.user_transcript}")
            print(f"Agent: {turn.agent_response}")

asyncio.run(main())
```

### Events

| Event | Data | Description |
|-------|------|-------------|
| `ready` | None | Pipeline initialized |
| `transcript` | str | User speech transcribed |
| `agent_text` | str | Streaming LLM token |
| `llm_response` | str | Complete response |
| `audio` | bytes | MP3 audio chunk |
| `audio_start` | None | Audio starting |
| `audio_end` | None | Audio complete |
| `interrupted` | None | Agent interrupted |
| `error` | str | Error message |

## Browser Client (JS)

Install:

```bash
npm install @callmissed/voice
```

### Quick Start

```typescript
import { VoiceSession } from "@callmissed/voice";

// Get wsUrl and token from your backend (POST /v1/voice/sessions)
const session = new VoiceSession({ wsUrl, token });

session.on("transcript", (text) => console.log("User:", text));
session.on("agentText", (text) => console.log("Agent:", text));
session.on("stateChange", (state) => updateUI(state));
session.on("error", (msg) => console.error(msg));

await session.connect(); // Starts mic + WebSocket

// Later...
session.disconnect();
```

### API

- `new VoiceSession({ wsUrl, token, sampleRate?, echoCancellation? })`
- `session.connect()` — Start mic and WebSocket
- `session.disconnect()` — Stop everything
- `session.on(event, handler)` — Register event handler
- `session.getState()` — Current state: idle | connecting | listening | speaking
- `session.getMicVolume()` — Current mic volume (0-1)