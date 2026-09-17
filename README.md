# HTML 2 Video for mcode

An agent skill that turns a topic, outline, or script into a **narrated MP4**: HTML slides with
staged entrance animations, a TTS voiceover, burned-in subtitles, and an ASR pass that verifies the
voiceover actually says what the script says.

Works in MiniMax Code (mcode) with its platform connectors, and in any other agent environment
through [`mmx-cli`](https://github.com/MiniMax-AI/cli).

## What it produces

A 60-second narrated video looks like this:

```
my-video/
├── slides/            8 HTML slides + tokens.css (13 themes, 17 layout recipes)
├── audio/             8 TTS clips
├── build/timings.json measured durations + per-layer entrance times
├── preview/*.png      terminal-state frames
└── out/
    ├── final.mp4      ★ the deliverable (1920×1080 or 1080×1920, H.264 + AAC)
    ├── subs.srt       subtitles for platform upload
    └── slide-*.mp4    per-slide segments
```

## Design decisions worth knowing

- **Nothing about timing is hand-written.** Every slide's duration and every animation's entrance
  time is derived from the TTS audio measured with ffprobe, so "the voiceover finished but the
  picture is still waiting" cannot happen by construction.
- **Every slide has a title layer and a detail layer**, on separate animation stages, so a slide is
  never just a big title with nothing to look at.
- **Animations land in the video.** Capture steps frames deterministically instead of screen-recording,
  so entrance animations are actually rendered rather than frozen at their end state.
- **Rendering is gated.** A static check refuses to render slides with undefined CSS variables,
  missing images, external resources, or entrance animations without an animation class — the
  failure modes that otherwise ship a video that looks broken while every script reports success.

## Install

```bash
# Claude Code / OpenClaw / other AgentSkills-compatible hosts
cp -r html2video-for-mcode ~/.claude/skills/          # or ~/.openclaw/skills/
# project-level
cp -r html2video-for-mcode <your-project>/.claude/skills/
# or from this repo
npx skills add Wzdhehe/html2video-for-mcode
```

Then install the two dependencies **in your video project** (not in the skill directory):

```bash
cd <your-video-project>
npm i playwright && npx playwright install chromium
# ffmpeg: winget install Gyan.FFmpeg / brew install ffmpeg / apt install ffmpeg
#         or: npm i ffmpeg-static ffprobe-static
```

## Quick start

```bash
node <skill>/scripts/init-project.mjs ./my-video --topic "My topic"
# fill research/notes.md and script.json (clauses = one line of narration each)
# synthesize audio/01.mp3 … audio/08.mp3
node <skill>/scripts/plan-timings.mjs ./my-video
node <skill>/scripts/check-slides.mjs ./my-video
node <skill>/scripts/capture.mjs ./my-video --mode motion
node <skill>/scripts/build-video.mjs ./my-video --asr
```

`SKILL.md` carries the full workflow (7 phases, 6 confirmation gates); `references/` holds the
authoring rules, asset-sourcing SOP, TTS/timing notes, and rendering internals.

## Requirements

- Node.js 18+ (plain ESM, no build step)
- ffmpeg / ffprobe (auto-detected: `PATH → project node_modules → skill parents → common locations`)
- Playwright Chromium for capture
- Voice synthesis: mcode connectors, `mmx-cli`, or your own TTS — the pipeline starts from the
  produced audio files
- Optional: `MINIMAX_API_KEY` for `scripts/asr.mjs` (speech recognition used to verify the voiceover)

## Network access

Only the steps you invoke reach the network: `scripts/asr.mjs` posts audio to
`api.minimaxi.com` / `api.minimax.io`; `scripts/fetch-official-images.mjs` opens the URL you pass;
voice synthesis goes through your chosen service. Timing, static checks, capture, encoding, and
theme validation are fully offline. No telemetry, no analytics.

## Troubleshooting

`SKILL.md` ends with a symptom → cause → fix table covering the failures this pipeline has actually
hit: silent tails after the voiceover, slides with nothing on them, elements entering at 0 seconds,
invisible text from undefined CSS variables, broken images, subtitles washed out on dark themes,
wrong voice language, and mismatched concat durations.

## License

MIT — see `LICENSE`. Portions of the design system are adapted from
[html-ppt-skill](https://github.com/lewislulu/html-ppt-skill) (MIT); see
[`THIRD-PARTY-NOTICES.md`](THIRD-PARTY-NOTICES.md) for the upstream notice.
