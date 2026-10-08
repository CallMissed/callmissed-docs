---
title: "Voice Cloning"
description: "Create a voice from a short recording of a real person, with their consent, and use it as your voice agent's voice."
slug: "voice-cloning"
breadcrumb: "Voice Agents"
---

# Voice Cloning

Create a voice from a short recording of a real person, with their consent, and use it as your voice agent's voice.

## Overview

A cloned voice is made from 10-15 seconds of one person speaking. Once it exists, any of your voice agents can speak with it: set the agent's `voice` to `clone:<id>`.

- **Consent is required.** Every clone records who confirmed the speaker's consent, when, and which version of the consent statement they accepted.
- **The recording is not kept.** CallMissed stores only a fingerprint (SHA-256) of the sample. The audio goes to the speech engine solely to create the voice.
- **Paid plans only.** Voice cloning is not available on the Free plan.
- **Your clones are private to your account.** Another account cannot list, use or delete them.

> Voice cloning is being rolled out. Until it is enabled for your account, creating a clone returns `403` with `"Voice cloning is not available on this account yet."`, and an agent set to a cloned voice speaks with its model's default voice.

## Authentication

```
Authorization: Bearer cm_your_api_key
```

Reading clones needs the `bots:read` scope; creating and deleting them needs `bots:write`.

## Get the consent statement

Show this text to the person creating the clone, and send its `version` as `consent_version`. A request with any other version is refused with `422`, so your app always shows the current statement.

```bash
curl https://api.callmissed.com/api/v1/voice/clones/consent \
  -H "Authorization: Bearer cm_your_api_key"
```

```json
{ "version": "2026-10-08", "text": "I confirm that the voice in this recording belongs to ..." }
```

## Create a clone

`POST /api/v1/voice/clones` takes `multipart/form-data`:

| Field | Required | Notes |
| --- | --- | --- |
| `sample` | yes | The recording: one person speaking clearly, no music or background noise. WAV, MP3 or FLAC (`cartesia` also accepts OGG and WebM). Up to 10 MB. |
| `speaker_name` | yes | The name of the person whose voice it is (1-100 characters). Shown as the clone's name. |
| `consent_attestation` | yes | Must be `true`: you confirm the speaker consented. |
| `consent_version` | yes | The `version` from `GET /api/v1/voice/clones/consent`. |
| `language` | yes | The language spoken in the sample, e.g. `hi-IN`, `en-IN`, `ta`. |
| `provider` | no | The speech engine: `cartesia` (default) or `sarvam`. |
| `gender` | no | `male` or `female`. |

```bash
curl -X POST https://api.callmissed.com/api/v1/voice/clones \
  -H "Authorization: Bearer cm_your_api_key" \
  -F "sample=@asha.wav;type=audio/wav" \
  -F "speaker_name=Asha Rao" \
  -F "consent_attestation=true" \
  -F "consent_version=2026-10-08" \
  -F "provider=sarvam" \
  -F "language=hi-IN"
```

```json
{
  "id": "4f0c6b1e-2d7a-4a53-9a51-6f5d2c1b9e10",
  "voice": "clone:4f0c6b1e-2d7a-4a53-9a51-6f5d2c1b9e10",
  "name": "Asha Rao",
  "provider": "sarvam",
  "language": "hi-IN",
  "gender": null,
  "status": "active",
  "consent_version": "2026-10-08",
  "consent_at": "2026-10-08T09:14:03Z",
  "created_at": "2026-10-08T09:14:03Z"
}
```

### Sample length and languages

| `provider` | Sample length | Languages |
| --- | --- | --- |
| `cartesia` | 5-60 seconds (10 s is enough; up to 60 s keeps more of the accent) | Most major languages, including `en`, `hi`, `bn`, `ta`, `te`, `kn`, `ml`, `mr`, `gu`, `pa`, `or`, `ur` |
| `sarvam` | 5-15 seconds (10-15 s recommended) | `as-IN`, `bn-IN`, `en-IN`, `gu-IN`, `hi-IN`, `kn-IN`, `ml-IN`, `mr-IN`, `od-IN`, `pa-IN`, `ta-IN`, `te-IN` |

A `sarvam` clone can speak any of its languages, whatever language the sample was in.

## Use a clone on an agent

```bash
curl -X PATCH https://api.callmissed.com/api/v1/bots/{bot_id}/config \
  -H "Authorization: Bearer cm_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{ "values": { "voice": "clone:4f0c6b1e-2d7a-4a53-9a51-6f5d2c1b9e10" } }'
```

- The clone speaks on its own engine: a `cartesia` clone uses `sonic-3.6` and a `sarvam` clone uses `bulbul:v3`, whatever `tts_model` the agent has.
- It works on agents that use speech-to-text, an LLM and text-to-speech. It cannot be combined with a `voice_tier` or a speech-to-speech model; saving that combination returns `422`.
- Saving a `clone:` id that is not one of your active clones returns `422`.
- If a clone is deleted later, agents still set to it speak with their model's default voice. Calls are never dropped over it.
- Cloned voices are not available on the [Managed Voice Agent](/docs/managed-voice-agent) WebSocket yet.

Speech in a cloned voice is billed per character on top of the agent's text-to-speech rate.

## List and get clones

```bash
curl https://api.callmissed.com/api/v1/voice/clones \
  -H "Authorization: Bearer cm_your_api_key"

curl https://api.callmissed.com/api/v1/voice/clones/{clone_id} \
  -H "Authorization: Bearer cm_your_api_key"
```

The list returns your clones newest first (deleted clones are left out).

## Delete a clone

```bash
curl -X DELETE https://api.callmissed.com/api/v1/voice/clones/{clone_id} \
  -H "Authorization: Bearer cm_your_api_key"
```

The voice is removed here and at the speech engine. The response `status` is `deleted` once the engine confirmed, or `deleting` while CallMissed keeps retrying in the background. Delete a clone whenever the speaker withdraws consent. When an account is erased, all of its clones are deleted at the speech engine too.

## Errors

| Status | When |
| --- | --- |
| `402` | Not enough credits to create a clone |
| `403` | Free plan, or voice cloning not yet enabled for the account |
| `404` | No such clone in your account |
| `413` | The sample is larger than 10 MB |
| `415` | The file is not a supported audio format |
| `422` | Missing consent, an old `consent_version`, an unsupported language, a sample of the wrong length, or a sample the engine could not clone (too noisy, or more than one speaker) |
| `429` | The daily limit on new clones is reached; try again tomorrow |
| `503` | Voice cloning is temporarily unavailable |
