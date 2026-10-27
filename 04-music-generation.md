# Music Generation on GMI Cloud: MiniMax Music 3.0 in Practice

*Generating production-usable music beds through GMI's inference API, with real output samples and the workflow for putting them under a video.*

## What it is

GMI Cloud serves `minimax-music-3.0` for text-to-music generation. It is billed per generation, and it produces finished instrumental tracks, not just loops. This is useful for one thing in particular: **music beds for video production**, generated to fit a mood instead of hunted through stock libraries.

## Samples

Three real tracks generated through the GMI generation API, in `samples/music/`:

- `gmi-music-f810749a-...mp3` — 3.1 MB, 96.6 s
- `gmi-music-df126c5d-...mp3` — 2.5 MB, 77.8 s
- `gmi-music-9e299915-...mp3` — 1.0 MB, 30.0 s

Same pipeline as every GMI generation: `estimate_generation` (free) then `submit_generation`, results delivered as downloadable media URLs.

## Workflow

1. **Describe the mood and function, not instruments.** "Slow tense underscore for a documentary about ancient ruins, low strings, sparse percussion, 80 BPM" outperforms "strings and drums"
2. **Estimate first.** Music is billed per generation; the estimate call costs nothing
3. **Generate 2-3 candidates.** Music taste is subjective and the first roll is not always the best. The cost per track is low enough that candidates are the right default
4. **Place under video with ffmpeg**, at a level that never fights narration:

```bash
ffmpeg -i video.mp4 -i music.mp3 -filter_complex \
  "[1:a]volume=0.18[m];[0:a][m]amix=inputs=2:duration=first:dropout_transition=3[a]" \
  -map 0:v -map "[a]" -c:v copy out.mp4
```

5. **Trim internal silence before timing.** Like the TTS engines, creative music generation pads. Trim and fade:
```bash
ffmpeg -i music.mp3 -af "afade=t=out:st=115:d=2" music_faded.mp3
```

## Production notes

- **Ask for "no vocals" explicitly** if the track is going under narration. Instrumental by default is not guaranteed
- **Match duration to your cut.** If you need a 60-second bed, say so in the prompt; loops can be crossfaded in ffmpeg but a right-length generation sounds better
- **The recovery rule applies.** A client timeout on submit may still complete server-side. Check `list_generations` before rolling again (see `02-spend-gates-and-recovery.md`)
- Music results arrive as media file URLs. Download immediately and keep your own copy; treat the remote URL as temporary

## Where this fits in a video pipeline

The pattern that works end to end on GMI:

```
script (LLM) -> voice (TTS) -> visuals (image + image-to-video) -> music (Music 3.0) -> assembly (ffmpeg)
```

Every stage is one API family, one account, one spend gate. The music stage is the cheapest of all of them and adds the most "produced" feeling per cent.

## Cost reality

Music generation on GMI is billed per generation and priced in cents, not dollars. Generate candidates without anxiety, but keep the estimate-first gate: the discipline matters more than the amount.


---

*Disclosure: this tutorial is part of the GMI Clouder program. All GMI outputs shown are real generations from the author's own account. [insert GMI compliance disclosure line]*
