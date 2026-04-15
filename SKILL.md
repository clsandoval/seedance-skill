---
name: seedance
description: Generate cinematic videos with Seedance 2.0 via Replicate — iterate-fast/ship-main workflow, prompt corpus research, blind QA critique loop, reference video + character swap, TikTok download, audio stitching
---

# Seedance — Video Generation Skill

Generate cinematic videos using ByteDance's Seedance 2.0 model via the Replicate API. Iterate on `seedance-2.0-fast`, ship final renders on `seedance-2.0` main. Includes a prompt research corpus, blind cinematography QA critique loop, content filter intelligence, and support for character swaps, TikTok downloading, and audio stitching.

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

**Seedance requires publicly accessible URLs for reference inputs.** Any direct-download URL works — the key requirement is that the URL returns the file directly without auth headers.

Upload local files to a temporary file host to get a public URL:

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

Try each in order until one succeeds — availability of free file hosts varies. The URL just needs to be a direct download link accessible without authentication.

**Note:** Replicate file API URLs (`https://api.replicate.com/v1/files/...`) do NOT work — they require auth headers that Seedance's backend doesn't send.

### Phase 3: Generate Video

#### Model Selection — Iterate Fast, Ship on Main

Use a two-phase workflow:

1. **Iteration phase** — use `seedance-2.0-fast` for rapid prompt iteration. Cheaper, faster (2-4 min), good enough to evaluate composition, motion, and storytelling. Iterate here until the prompt produces good results.
2. **Final render** — once the prompt is locked, re-run on `seedance-2.0` (main) for maximum quality. Slower (~5-10 min) but higher fidelity output.

**Note on reference videos:** The non-fast `seedance-2.0` model rejects reference video URLs with "reference_video must be provided as a web url" regardless of hosting. For reference-video workflows, use `seedance-2.0-fast` for both phases.

