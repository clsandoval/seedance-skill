---
name: seedance
description: Generate videos with Seedance 2.0 via Replicate — reference video + character swap, TikTok download, audio stitching
---

# Seedance — Video Generation Skill

Generate videos using ByteDance's Seedance 2.0 model via the Replicate API. Supports character swaps using reference videos and images, TikTok video downloading, and audio stitching.

**Announce at start:** "I'm using the seedance skill to [generate a video / download references / swap a character]."

## When to Use

- User wants to generate a video with Seedance 2.0
- User wants to swap a character into an existing video's choreography/motion
- User wants to download TikTok videos as reference clips
- User mentions "seedance", "video generation", "character swap", "dance video", "reference video"

## Required Environment

### API Keys (check before proceeding)

```bash
echo "REPLICATE_API_TOKEN=${REPLICATE_API_TOKEN:+SET}${REPLICATE_API_TOKEN:-UNSET}"
```

If `REPLICATE_API_TOKEN` is not set, stop and ask the user to set it.

### CLI Tools

```bash
which yt-dlp ffmpeg ffprobe jq curl
```

- `yt-dlp` — downloading reference videos from TikTok, YouTube, etc.
- `ffmpeg` / `ffprobe` — audio extraction, stitching, frame extraction, duration checks
- `jq` — JSON parsing for API responses
- `curl` — API calls

If any are missing, tell the user to install them.

## Workflow

### Phase 1: Gather Assets

The user provides some combination of:

1. **Reference video** — a URL (TikTok, YouTube, direct link) or local file path
2. **Character image** — a local file path or URL
3. **Prompt** — what the generated video should look like

#### Downloading Reference Videos

For TikTok/YouTube URLs, use yt-dlp:

```bash
yt-dlp --no-warnings -o "/tmp/seedance-work/reference.%(ext)s" "<URL>"
```

#### Analyzing Reference Videos

Extract frames to understand the video content for prompt writing:

```bash
ffmpeg -y -i reference.mp4 -vf "fps=1" -q:v 2 frame_%02d.jpg
```

Then read the frames to describe the motion, style, and content for the prompt.

### Phase 2: Upload Assets to Public URLs

**Seedance requires publicly accessible URLs for reference inputs.** Any direct-download URL works — GitHub raw URLs, cloud storage signed URLs, CDN links, etc. The key requirement is that the URL returns the file directly (no auth headers, no redirects to login pages).

```bash
# Base64 encode the file
python3 -c "
import json, base64
with open('LOCAL_FILE', 'rb') as f:
    b64 = base64.b64encode(f.read()).decode()
print(json.dumps({'message': 'tmp: seedance asset', 'content': b64}))
" > /tmp/upload-payload.json

# Upload via GitHub API
curl -s -X PUT "https://api.github.com/repos/OWNER/REPO/contents/tmp/FILENAME" \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/upload-payload.json | jq '{download_url: .content.download_url}'
```

The resulting URL format: `https://raw.githubusercontent.com/OWNER/REPO/BRANCH/tmp/FILENAME`

**Important:** Clean up these temp files after the job is done:
```bash
# Get the file SHA first
SHA=$(curl -s "https://api.github.com/repos/OWNER/REPO/contents/tmp/FILENAME" \
  -H "Authorization: token $GITHUB_TOKEN" | jq -r '.sha')

curl -s -X DELETE "https://api.github.com/repos/OWNER/REPO/contents/tmp/FILENAME" \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"cleanup: remove seedance temp file\", \"sha\": \"$SHA\"}"
```

### Phase 3: Generate Video

#### Model Selection

**Always use `seedance-2.0-fast`.** The non-fast `seedance-2.0` model rejects reference video URLs with "reference_video must be provided as a web url" regardless of hosting — Replicate files, temp hosts, GitHub raw, CDN links all fail. The fast variant accepts the same URLs without issue. This is an API-level difference, not a URL format problem.

```bash
# Get latest version
VERSION=$(curl -s "https://api.replicate.com/v1/models/bytedance/seedance-2.0-fast" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" | jq -r '.latest_version.id')
```

#### API Schema

