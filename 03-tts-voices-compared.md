# GMI TTS Voices Compared: Inworld vs MiniMax, With Real Samples

*A practical guide to picking and using text-to-speech voices on GMI Cloud, with side-by-side samples, per-character pricing, and the parameter rules that waste your credits when ignored.*

## The two engines

GMI Cloud serves several TTS families (including Resemble Chatterbox and earlier Inworld and MiniMax lines). This guide compares the two it uses most:

| | Inworld TTS 2 | MiniMax Speech 2.8 HD |
|---|---|---|
| Voices | 282 | Personality-named set |
| Billing | per character (about $0.000025/char list; a 130-char line is about $0.0033) | per 1000 chars (about $0.10/1k; a 130-char line is about $0.013) |
| Character | conversational, personality-rich | strongest "sounds human" narration |
| Length limit | roughly 350-750 chars per request before failures | long texts in one shot |
| Output | OGG_OPUS available directly | mp3/flac (convert for voice apps) |

There is also `inworld-tts-2-flash`, the cheap Inworld variant, good for drafts and bulk lines.

**Cost intuition (live-tested October 2026; re-check with `estimate_generation`):** a 130-character line costs about $0.0033 on Inworld versus about $0.013 on MiniMax. Per character Inworld is roughly 4x cheaper. MiniMax wins long-form on mechanics, not price: it takes long texts in a single request, while Inworld needs chunking on long scripts.

## The samples

All files in `samples/tts/` were generated on GMI Cloud:

- `gmi_blake_story_condensed.ogg` — Inworld Blake, CREATIVE delivery, storytelling
- `gmi_ronald.ogg`, `gmi_amina.ogg`, `gmi_chioma.ogg` — Inworld voice personalities
- `ng_voice_male.mp3`, `ng_voice_female.mp3` — Nigerian-accented Inworld voices (the voice enum includes Emeka, Tunde, Chioma, Folake, Amara, and Amina)
- `gmi_minimax_storyteller.mp3` — MiniMax `English_CaptivatingStoryteller` (the two MiniMax `.ogg` files below were converted from mp3 output with the rule-5 ffmpeg command)
- `gmi_minimax_deepgentleman.ogg` — MiniMax `English_Deep-VoicedGentleman`
- `gmi_minimax_magnetic.ogg` — MiniMax magnetic narration voice
- `n1.mp3`, `n2.mp3` — production narration clips from a finished piece

Listen to the same text across engines before choosing. The difference in warmth is audible in three seconds. Quality impressions and prices in this guide are the author's own from testing in October 2026, not official product claims.

## Parameter rules (each of these cost real debugging time)

1. **Voice IDs are an enum of exact strings.** Read the model's `voice_id` enum before submitting. Values are case-sensitive. Include `voice_id` explicitly on submit even when the schema shows a default
2. **`language` is a supported optional parameter (BCP-47).** Live testing in October 2026 priced `en-NG` and `en-US` requests normally. An earlier September 2026 run saw estimate rejections when a language tag was present; if an estimate rejects, try dropping the tag and letting the voice imply the locale
3. **Text goes in `parameters.text`** (and the top-level prompt). Other options (`audio_encoding`, `speaking_rate`, `instruction`) go inside `parameters`
4. **Inworld is documented to 2,000 characters per request, but long texts have failed in practice.** In our testing, requests past roughly 750 characters failed terminally and retries kept failing, while under about 350 characters was reliably safe. Treat the documented 2,000 as a ceiling, not a target: split long scripts into chunks, submit in parallel, join with ffmpeg:
   ```
   ffmpeg -i part1.ogg -i part2.ogg -filter_complex "[0:a][1:a]concat=n=2:v=0:a=1[out]" -map "[out]" -c:a libopus -b:a 48k joined.ogg
   ```
5. **For Telegram voice bubbles ask for `audio_encoding: "OGG_OPUS"` directly** on Inworld. MiniMax outputs mp3/flac, convert with `ffmpeg -i x.mp3 -c:a libopus -b:a 48k x.ogg`

## Making voices sound human

- A short `instruction` field works: *"speak naturally and warmly, like a person talking to a friend"*
- Inworld's `delivery_mode: "CREATIVE"` plus an emotional instruction makes storytelling voices carry real feeling
- For multi-clip narration, keep the same `voice_id`, a fixed `seed`, and `temperature` around 0.3 in every request. The voice and pace stay steady across clips
- Creative delivery pads with pauses: a "60 second" script can land at 77 seconds. Trim internal pauses in post if you need exact timing:
  ```
  ffmpeg -i in.ogg -af "silenceremove=stop_periods=-1:stop_duration=0.25:stop_threshold=-32dB" out.ogg
  ```

## Choosing fast

- **Narration for videos or stories:** MiniMax Captivating Storyteller (single-request long texts) or Inworld Blake with CREATIVE delivery (warmer delivery)
- **Character voices:** Inworld, no contest
- **Bulk/utility lines:** `inworld-tts-2-flash`
- **Nigerian-accented English:** Inworld has native voices (Emeka, Tunde, Chioma, Folake, Amara, Amina)

## The recovery rule applies here too

A client timeout on a TTS submit is not a failure. See `02-spend-gates-and-recovery.md`: recover via `list_generations` filtered to the model, then fetch by request_id. When recovering timed-out batches you cannot map request_id to voice directly; infer by creation order, and if any line was previously generated as a sample, match by exact duration.


---

*Disclosure: this tutorial was produced as part of the GMI Clouder program (paid). All GMI outputs shown are real generations from the author's own account.*