```bash
# Iteration phase (fast)
VERSION=$(curl -s "https://api.replicate.com/v1/models/bytedance/seedance-2.0-fast" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" | jq -r '.latest_version.id')

# Final render (main — text-to-video and image-to-video only)
VERSION=$(curl -s "https://api.replicate.com/v1/models/bytedance/seedance-2.0" \
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

ByteDance's moderation flags ~37% of prompts (E005 "flagged as sensitive"). The filter is a language model evaluating intent, not a keyword blocker.

**Prompt length:** Prompts over 2000 chars are truncated. Keep under 1500 chars for safety.

**Confirmed trigger words/concepts (tested April 2026):**
- "robot" — consistently flagged even in innocent Pixar contexts. Use "character", "mascot", "creature" instead
- Violence language: "smashing", "slams", "barrage", "shrapnel", "projectile", "detonation", "explosion", "strikes", "white-hot"
- "command center", "control room" — may trigger military/surveillance flags
- "lip-sync", "screaming", "attack"
- Humanoid figures with glowing eyes in dark environments

**Safe alternatives:**
- Animals (cats, dogs) pass easily in identical scenarios where robots fail
- "taps" instead of "slams", "sends" instead of "fires", "arrives" instead of "strikes"
- Bright/warm environments pass more often than dark/moody ones
- Front-load cinematic language (camera types, lens specs) — signals professionalism to the filter

**Main model vs fast model filters:** The main `seedance-2.0` model has significantly stricter content filtering than `seedance-2.0-fast`. Prompts that pass on fast will frequently fail on main. For final renders on main, **dramatically shorten the prompt** — strip all cinematic direction, timestamps, and detail down to the bare concept (under 250 chars works best). The main model needs less guidance anyway since it produces higher quality output.

**Filter bypass strategies (from Emily2040/seedance-2.0 filter guide):**
1. Write like a filmmaker, not a casual user — production language gets more latitude
2. Chinese prompts have different filter thresholds and may pass where English fails
3. The filter evaluates the whole prompt as one scene — professional framing helps
4. Retry with different seeds — the filter is sometimes non-deterministic
5. For main model: use ultra-short prompts (under 250 chars), fire up to 3 attempts with different seeds

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

Remove local temporary files:

```bash
rm -rf /tmp/seedance-work/
```

Uploaded temp files on hosts like tmpfiles.org expire automatically (typically 1-24 hours).

## Iteration Workflow

The core loop is: **research → draft → generate on fast → extract frames → critique → revise → repeat until good → final render on main.**

### Step 1: Research the Prompt Corpus

Before writing any prompt, grep the local prompt library for structural templates matching the user's requested style:

```bash
grep -i -A 30 "STYLE_KEYWORD" .claude/skills/seedance/awesome-seedance-2-prompts/README.md
```

Use matching prompts as structural templates. Never invent prompt structure from scratch — adapt proven formats.

### Step 2: Draft and Storyboard QA

Write the prompt, then spawn a **blind storyboard QA subagent** BEFORE generating. The subagent must NOT know what product or brand this is for — it reviews purely as a film storyboard.

Pass the subagent ONLY the raw prompt text. It should critique:

- **Narrative coherence** — does the story make logical sense moment to moment? Would a viewer follow it?
- **Directional consistency** — if things move outward, do they keep moving outward? Any contradictions in spatial flow?
- **Time budget** — is each beat given enough seconds to register, or is it cramped? Is any beat given too long?
- **Transition logic** — are jumps between shots/scales smooth or jarring? Are there impossible camera moves?
- **Filter risk** — flag any words/concepts likely to trigger content moderation (violence, "robot", military, etc.)
- **Char count** — will this fit under 1500 chars after cleanup? If over, what to cut?
- **Ambiguity** — anything a video model could misinterpret? Vague descriptions that could go wrong?
- **The ending** — does the storyboard resolve, or does it just stop?

The subagent should give a verdict: **GENERATE** or **REVISE**, with specific line-level fixes if revise. Under 300 words.

After the storyboard QA passes, verify the prompt is under 1500 chars and fire on `seedance-2.0-fast`. Show the user the full prompt before firing.

### Step 3: Generate on Fast

Fire on `seedance-2.0-fast`. Poll every 60-90 seconds.

### Step 4: Extract Frames and Post-Render Critique

### Step 4 (cont.): Extract Frames and Post-Render Critique

Once the video succeeds, extract frames at 1fps and review every frame:

```bash
ffmpeg -y -i video.mp4 -vf "fps=1" -q:v 2 /tmp/seedance-work/frame_%02d.jpg
```

Read all frames. Spawn a **blind cinematography critique subagent**. The subagent must NOT know:
- What the video is for (no product/brand context)
- What the intended story beats were (no prompt shown)
- That it's AI-generated

The subagent reviews purely as a piece of animation filmmaking. It should read ALL extracted frames (1fps, every frame) and critique:

- **Composition** — framing, focal point clarity, visual balance per frame
- **Camera execution** — movement, angle choices, transitions between shots
- **Lighting & color** — palette, contrast, mood consistency
- **Story clarity** — can you understand what's happening with zero context? Describe the story you see
- **Logical consistency** — do elements behave consistently? If something moves left, does it keep going left? Do characters/objects follow coherent spatial logic throughout?
- **Pacing** — does the energy build? Any dead or wasted frames? Any repeated shots?
- **Production value** — Pixar theatrical or cheap mobile game ad?
- **Character design & consistency** — do characters hold together across frames? Proportions stable?
- **The ending** — does the final moment land? Is there a payoff or does it just stop?

The subagent should give letter grades (A-F) per category, identify the 3 weakest and 3 strongest frames by number, and give specific actionable changes a director would make. End with a one-word verdict: SHIP or ITERATE. Critique should be at the level of judging an Oscar-nominated animated short — ruthless and precise. Under 500 words.

### Step 5: Revise and Iterate

Based on the critique, revise the prompt. Common fixes:
- **Dead frames** → compress that time segment, give it action
- **Lost detail at wide shots** → don't pull back as far, specify "three-quarter overhead" not "full overhead"
- **Flat lighting** → specify color contrast (e.g., "cool blue vs warm gold")
- **No story payoff** → add a resolution beat (conflict → escalation → victory)
- **Monochrome mood** → add a contrasting color moment

Repeat Steps 3-5 until the critique passes. Re-run storyboard QA (Step 2) if the prompt changes substantially.

### Step 6: Final Render on Main

Once the prompt is locked on fast, re-run on `seedance-2.0` (main) for maximum quality:

```bash
VERSION=$(curl -s "https://api.replicate.com/v1/models/bytedance/seedance-2.0" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" | jq -r '.latest_version.id')
```

Same prompt, same parameters. Main model is slower (~5-10 min) but higher fidelity. Only use main for text-to-video and image-to-video — reference-video workflows must stay on fast.

## Prompt Engineering

### Prompt Library (local reference)

A full library of 1800+ curated community prompts should be cloned locally. If not present, clone it on first use:

```bash
git clone https://github.com/YouMind-OpenLab/awesome-seedance-2-prompts.git \
  .claude/skills/seedance/awesome-seedance-2-prompts
```

Then grep it for relevant examples matching the user's requested style/genre:

```bash
# Search for prompts by style (e.g., anime, cinematic, pixar, commercial, cyberpunk)
grep -i -A 30 "anime\|action\|combat" .claude/skills/seedance/awesome-seedance-2-prompts/README.md

# Search for structural patterns
grep -i -A 30 "\[00:00" .claude/skills/seedance/awesome-seedance-2-prompts/README.md
```

Use matching prompts as structural templates. Adapt the format and level of detail to the user's request — don't invent prompt structure from scratch.

### Prompt Structure (derived from top community prompts)

The best Seedance prompts follow this structure:

1. **Header line** — genre/style declaration + duration + quality tier
2. **Bracketed metadata sections:**
   - `【Core Focus】` or `【Style】` — the central concept and visual identity
   - `【Style】` — rendering quality, camera type, resolution, aesthetic references
   - `【Duration】` — total length
   - `【Scene】` — environment and atmosphere setup
3. **Timestamped shot breakdowns:**
   - `[00:00-00:05] Shot 1: Title · Subtitle (Emotional beat)`
   - `Visuals:` — what the camera sees
   - `Action:` — what happens / movement
   - `Special Effects Details:` — particles, lighting, VFX specifics
4. **Technical quality anchors** — scattered throughout: "8K sharp", "shallow depth of field", "cinematic", "no deformation or drift"
5. **Character consistency callouts** — "characters maintain consistent faces, clothing, and hairstyles throughout without deformation, drift, or artifacts"
6. **Negative constraints** — "no text, watermarks, or subtitles", "no gore"

### Key Tips

- **Character consistency**: Always include "maintain consistent faces, clothing, and hairstyles throughout without deformation, drift, or artifacts"
- **Motion quality**: Specify "natural subtle movements" to avoid robotic feel
- **Lip sync**: Use "lip synchronization is natural and precise" for dialogue scenes
- **Clean output**: Include "no text, watermarks, or subtitles" unless text is intentional
- **Depth of field**: "shallow depth of field, creamy blurred background" for cinematic look
- **Continuity**: "continuous motion, no cuts" for seamless sequences
- **Camera**: Specify camera type — "handheld", "tracking", "orbital", "FPV", "slow push-in"
- **Emotional beats**: Use parenthetical notes like `(Sense of charging)` or `(Ultimate clash)` to guide pacing

### For Character Swaps (reference image + reference video)

```
The [character description] from [Image1] moves in the same style as [Video1].
[Background description]. [Motion description]. [Style description].
```

Keep it simple. Over-describing triggers content filters.

### For Image-to-Video (first frame)

Use the `image` parameter instead of `reference_images`. The image becomes the first frame and the model animates from there.

### Content Filter Workarounds

If prompts get flagged (error E005), the prompt library has sanitized versions of intense scenes. Grep for the genre and look at how they phrase action/combat without triggering filters.

## Working Directory

All temporary files go to `/tmp/seedance-work/`. Create it at the start:

```bash
mkdir -p /tmp/seedance-work
```