```
POST https://api.replicate.com/v1/predictions
Authorization: Bearer $REPLICATE_API_TOKEN
Content-Type: application/json

{
  "version": "<version_hash>",
  "input": {
    "prompt": "string (required)",
    "image": "uri — first frame image (cannot combine with reference_images)",
    "last_frame_image": "uri — last frame (requires image)",
    "reference_images": ["uri", ...] — up to 9, for character/style consistency,
    "reference_videos": ["uri", ...] — up to 3, total max 15s, for motion transfer,
    "reference_audios": ["uri", ...] — up to 3, total max 15s, for lip-sync,
    "duration": int (-1 for auto, or 1-15 seconds, default 5),
    "resolution": "480p" | "720p" (default "720p"),
    "aspect_ratio": "16:9" | "4:3" | "1:1" | "3:4" | "9:16" | "21:9" | "adaptive",
    "generate_audio": bool (default true),
    "seed": int | null
  }
}
```

Reference tags in prompts: `[Image1]`, `[Video1]`, `[Audio1]`, etc.

#### Content Filter Avoidance

ByteDance's moderation can flag prompts. If you get error E005 ("flagged as sensitive"):

1. Remove words like "lip-sync", "screaming", "attack", etc.
2. Disable `generate_audio: false` — audio gen triggers filters more often
3. Use gentler language: "moves happily" instead of "dances wildly"
4. Retry — the filter is sometimes non-deterministic

#### Creating the Prediction

```bash
curl -s -X POST "https://api.replicate.com/v1/predictions" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg version "$VERSION" \
    --arg prompt "$PROMPT" \
    --arg video_url "$VIDEO_URL" \
    --arg char_url "$CHAR_URL" \
    '{
      version: $version,
      input: {
        prompt: $prompt,
        reference_images: [$char_url],
        reference_videos: [$video_url],
        duration: -1,
        resolution: "720p",
        aspect_ratio: "9:16",
        generate_audio: false
      }
    }')"
```

#### Polling

Seedance fast typically takes 2-4 minutes. Poll every 60-90 seconds:

```bash
curl -s "https://api.replicate.com/v1/predictions/$PRED_ID" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" | jq '{status, error, output}'
```

Statuses: `starting` → `processing` → `succeeded` | `failed`

On success, `output` contains a direct download URL for the generated MP4.

### Phase 4: Post-Processing

#### Audio Stitching

The generated video often has no audio (especially if `generate_audio: false`). Stitch audio from the original reference video:

```bash
# Get durations
GEN_DURATION=$(ffprobe -v error -show_entries format=duration -of csv=p=0 generated.mp4)
REF_DURATION=$(ffprobe -v error -show_entries format=duration -of csv=p=0 reference.mp4)

# Mux: take video from generated, audio from reference, trim to shorter duration
ffmpeg -y \
  -i generated.mp4 \
  -i reference.mp4 \
  -map 0:v -map 1:a \
  -c:v copy -c:a aac -b:a 192k \
  -t "$GEN_DURATION" \
  output.mp4
```

#### Audio Extraction Only

If the user just wants the audio track:

```bash
ffmpeg -y -i input.mp4 -vn -q:a 2 output.mp3
```

### Phase 5: Cleanup

Remove temporary files from GitHub:

```bash
for FILE in tmp/reference.mp4 tmp/character.jpg; do
  SHA=$(curl -s "https://api.github.com/repos/OWNER/REPO/contents/$FILE" \
    -H "Authorization: token $GITHUB_TOKEN" | jq -r '.sha // empty')
  [ -n "$SHA" ] && curl -s -X DELETE "https://api.github.com/repos/OWNER/REPO/contents/$FILE" \
    -H "Authorization: token $GITHUB_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"message\": \"cleanup: remove seedance temp\", \"sha\": \"$SHA\"}"
done
```

## Prompt Engineering Tips

### For Character Swaps (reference image + reference video)

```
The [character description] from [Image1] moves in the same style as [Video1].
[Background description]. [Motion description]. [Style description].
```

Keep it simple. Over-describing triggers content filters.

### For Text-to-Video (prompt only)

```
[Subject] [action] in [setting]. [Camera movement]. [Lighting]. [Style].
```

### For Image-to-Video (first frame)

Use the `image` parameter instead of `reference_images`. The image becomes the first frame and the model animates from there.

## Example Invocations

**"Make my character do the TikTok dance from this video"**
→ Download video with yt-dlp, get character image, upload both for public URLs, call Seedance with reference_images + reference_videos, stitch audio.

**"Generate a video of a cat walking through a garden"**
→ Text-to-video only, no references needed. Just prompt + Seedance API call.

**"Animate this image"**
→ Use the `image` parameter (first frame mode), not `reference_images`.

## Working Directory

All temporary files go to `/tmp/seedance-work/`. Create it at the start:

```bash
mkdir -p /tmp/seedance-work
```
