# seedance-skill

Claude Code skill for generating videos with [Seedance 2.0](https://replicate.com/bytedance/seedance-2.0-fast) via the Replicate API.

## What it does

- Download reference videos from TikTok/YouTube via yt-dlp
- Swap characters into existing video choreography using Seedance 2.0
- Stitch audio from original reference onto generated video

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

## Usage

Just ask Claude Code:

- "Download this TikTok and make my character do the same dance"
- "Generate a 10 second video of a robot walking through a city"
- "Animate this image into a video"
- "Swap the character in this video with my mascot"

## Learned the hard way

- **Use `seedance-2.0-fast`, not `seedance-2.0`** — the non-fast model rejects ALL reference video URLs ("must be provided as a web url") regardless of hosting. The fast model accepts the same URLs fine. This is an API-level issue, not a URL format problem.
- **Any public URL works for the fast model** — GitHub raw, CDN links, cloud storage signed URLs, temp file hosts. Just needs to be a direct download without auth.
- **Replicate file API URLs don't work** — they require auth headers to download, which Seedance's backend doesn't send
- **Content filters are aggressive** — disable `generate_audio` and use mild language if you get E005 errors
- **Audio stitching is usually needed** — generate video without audio, then mux the original audio track with ffmpeg
