# Replicate API for Seedance 2.0

Execute Seedance 2.0 generations via the Replicate API. This is the primary execution backend for this workspace.

## Environment

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

## Model Selection — Always Use Main

**Always use `seedance-2.0` (main model).** Never use `seedance-2.0-fast` — the quality difference is significant and fast produces worse venue/reference fidelity.

**Note on reference videos:** The main `seedance-2.0` model rejects reference video URLs. For reference-video workflows, this is a known limitation — use `image` (first frame) instead where possible.

```bash
VERSION=$(curl -s "https://api.replicate.com/v1/models/bytedance/seedance-2.0" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" | jq -r '.latest_version.id')
```

## API Schema

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

## Uploading Assets to Public URLs

**Seedance requires publicly accessible URLs for reference inputs.** Any direct-download URL works — the key requirement is that the URL returns the file directly without auth headers.

```bash
# Option 1: tmpfiles.org (preferred — returns direct download URL)
RAW_URL=$(curl -s -F "file=@LOCAL_FILE" https://tmpfiles.org/api/v1/upload | jq -r '.data.url')
# Convert to direct download URL
PUBLIC_URL=$(echo "$RAW_URL" | sed 's|tmpfiles.org/|tmpfiles.org/dl/|')

# Option 2: file.io (single-use download link)
PUBLIC_URL=$(curl -s -F "file=@LOCAL_FILE" https://file.io | jq -r '.link')

# Option 3: litterbox.catbox.moe (temp files, 1h-72h expiry)
PUBLIC_URL=$(curl -s -F "reqtype=fileupload" -F "time=24h" -F "fileToUpload=@LOCAL_FILE" https://litterbox.catbox.moe/resources/internals/api.php)
```

Try each in order until one succeeds.

**Note:** Replicate file API URLs (`https://api.replicate.com/v1/files/...`) do NOT work — they require auth headers that Seedance's backend doesn't send.

## Creating a Prediction

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

## Polling

Seedance typically takes 2-4 minutes. Poll every 60-90 seconds:

```bash
curl -s "https://api.replicate.com/v1/predictions/$PRED_ID" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" | jq '{status, error, output}'
```

Statuses: `starting` → `processing` → `succeeded` | `failed`

On success, `output` contains a direct download URL for the generated MP4.

## Parallel Predictions on Main Model

When shipping a prompt to main, fire **two predictions in parallel**:
1. **Full prompt** — exact copy of the prompt, same seed
2. **Intent-only prompt** — strip to bare creative intent under 250 chars (no timestamps, no camera direction, just the scene concept + style + references)

If both pass, compare results. If only one passes, use that. If both fail, apply E005 retry policy.

**Max 3 concurrent predictions** — never fire more than 3 at once.

## E005 Content Filter Retry Policy

The content filter is non-deterministic. The same prompt + references can fail then pass on retry. **NEVER** drop user-provided reference images or strip creative intent after a filter rejection. Instead:

1. Retry with a different `seed` value (e.g., 42, 123, 7777)
2. On each retry, swap synonyms on low-value words:
   - Camera verbs: "swoops" <-> "glides" <-> "sweeps" <-> "rises"
   - Tracking: "snaps forward" <-> "pushes forward" <-> "tracks forward"
   - Effects: "aura" <-> "glow" <-> "rim light" <-> "shimmer"
   - Mood: "intense focus" <-> "deep concentration" <-> "steady gaze"
   - Scale: "luminous" <-> "glowing" <-> "radiant" <-> "brilliant"
3. Fire 2-3 retries in parallel with different seed + synonym combos
4. Retry up to 10 times before considering any structural prompt changes
5. Never remove user-provided reference images

## Post-Processing

### Audio Stitching

```bash
GEN_DURATION=$(ffprobe -v error -show_entries format=duration -of csv=p=0 generated.mp4)
ffmpeg -y -i generated.mp4 -i reference.mp4 \
  -map 0:v -map 1:a -c:v copy -c:a aac -b:a 192k \
  -t "$GEN_DURATION" output.mp4
```

### Frame Extraction (for QA)

```bash
ffmpeg -y -i video.mp4 -vf "fps=1" -q:v 2 /tmp/seedance-work/frame_%02d.jpg
```

### TikTok / YouTube Download

```bash
yt-dlp --no-warnings -o "/tmp/seedance-work/reference.%(ext)s" "<URL>"
```

## Working Directory

All temporary files go to `/tmp/seedance-work/`:

```bash
mkdir -p /tmp/seedance-work
```

## Telegram Integration

Send finished videos back via NanoClaw bot for mobile review:

```bash
BOT_TOKEN="<from .env>"
curl -s -X POST "https://api.telegram.org/bot$BOT_TOKEN/sendVideo" \
  -F "chat_id=<CHAT_ID>" -F "video=@/tmp/seedance-work/output.mp4" -F "caption=<description>"
```
