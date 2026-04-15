# seedance-skill

Claude Code skill for generating videos with [Seedance 2.0](https://replicate.com/bytedance/seedance-2.0-fast) via the Replicate API.

## What it does

- Download reference videos from TikTok/YouTube via yt-dlp
- Swap characters into existing video choreography using Seedance 2.0
- Stitch audio from original reference onto generated video
- Deliver results via Telegram or local file

## Install

```bash
# Add to your Claude Code project
claude mcp add-skill /path/to/seedance-skill
```

Or clone and reference the SKILL.md path in your Claude Code settings.

## Requirements

- `REPLICATE_API_TOKEN` — [Get one here](https://replicate.com/account/api-tokens)
- `GITHUB_TOKEN` — For uploading temp assets (Seedance needs public URLs)
- `yt-dlp`, `ffmpeg`, `jq`, `curl`
- Optional: `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` for Telegram delivery

## Usage

Just ask Claude Code:

- "Download this TikTok and make my character do the same dance"
- "Generate a 10 second video of a robot walking through a city"
- "Animate this image into a video"
- "Swap the character in this video with my mascot"

## Learned the hard way

- **Use `seedance-2.0-fast`, not `seedance-2.0`** — the non-fast model rejects reference video URLs that the fast model accepts fine
- **Replicate file API URLs don't work** as Seedance reference inputs — ByteDance's backend can't fetch them (auth required). Use GitHub raw URLs instead
- **Temp file hosts don't work either** — catbox.moe, 0x0.st, tmpfiles.org all failed. GitHub raw is the reliable path
- **Content filters are aggressive** — disable `generate_audio` and use mild language if you get E005 errors
- **Audio stitching is usually needed** — generate video without audio, then mux the original audio track with ffmpeg
