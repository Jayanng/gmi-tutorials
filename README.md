# GMI Cloud Tutorials

Practical guides for building with GMI Cloud, written from production use. All sample outputs in `samples/` were generated through the GMI inference API.

## Tutorials

1. **[Running the GMI Cloud MCP server on a headless VPS](01-headless-mcp-oauth.md)** — OAuth that actually completes without a browser, plus the two token bugs that break it
2. **[Spend gates and recovery](02-spend-gates-and-recovery.md)** — never pay twice for one generation: estimate-first gates, timeout recovery, per-request balance rules
3. **[GMI TTS voices compared](03-tts-voices-compared.md)** — Inworld vs MiniMax with real samples, per-character pricing, and the parameter rules that waste credits
4. **[Music generation on GMI](04-music-generation.md)** — MiniMax Music 3.0 for video music beds, with sample tracks and ffmpeg placement

## Samples

- `samples/tts/` — 12 real TTS outputs across Inworld and MiniMax voices
- `samples/music/` — 3 generated music tracks
- `samples/images/` — generated image frames (Gemini image model)

Written for the GMI Clouder program, October 2026. @gmi_cloud
