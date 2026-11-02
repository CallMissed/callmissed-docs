---
title: "Voices"
description: "Available voices for text-to-speech synthesis."
slug: "tts-voices"
breadcrumb: "Speech"
---

# Voices

Available voices for text-to-speech synthesis.

## Indic Voices

**bulbul:v3** provides **37 voices** spanning 11 Indian languages (`bn-IN`, `en-IN`, `gu-IN`, `hi-IN`, `kn-IN`, `ml-IN`, `mr-IN`, `od-IN`, `pa-IN`, `ta-IN`, `te-IN`). Pass the voice ID as the `voice` parameter and the target `language`. The default voice is `shubh`. An unrecognized voice falls back to the recommended voice for the request's `language` (for example `ratan` for `en-IN`, `ta-IN`, `mr-IN` and `gu-IN`, `shubh` for the rest).

```text
shubh · aditya · ritu · priya · neha · rahul · pooja · rohan · simran · kavya
amit · dev · ishita · shreya · ratan · varun · manan · sumit · roopa · kabir
aayan · ashutosh · advait · anand · tanya · tarun · sunny · mani · gokul · vijay
shruti · suhani · mohit · kavitha · rehan · soham · rupali
```

Preview every voice in the [Playground](https://platform.callmissed.com/playground/tts).

**bulbul:v4-flash** provides **222 persona voices** across 9 Indian languages (`hi-IN`, `en-IN`, `bn-IN`, `gu-IN`, `kn-IN`, `mr-IN`, `pa-IN`, `ta-IN`, `te-IN`) plus Hinglish (English-Hindi code-mixed) personas. Voice IDs follow `<name>_<language>_<style>`, for example `simran_en_customer` or `aparna_hi_customer`. The default voice is `shubh_enhi_ads`. The `bulbul:v3` names above (`shubh`, `priya` and the rest) are not valid on this model, and `ml-IN` and `od-IN` are not available on it; use `bulbul:v3` for those.

Default voice per language:

| Language | Default voice |
|----------|---------------|
| `hi-IN`, `en-IN` | `shubh_enhi_ads` |
| `bn-IN` | `arnab_bn_conversation` |
| `gu-IN` | `bhavik_gu_conversation` |
| `kn-IN` | `chaitra_kn_conversation` |
| `mr-IN` | `ishita_mr_conversational` |
| `pa-IN` | `anand_pa_conversation` |
| `ta-IN` | `gokul_ta_narration` |
| `te-IN` | `kavitha_te_conversation` |

The full list, grouped by language. `GET /api/v1/models` also returns it as `voice_names` on the `bulbul:v4-flash` entry.

#### Hindi (hi-IN) — 116 voices

```text
aayan_hi_conversational · amit_hi_conversational · ashutosh_hi_conversational · kabir_hi_conversational
kavya_hi_conversational · manan_hi_conversational · rahul_hi_conversational · sumit_hi_conversational
aditya_hi_conversational · aditya_hi_sales · anand_hi_documentary · anand_hi_news
aparna_hi_customer · aparna_hi_kyc · ashok_hi_character · ashok_hi_news
chhavi_hi_kids · ishita_hi_ads · ishita_hi_edtech · ishita_hi_banking
ishita_hi_ads_informal · ishita_hi_devotional · ishita_hi_numbers · ishita_hi_social
kunal_hi_kids · mahesh_hi_documentary · mani_hi_devotional · mani_hi_conversational
mohit_hi_conversational · nachiket_hi_devotional · priya_hi_recovery · ratan_hi_latenight
ratan_hi_customer_expressive · ratan_hi_documentary · ratan_hi_devotional · ratan_hi_recovery
ratan_hi_social · ratan_hi_sports · ratan_hi_latenight_warm · rehan_hi_social
ritu_hi_customer_utility · ritu_hi_kids · ritu_hi_conversation · ritu_hi_customer
ritu_hi_edtech · ritu_hi_ads_formal · ritu_hi_banking · ritu_hi_ads_informal
ritu_hi_insurance · ritu_hi_edtech_bot · ritu_hi_medical · ritu_hi_sales
ritu_hi_reels · ritu_hi_social · ritu_hi_customer_warm · ritu_hi_social_lively
roopa_hi_companion · roopa_hi_narration · roopa_hi_recovery · roopa_hi_market
roopa_hi_conversational · sanchita_hi_assistant · sanchita_hi_edtech · sanchita_hi_banking
sanchita_hi_feedback · sanchita_hi_ads_formal · sanchita_hi_ads_informal · sanchita_hi_interview
sanchita_hi_romantic · sanchita_hi_market · sanchita_hi_social · sanchita_hi_kyc
sarika_hi_conversation · shalini_hi_companion · shalini_hi_social · shreya_hi_conversational
shreya_hi_news · shruti_hi_edtech · shubh_hi_customer · shubh_hi_ecomm
shubh_hi_stories_mixed · shubh_hi_devotional · shubh_hi_ads · shubh_hi_recovery
shubh_hi_stories_dramatic · simran_hi_assistant · simran_hi_narration · simran_hi_automobile
simran_hi_conversation · simran_hi_news_breaking · simran_hi_social_energetic · simran_hi_social_excited
simran_hi_latenight · simran_hi_news · simran_hi_recovery · simran_hi_sales
suchitra_hi_ecomm · suhani_hi_social · sunny_hi_ads · sunny_hi_reels
tarun_hi_conversational · tarun_hi_sales · chaitra_hi_customer · shilpa_hi_narration
tanya_hi_narration · aarti_hi_customer · advait_hi_character · aryaman_hi_ads
chirag_hi_social · girish_hi_devotional · mukul_hi_ads · mukul_hi_suspense
suman_hi_companion · vaibhav_hi_social · vandana_hi_ecomm · vipul_hi_social
```

#### English (en-IN) — 62 voices

```text
aditi_en_stories · aparna_en_companion · aparna_en_edtech · ashwin_en_sports
ashwin_en_sports_energetic · chandrika_en_stories · dev_en_recovery · dev_en_conversational
deven_en_conversation · ishita_en_customer · ishita_en_medical · ishita_en_numbers
ishita_en_social · ishita_en_stories · kalpit_en_edtech · nachiket_en_ads
neha_en_customer · neha_en_latenight · nupur_en_kids · ojas_en_social
ritu_en_edtech · ritu_en_latenight · ritu_en_medical · ritu_en_reels
rohan_en_recovery · roopa_en_conversational · rustom_en_suspense · sanchita_en_companion
sanchita_en_insurance · sanchita_en_recovery · sanchita_en_market · sanchita_en_social
shabana_en_edtech · shalini_en_companion · shalini_en_customer · shubh_en_narration
shubh_en_numbers · shubh_en_ads · shubh_en_recovery · shubh_en_audiobook
shubh_en_narration_gentle · shubh_en_sports · simran_en_narration · simran_en_automobile
simran_en_conversation · simran_en_customer · simran_en_edtech · simran_en_edtech_bot
simran_en_sales · simran_en_recovery · simran_en_ads · simran_en_therapist
sunny_en_social · varun_en_ads · varun_en_suspense · zarina_en_conversation
amelia_en_conversational · sophia_en_conversational · girish_en_documentary · girish_en_devotional
payal_en_edtech · sarang_en_narration
```

#### Hinglish, English-Hindi code-mixed (hi-IN / en-IN) — 13 voices

```text
ishita_enhi_companion · ishita_enhi_customer · ishita_enhi_customer_expressive · sanchita_enhi_companion
shalini_enhi_companion · shalini_enhi_customer · shubh_enhi_companion · shubh_enhi_ads
shubh_enhi_banking · simran_enhi_companion · simran_enhi_customer · simran_enhi_banking_expressive
sunny_enhi_customer
```

#### Marathi (mr-IN) — 9 voices

```text
ishita_mr_conversational · mrunal_mr_narration · neha_mr_narration · nilesh_mr_conversation
ritu_mr_insurance · ritu_mr_narration · rupali_mr_stories · soham_mr_narration
mukul_mr_stories
```

#### Kannada (kn-IN) — 4 voices

```text
chaitra_kn_conversation · chaitra_kn_narration · chetan_kn_conversation · suchitra_kn_narration
```

#### Punjabi (pa-IN) — 4 voices

```text
anand_pa_conversation · anand_pa_customer · harpreet_pa_narration · jaspal_pa_banking
```

#### Tamil (ta-IN) — 4 voices

```text
gokul_ta_narration · aravind_ta_ads · aravind_ta_suspense · vijay_ta_narration
```

#### Telugu (te-IN) — 4 voices

```text
kavitha_te_conversation · kavitha_te_narration · pooja_te_conversation · tarun_te_narration
```

#### Gujarati (gu-IN) — 3 voices

```text
bhavik_gu_conversation · pooja_gu_conversational · pooja_gu_customer
```

#### Bengali (bn-IN) — 3 voices

```text
arnab_bn_conversation · roopa_bn_conversational · bimal_bn_suspense
```

**gnani-timbre-v3.0** (Gnani Timbre v3) provides **12 voices**. Every voice speaks all 10 Indian languages (`hi-IN`, `en-IN`, `ta-IN`, `te-IN`, `kn-IN`, `ml-IN`, `mr-IN`, `pa-IN`, `bn-IN`, `gu-IN`) plus `auto` and Hinglish `hi-en`. The default voice is `Shivani`; an unrecognized voice falls back to the default. The text also accepts [expressive audio tags](/docs/text-to-speech#gnani-timbre-v3-audio-tags).

| Voice | Style | Character |
|-------|-------|-----------|
| `Shivani` (default) | Feminine | Warm, patient, empathy-first voice for inbound customer support |
| `Lakshmi` | Feminine | Expressive, warm voice for storytelling and e-learning narration |
| `Kalpana` | Feminine | Empathetic, composed voice for customer care and insurance |
| `Chetana` | Feminine | Gentle, empathetic voice for healthcare and telecom customer service |
| `Vasudha` | Feminine | Expressive, composed voice for narration and e-learning |
| `Riyaa` | Feminine | Confident, lively voice for outbound sales and insurance |
| `Tarika` | Feminine | Warm, reassuring voice for banking and insurance customer care |
| `Avinash` | Masculine | Professional, firm voice with deliberate pacing for collections and BFSI |
| `Shubham` | Masculine | Energetic, confident voice for outbound sales and lead qualification |
| `Prakash` | Masculine | Steady, deliberate voice for payment recovery and BFSI |
| `Devansh` | Masculine | Energetic, young conversational voice for sales and fintech |
| `Yashraj` | Masculine | Warm narrative voice for e-learning and IVR narration |

## Cartesia Voices

**sonic-3.6** (Cartesia Sonic 3.6) speaks 44 languages with native-quality Hindi and Hinglish. Pass a featured handle (`skylar`, default) or any public Cartesia voice UUID as `voice`, plus a base ISO `language` code (`en`, `hi`). An unrecognized non-UUID handle falls back to `skylar`.

The live public library is paginated — do not treat the 16 featured aliases as the full set:

```bash
curl "https://api.callmissed.com/api/v1/models/sonic-3.6/voices?q=hindi&limit=50"
curl "https://api.callmissed.com/v1/audio/voices?model=sonic-3.6&q=skylar" \
  -H "Authorization: Bearer cm_YOUR_KEY"
```

`GET /api/v1/models/sonic-3.6/voices/{id}/preview` streams Cartesia's own pre-recorded sample (no synthesis, no credit charge). Neither listing endpoint synthesizes or bills.

Query params (both listing endpoints): `q` (search, max 200 chars), `language` (base code such as `hi`), `gender` (`masculine` / `feminine` / `gender_neutral`; `male` / `female` also accepted), `limit` (1–100, default 50), `starting_after` (a voice `id` from the previous page). `GET /v1/audio/voices` takes `model=sonic-3.6` (the default and the only accepted value; any other returns `400`).

```json
{
  "object": "list",
  "model": "sonic-3.6",
  "data": [
    {
      "id": "voice-uuid",
      "handle": "skylar",
      "name": "Skylar",
      "gender": "feminine",
      "language": "en",
      "locale": "en-US",
      "accent": "",
      "tagline": "",
      "description": "",
      "has_preview": true,
      "preview": "/api/v1/models/sonic-3.6/voices/voice-uuid/preview"
    }
  ],
  "has_more": true,
  "next_page": "…"
}
```

`handle` is set only for the featured aliases below (otherwise `null`); pass either the `id` or the handle as `voice` on `POST /v1/audio/speech`. `preview` is `null` when a voice has no sample. When `has_more` is `true`, pass the last voice's `id` as `starting_after` to fetch the next page.

If the upstream library is briefly unreachable, the first page answers `200` with the featured aliases below and an extra `"degraded": true` field instead of failing, so a voice picker still renders and every returned voice remains usable for synthesis. The field is **absent** on a normal response — treat its presence as "this is the short list, retry later for the full library". A request carrying `starting_after` is not degraded: pagination returns `503` so you keep the page you already have.

Featured aliases (stable handles, also valid UUIDs in the library):

```text
skylar · daniel · jacqueline · katie · cathy · caroline · ronald · carson · jameson
gemma · archie · riya · arushi · siya · parvati · kabir
```

`riya`, `arushi`, `siya`, `parvati`, and `kabir` are native Hindi speakers (pair with `"language": "hi"`). Browse the full library in the [console Voices page](https://console.callmissed.com/voice/voices), the [Playground](https://platform.callmissed.com/playground/tts), or [Talk](https://callmissed.com/talk).

> **Other TTS providers** also expose voices via the same `POST /v1/audio/speech` endpoint — **deepgram-aura-2** (90 voices across English, Spanish, German, French, Dutch, Italian, and Japanese, default `thalia`), **deepgram-aura-1** (12 legacy English voices at half the Aura-2 rate, default `asteria`), and **gpt-4o-mini-tts** (`alloy`, `ash`, `ballad`, `coral`, `echo`, `fable`, `nova`, `onyx`, `sage`, `shimmer`, `verse`, `marin`, `cedar`). See [Credits & Rate Limits](/docs/credits-rate-limits) for per-model pricing.

## MAI Voice (preview)

`mai-voice-2.1` ($0.2292 / 10K chars) and `mai-voice-2.1-flash` ($0.1563 / 10K chars) are Microsoft AI's text-to-speech models. **Both are preview.** They share the same 97 voices across 23 languages: `cs`, `da`, `de`, `en`, `es`, `fi`, `fr`, `hi`, `hu`, `id`, `it`, `ko`, `nb`, `nl`, `pl`, `pt`, `ro`, `ru`, `sv`, `th`, `tr`, `vi`, `zh`. They work on `POST /v1/audio/speech` and in voice agents and voice sessions (`tts_model`). `mai-voice-2.1-flash` is built for low latency and is the pick for live calls; `mai-voice-2.1` is the most expressive and suits narration and long-form audio.

Voice handles are lowercase `<locale>-<name>` (for example `hi-in-kavya`) and case-insensitive. A voice speaks only its own locale. With no `voice`, the first voice for the request `language` is used (`hi` gives `hi-in-arjun`), otherwise `en-in-priya`. Input is plain text: SSML and tags are not supported, and the text is read as written. The one voice-agent control is `tts_speed`, 0.5 to 2.0. `GET /api/v1/models?category=tts` returns every voice in `voice_names` and `voice_details`.

| Voice | Gender | Locale |
|-------|--------|--------|
| `hi-in-arjun` (Hindi default) | Masculine | Hindi |
| `hi-in-dhruv` | Masculine | Hindi |
| `hi-in-grant` | Masculine | Hindi |
| `hi-in-harper` | Feminine | Hindi |
| `hi-in-kavya` | Feminine | Hindi |
| `hi-in-priya` | Feminine | Hindi |
| `en-in-priya` (default) | Feminine | English (India) |
| `en-in-dhruv` | Masculine | English (India) |

They are not processed in India, so an agent with `hosting: "india"` cannot use them. They are paid-plan only, are never a default, cannot be used in `tts_fallbacks`, and are not offered on the public voice demo.

:::note
Preview models may change or be retired at short notice. If one is retired, its ID keeps working and is answered by a named successor, as described in [Retired and deprecated models](/docs/models#retired-and-deprecated-models), so an agent that uses it keeps working but may sound different. Use a generally available model for production-critical agents.
:::

## Voice session voices

Six more text-to-speech models speak only inside a voice agent: set one as `tts_model` on an agent or a [voice session](/docs/voice-sessions-api). `POST /v1/audio/speech` returns `400` for them. The section also covers `deepgram-flux-tts`, which is valid on `POST /v1/audio/speech` as well. Pass the voice as `voice` (a handle from the tables below, case-insensitive) and a base ISO code as `language`.

Prices are per 10K characters. `GET /api/v1/models?category=tts` returns `voice_details` (gender, accent and description per voice), `language_codes`, `supports_ssml`, `supports_delivery_tags` and `audio_tags` for each of them, so a picker can be built from the API.

Markup support differs per model. A voice agent's prompt teaches its model the right form automatically; when you send text yourself, use this:

| Model | Delivery directions | Sounds | Pauses | Everything else |
|-------|---------------------|--------|--------|-----------------|
| `inworld-tts-2` | `<expression value="gently warm"/>` before a sentence, written in English | `<sound value="laugh"/>`: laugh, sigh, breathe, clear throat, cough | `<break time="500ms"/>` | — |
| `inworld-tts-2-flash` | Not performed (ignored) | Same as TTS-2 | `<break time="500ms"/>` | — |
| `fish-audio-drama-3` | `<expression value="warm"/>` | `<sound value="chuckling"/>`: laughing, chuckling, sighing | — | Write amounts and dates in words: the `₹` sign is read as dollars |
| `rime-coda` | — | — | — | Plain text; tags and SSML are read aloud. `spell(AB79X)` spells a code |
| `deepgram-flux-tts` | — | — | — | Plain text; tags and SSML are read aloud, and a `{pause}` control stops the audio |
| `gradium-tts-beta` | — | — | — | Plain text; tags are read aloud and `₹` is read as euros |
| `microsoft-neural-tts` | — | — | — | Plain text only: no SSML, no audio tags, no delivery tags. It reads rupee amounts and dates itself |

On `fish-audio-drama-3`, `gradium-tts-beta`, `rime-coda` and `deepgram-flux-tts`, voice agents rewrite amounts, slash dates (day first) and long digit runs into words before speaking, because these models misread them otherwise. Set `tts_normalize` to `false` on the agent to turn this off.

### `deepgram-flux-tts` — Deepgram Flux TTS

Deepgram's conversational TTS built for voice agents, $0.4688 / 10K chars. **266 voices in 42 languages**, and the only Deepgram model here that is also valid on `POST /v1/audio/speech` (see [Text to Speech](/docs/text-to-speech#deepgram-flux-tts)). 149 voices in 10 languages (including Hindi) are generally available; 117 voices in 32 languages (including Tamil, Telugu, Kannada, Marathi, Gujarati, Bengali and Urdu) are **Early Access**. Pass the voice as a bare name (`ajay`); each voice belongs to one language, so the voice sets the language. Leave `voice` out to get the default for `language` (`priya` for English, or a language Flux has no voice for).

Early Access languages read numbers and dates as written, so spell them out ("two thousand twenty-six", not "2026"); the speed control works for English voices only. Every voice in `GET /v1/models/deepgram-flux-tts` carries a `language` field.

| Code | Language | Voices | Status | Default voice |
|------|----------|--------|--------|---------------|
| `en` | English | 36 | GA | `priya` |
| `nl` | Dutch | 13 | GA | `anouk` |
| `fr` | French | 12 | GA | `arnaud` |
| `de` | German | 13 | GA | `benjamin` |
| `hi` | Hindi | 14 | GA | `ajay` |
| `it` | Italian | 11 | GA | `angela` |
| `ja` | Japanese | 4 | GA | `ayumi` |
| `pt` | Portuguese (Brazil) | 18 | GA | `bianca` |
| `ru` | Russian | 9 | GA | `andrei` |
| `es` | Spanish | 19 | GA | `aina` |
| `bn` | Bengali | 6 | Early Access | `afsana` |
| `bg` | Bulgarian | 2 | Early Access | `desislava` |
| `ca` | Catalan | 2 | Early Access | `jordi` |
| `zh` | Chinese (Mandarin) | 14 | Early Access | `bo` |
| `hr` | Croatian | 7 | Early Access | `dalibor` |
| `cs` | Czech | 1 | Early Access | `jakub` |
| `da` | Danish | 3 | Early Access | `pernille` |
| `et` | Estonian | 1 | Early Access | `triin` |
| `fi` | Finnish | 2 | Early Access | `kirsi` |
| `el` | Greek | 1 | Early Access | `kostas` |
| `gu` | Gujarati | 3 | Early Access | `bhavna` |
| `hu` | Hungarian | 4 | Early Access | `bence` |
| `id` | Indonesian | 5 | Early Access | `dewi` |
| `kn` | Kannada | 1 | Early Access | `manjunath` |
| `ko` | Korean | 6 | Early Access | `eunji` |
| `lv` | Latvian | 2 | Early Access | `agnese` |
| `lt` | Lithuanian | 2 | Early Access | `goda` |
| `ms` | Malay | 2 | Early Access | `haziq` |
| `mr` | Marathi | 3 | Early Access | `pooja` |
| `no` | Norwegian | 6 | Early Access | `erlend` |
| `pl` | Polish | 6 | Early Access | `bartosz` |
| `ro` | Romanian | 2 | Early Access | `cosmin` |
| `sk` | Slovak | 2 | Early Access | `dominika` |
| `sl` | Slovenian | 2 | Early Access | `gregor` |
| `sv` | Swedish | 7 | Early Access | `axel` |
| `tl` | Tagalog | 5 | Early Access | `jerome` |
| `ta` | Tamil | 4 | Early Access | `gokul` |
| `te` | Telugu | 3 | Early Access | `pavan` |
| `tr` | Turkish | 7 | Early Access | `dilek` |
| `uk` | Ukrainian | 2 | Early Access | `denys` |
| `ur` | Urdu | 2 | Early Access | `ambreen` |
| `vi` | Vietnamese | 2 | Early Access | `hoa` |

The English voices with a full description:

| Voice | Gender | Accent | Description |
|-------|--------|--------|-------------|
| `priya` (default) | Feminine | Indian | Self-assured and reassuring, with an empathetic, calm professionalism |
| `meena` | Feminine | Indian | Patient and reassuring, with a calm, professional empathy |
| `naveen` | Masculine | Indian | Knowledgeable and composed, with a clear, caring professionalism |
| `hannah` | Feminine | American | Articulate and self-assured, with a thoughtful, pleasant manner |
| `alexis` | Feminine | American | Poised and articulate, with a calm, compassionate tone |
| `cole` | Masculine | American | Friendly and lively, with an engaging, energetic style |
| `miles` | Masculine | American | Calm and sincere, with a clear, confident professional voice |
| `sienna` | Feminine | American | Polished and steady, with a warm, caring delivery |
| `cliff` | Masculine | American | Low-pitched and self-assured, with a calm, slightly gravelly quality |
| `brooke` | Feminine | American | Quick-paced and upbeat, with an intelligent, self-possessed feel |
| `haley` | Feminine | American | Professional and calm, with an empathetic, caring tone |
| `kit` | Masculine | British | Lively but composed, with a helpful, thoughtful streak |
| `colin` | Masculine | British | Warm and dependable, with an authoritative yet approachable presence |
| `gemma` | Feminine | British | Cheerful and kind, with an approachable, caring warmth |
| `sharon` | Feminine | Australian | Formal yet relaxed, with a calm, self-assured manner |
| `kai` | Masculine | Singaporean | Knowledgeable and calm, with a caring, precise professional style |

All 266 voice names are listed under [Flux TTS voices by language](#flux-tts-voices-by-language).

### `rime-coda` — Rime Coda

Rime's flagship conversational TTS, $0.5208 / 10K chars. 8 languages: `en`, `hi`, `es`, `fr`, `de`, `ja`, `pt`, `ar`. 13 voices; `nadi` and `taru` are native Hindi speakers. The default is `nadi`.

| Voice | Gender | Accent | Description |
|-------|--------|--------|-------------|
| `nadi` (default) | Feminine | Indian | A balanced female voice. Native Hindi |
| `taru` | Masculine | Indian | A balanced, professional sounding native Hindi speaker |
| `astra` | Feminine | American | Bright, lively California voice, quick and expressive |
| `luna` | Feminine | American | A bright, lively voice, energetic with a casual core |
| `lyra` | Feminine | American | A warm young American voice, bright and lively |
| `celeste` | Feminine | American | Chill Gen-Z American voice |
| `clementine` | Feminine | American | A warm, lively twenty-something American voice, polished and energetic |
| `eyre` | Feminine | American | A warm, friendly American voice, calm and easy to listen to |
| `vespera` | Feminine | American | A friendly, lively young Californian voice, bright and adaptable |
| `bancroft` | Masculine | American | A composed American voice, strong on professional and formal contexts |
| `cupola` | Masculine | American | A confident, energetic American voice, professional with warmth |
| `lawton` | Masculine | American | A warm American voice, calm and friendly throughout |
| `marlu` | Masculine | Australian | An older Australian voice; low, mature, unhurried |

### `inworld-tts-2` and `inworld-tts-2-flash` — Inworld TTS-2

Inworld's newest TTS. Every voice speaks every language: 94 base codes, including Hindi (`hi`), Marathi (`mr`), Bengali (`bn`), Tamil (`ta`), Telugu (`te`), Gujarati (`gu`), Kannada (`kn`), Malayalam (`ml`), Punjabi (`pa`), Urdu (`ur`) and Odia (`or`). `inworld-tts-2` is $0.2604 / 10K chars; `inworld-tts-2-flash` is the faster, lower-cost model at $0.1563 / 10K chars. Both take the same 14 voices; the default is `riya`. Inworld publishes gender and a description for only some voices, so the others show a dash.

| Voice | Gender | Accent | Description |
|-------|--------|--------|-------------|
| `riya` (default) | — | Indian | Hindi voice |
| `manoj` | — | Indian | Hindi voice |
| `priya` | — | — | — |
| `saanvi` | — | — | — |
| `aanya` | — | — | — |
| `shreya` | — | — | — |
| `aarav` | — | — | — |
| `arjun` | — | — | — |
| `ashley` | Feminine | American | Warm, natural American voice |
| `olivia` | Feminine | British | Upbeat, friendly British voice |
| `edward` | Masculine | American | Fast-talking, emphatic American voice |
| `diego` | Masculine | Mexican | Soothing, gentle Mexican voice |
| `sarah` | — | American | — |
| `lauren` | — | American | — |

### `fish-audio-drama-3` — Fish Audio Drama 3

Fish Audio's expressive TTS, $0.5208 / 10K chars. 13 languages (`en`, `zh`, `ja`, `de`, `fr`, `es`, `ko`, `ar`, `ru`, `nl`, `it`, `pl`, `pt`), no Indian languages. 7 voices, default `selene`.

| Voice | Gender | Accent | Description |
|-------|--------|--------|-------------|
| `selene` (default) | Feminine | American | Meditative voice |
| `laura` | Feminine | American | Confident narrator |
| `adrian` | Masculine | American | Friendly, casual American voice |
| `hannah` | — | American | Conversational specialist |
| `ethan` | — | American | Curious explainer |
| `sarah` | — | American | Engaged speaker |
| `jordan` | — | American | Motivational speaker |

### `gradium-tts-beta` — Gradium TTS

Gradium's newest TTS, $0.50 / 10K chars. 5 languages (`en`, `fr`, `de`, `es`, `pt`), no Indian languages. 13 voices, default `harper`.

| Voice | Gender | Accent | Description |
|-------|--------|--------|-------------|
| `harper` (default) | Feminine | American | Modern, confident and friendly voice with a standard American accent |
| `zoey` | Feminine | American | Playful, upbeat Gen Z energy with a standard American accent |
| `sunnie` | Feminine | American | A joyful young American voice, warm and bubbly, keeping support calls bright and light |
| `brooklyn` | Feminine | American | A warm, effusive young American voice that makes customers feel at home |
| `sterling` | Masculine | American | A warm, energetic American adult voice with theatrical flair |
| `garrett` | Masculine | American | A smooth, low-pitched American adult voice with easy confidence |
| `marcus` | Masculine | American | A high-energy, resonant American adult voice with unshakeable conviction |
| `tilly` | Feminine | British | A bright, welcoming British voice with the warmth of a great receptionist |
| `maeve` | Feminine | British | A sparky, attentive British voice that gets to the point with a smile |
| `archie` | Masculine | British | A bright, welcoming British voice, made for front-line support |
| `toby` | Masculine | British | A sparky, attentive British voice for fast, efficient support |
| `aoife` | Feminine | Irish | Bright and engaging Irish English voice with lively, natural pacing |
| `cormac` | Masculine | Irish | Bright and warm Irish English voice with cheerful, natural pacing |

### `microsoft-neural-tts` — Microsoft Neural voices

Microsoft's neural text-to-speech, processed in India, $0.1563 / 10K chars. 57 voices across 13 languages: `as`, `bn`, `en`, `gu`, `hi`, `kn`, `ml`, `mr`, `or`, `pa`, `ta`, `te`, `ur`. The default is `en-in-neerja`. Input is plain text, with no SSML, audio tags or delivery tags, and the model normalises rupee amounts and dates itself. The one control is speaking speed, `tts_speed` from 0.5 to 2.0 (default 1.0).

A voice speaks only its own locale (the handle starts with it, for example `hi-in-swara` speaks Hindi), except the four Indic voices (`en-in-aartiindic`, `en-in-arjunindic`, `en-in-neerjaindic`, `en-in-prabhatindic`), which speak English (India) plus Hindi, Tamil, Telugu, Gujarati, Marathi, Assamese, Punjabi, Odia, Malayalam, Kannada and Bengali. Gender is the value Microsoft publishes for each voice; Microsoft publishes a description for the English (US) and English (UK) voices only.

| Voice | Gender | Locale | Description |
|-------|--------|--------|-------------|
| `en-in-neerja` (default) | Feminine | English (India) | — |
| `en-in-aartiindic` | Feminine | English (India) | Speaks English (India) plus 11 Indian languages |
| `en-in-arjunindic` | Masculine | English (India) | Speaks English (India) plus 11 Indian languages |
| `en-in-neerjaindic` | Feminine | English (India) | Speaks English (India) plus 11 Indian languages |
| `en-in-prabhatindic` | Masculine | English (India) | Speaks English (India) plus 11 Indian languages |
| `en-in-aarav` | Masculine | English (India) | — |
| `en-in-aashi` | Feminine | English (India) | — |
| `en-in-aarti` | Feminine | English (India) | — |
| `en-in-arjun` | Masculine | English (India) | — |
| `en-in-ananya` | Feminine | English (India) | — |
| `en-in-kavya` | Feminine | English (India) | — |
| `en-in-kunal` | Masculine | English (India) | — |
| `en-in-prabhat` | Masculine | English (India) | — |
| `en-in-rehaan` | Masculine | English (India) | — |
| `hi-in-swara` | Feminine | Hindi | — |
| `hi-in-madhur` | Masculine | Hindi | — |
| `hi-in-aarav` | Masculine | Hindi | — |
| `hi-in-ananya` | Feminine | Hindi | — |
| `hi-in-aarti` | Feminine | Hindi | — |
| `hi-in-arjun` | Masculine | Hindi | — |
| `hi-in-kavya` | Feminine | Hindi | — |
| `hi-in-kunal` | Masculine | Hindi | — |
| `hi-in-rehaan` | Masculine | Hindi | — |
| `as-in-yashica` | Feminine | Assamese | — |
| `as-in-priyom` | Masculine | Assamese | — |
| `bn-in-tanishaa` | Feminine | Bengali | — |
| `bn-in-bashkar` | Masculine | Bengali | — |
| `gu-in-dhwani` | Feminine | Gujarati | — |
| `gu-in-niranjan` | Masculine | Gujarati | — |
| `kn-in-sapna` | Feminine | Kannada | — |
| `kn-in-gagan` | Masculine | Kannada | — |
| `ml-in-sobhana` | Feminine | Malayalam | — |
| `ml-in-midhun` | Masculine | Malayalam | — |
| `mr-in-aarohi` | Feminine | Marathi | — |
| `mr-in-manohar` | Masculine | Marathi | — |
| `or-in-subhasini` | Feminine | Odia | — |
| `or-in-sukant` | Masculine | Odia | — |
| `pa-in-vaani` | Feminine | Punjabi | — |
| `pa-in-ojas` | Masculine | Punjabi | — |
| `ta-in-pallavi` | Feminine | Tamil | — |
| `ta-in-valluvar` | Masculine | Tamil | — |
| `te-in-shruti` | Feminine | Telugu | — |
| `te-in-mohan` | Masculine | Telugu | — |
| `ur-in-gul` | Feminine | Urdu | — |
| `ur-in-salman` | Masculine | Urdu | — |
| `en-us-ava` | Feminine | English (US) | Pleasant, Caring, Friendly. Suited to chat, documentary |
| `en-us-andrew` | Masculine | English (US) | Confident, Authentic, Warm. Suited to chat, advertisement |
| `en-us-emma` | Feminine | English (US) | Cheerful, Light-hearted, Casual. Suited to e-learning, chat |
| `en-us-brian` | Masculine | English (US) | Sincere, Calm, Approachable. Suited to podcast, chat |
| `en-us-jenny` | Feminine | English (US) | Sincere, Pleasant, Approachable. Suited to news, assistant |
| `en-us-guy` | Masculine | English (US) | Light-hearted, Whimsical, Friendly. Suited to news, gaming |
| `en-us-aria` | Feminine | English (US) | Crisp, Bright, Clear. Suited to news, e-learning |
| `en-us-davis` | Masculine | English (US) | Soothing, Calm, Smooth. Suited to audiobook, meditation |
| `en-gb-sonia` | Feminine | English (UK) | Gentle, Soft. Suited to narration, news |
| `en-gb-ryan` | Masculine | English (UK) | Bright, Engaging. Suited to assistant, news |
| `en-gb-libby` | Feminine | English (UK) | Crisp, Bright, Clear. Suited to news, e-learning |
| `en-gb-maisie` | Feminine | English (UK) | Crisp, Cheerful, Bright. Suited to audiobook, narration |

## Flux TTS voices by language

The managed Voice Agent's Flux engine (`tts_engine: "flux"`) uses the same 266 voices as `deepgram-flux-tts`, in all 42 languages; it is billed inside the per-minute voice rate. Synthesis is turn-based and prosody carries across turns. The default is `priya`, an Indian-accented English voice, and with no voice set a call speaks its own language's default voice. An unrecognized voice falls back to that default.

English (36):

```text
alexis · bree · brittany · brooke · bruce · cliff · cole · colin · conor
donovan · drew · elise · gemma · haley · hannah · heather · jack · kai
kelsey · kit · maeve · marcelo · marcus · meena · meghan · miles · naveen
paige · priya · rufus · sean · sharon · sienna · tanner · wade · wes
```

Other languages (language code, then voices):

```text
nl: anouk · bram · dirk · fleur · irene · lotte · luuk · monique · niels · ruben · sanne · veerle · yvonne
fr: arnaud · benoit · brigitte · camille · chloe · florian · julien · mathieu · pierre · thierry · veronique · yasmine
de: benjamin · christian · janine · leonie · magdalena · moritz · nadine · rolf · sonja · sven · tobias · verena · yannick
hi: ajay · anjali · arjun · divya · jyoti · nikhil · pallavi · prachi · rahul · rajesh · ritu · shreya · sunita · vikram
it: angela · chiara · federico · giorgio · matteo · paola · rocco · sofia · stefano · vincenzo · vito
ja: ayumi · hayato · nozomi · sayaka
pt: artur · benedita · bianca · bruno · catarina · claudia · hugo · leandro · marcia · mariana · marta · mateus · miguel · pedro · renato · ricardo · roberto · rodrigo
ru: andrei · kirill · maxim · mikhail · natalia · nikolai · olga · polina · vladimir
es: aina · alfredo · carmen · cecilia · duvan · eduardo · ernesto · fernanda · fiorella · gustavo · jaime · martin · mauricio · patricia · pilar · rafael · tomas · valentina · ysabel
bn: afsana · ishrat · mamun · sabbir · shuvo · sumaiya
bg: desislava · gergana
ca: jordi · judit
zh: bo · chiaying · haoran · jian · jie · kuanyu · lijuan · manlok · shufen · wenbo · wenjing · yawen · yuting · zhen
hr: dalibor · danijela · domagoj · jasna · josipa · mihaela · vedran
cs: jakub
da: pernille · rasmus · rikke
et: triin
fi: kirsi · teemu
el: kostas
gu: bhavna · hetal · jignesh
hu: bence · eszter · kinga · tibor
id: dewi · fajar · fitri · rizky · teguh
kn: manjunath
ko: eunji · hyunwoo · mikyung · minjun · soojin · sungmin
lv: agnese · edgars
lt: goda · neringa
ms: haziq · izzah
mr: pooja · pratik · sneha
no: erlend · espen · ida · signe · silje · vegard
pl: bartosz · ewa · justyna · kacper · konrad · zuzanna
ro: cosmin · ioana
sk: dominika · juraj
sl: gregor · katja
sv: axel · charlotta · frida · ludvig · oskar · pontus · wilma
tl: jerome · jomar · liezel · noel · trisha
ta: gokul · kavya · sowmya · vinoth
te: pavan · ravi · sravani
tr: dilek · ecem · emre · gizem · mehmet · serkan · zeynep
uk: denys · solomiya
ur: ambreen · usman
vi: hoa · minh
```

The model exposes no expressive/emotion/style controls and does not interpret SSML.
