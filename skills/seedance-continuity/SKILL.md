---
name: seedance-continuity
description: 'Maintain visual consistency across sequential Seedance 2.0 clips using a shot-chain protocol. Manages reference frame extraction, slot allocation, and human-in-the-loop verification via Telegram. Use when building multi-clip sequences, when characters or environments drift between generations, or when stitching clips into a longer scene.'
license: MIT
user-invocable: true
user-invokable: true
tags: ["continuity", "shot-chain", "multi-clip", "consistency", "reference-management", "seedance-20"]
metadata: {"version": "1.0.0", "updated": "2026-04-16", "parent": "seedance-20", "author": "clsandoval", "repository": "https://github.com/clsandoval/seedance-skill"}
---

# seedance-continuity

Shot-chain protocol for multi-clip visual consistency in Seedance 2.0.

## Scope

- Reference frame extraction from generated clips
- Slot allocation strategy across identity, environment, and props
- Human-in-the-loop verification via Telegram
- Platform-specific reference strategies (Replicate, Dreamina, Volcengine)
- Shot-type-aware reallocation

## Out of scope

- Prompt writing — see [skill:seedance-prompt]
- Camera/motion per clip — see [skill:seedance-camera], [skill:seedance-motion]
- Character card creation — see [skill:seedance-characters]
- Style decisions — see [skill:seedance-style]
- Post-production stitching — see [skill:seedance-pipeline]

---

## Terminology

- **Anchor clip** — Clip 1. The creative foundation. All references are user-provided.
- **Extension clip** — Clip 2+. Inherits references from prior clips.
- **Anchor frames** — Curated frames extracted from a generated clip that capture identity, environment, and props for reuse.
- **Reference bank** — The accumulated set of approved anchor frames across all clips in the chain.
- **Shot chain** — The ordered sequence of clips being built.

---

## The Protocol

### Phase 0: Shot Chain Setup

Before generating anything, collect from the user:

1. **Character card(s)** — per [skill:seedance-characters] format. Fixed nouns, no renaming mid-chain.
2. **Style anchors** — 1-2 reference images or text descriptors.
3. **Environment description** — where the scene takes place.
4. **Prop list** — anything that must persist across clips (weapons, vehicles, furniture, logos).
5. **Shot list** — brief description of each planned clip. Can be vague. Just enough to anticipate reference needs.

Initialize the reference bank as empty.

---

### Phase 1: Anchor Clip

1. User provides references (up to 9 images) + prompt.
2. Generate clip 1 via the platform.
3. Extract **anchor frames** — one frame per category:
   - **Identity frame(s):** Clearest face/body shot of each character.
   - **Environment frame(s):** Widest shot showing the setting.
   - **Prop frame(s):** Clearest view of key props/objects.
4. Send anchor frames to Telegram with labels:
   ```
   Anchor frames for Clip 1:
   
   [Identity] Maya face — frame 47
   [Environment] Alley wide shot — frame 12
   [Prop] Sword detail — frame 83
   ```
5. Wait for user approval. If user requests different frames, re-extract and re-send.
6. Add approved frames to the **reference bank**.

---

### Phase 2: Extension Clips

Repeat for each subsequent clip:

#### Step 1: Allocate 9 reference slots

Default allocation:

| Slot | Category | Source |
|------|----------|--------|
| 1-3 | **Identity** | Character frames from the reference bank. Prioritize the anchor clip. Pull from most recent clip if character pose has changed significantly. |
| 4-6 | **Environment** | Setting frames from whichever prior clip best shows the current environment. If environment changes between clips, these become new creative input. |
| 7-8 | **Props** | Prop frames from whichever prior clip shows them most clearly. |
| 9 | **Flex** | Style reference, new creative input, or additional consistency frame based on what's weakest. |

Adjust allocation based on shot type (see Slot Reallocation table below).

#### Step 2: Propose allocation to user

Send to Telegram before generating:

```
Proposed references for Clip 3:

[1] Identity — Maya face (from Clip 1, frame 47)
[2] Identity — Maya full body (from Clip 2, frame 31)
[3] Identity — Maya 3/4 angle (from Clip 1, frame 62)
[4] Environment — Alley wide (from Clip 1, frame 12)
[5] Environment — Alley neon signs (from Clip 2, frame 5)
[6] Environment — Alley depth (from Clip 2, frame 44)
[7] Prop — Sword detail (from Clip 1, frame 83)
[8] Prop — Sword in hand (from Clip 2, frame 70)
[9] Flex — Style ref (original user-provided)

Approve? Or swap any slots?
```

#### Step 3: User approves or swaps

Wait for approval. Adjust if requested.

#### Step 4: Generate the clip

Use the approved 9 references + prompt. Follow [skill:seedance-prompt] for prompt construction.

#### Step 5: Extract and bank new anchor frames

Extract anchor frames from the new clip. Send to Telegram for approval. Add to the reference bank. These are now available for future clips in the chain.

#### Step 6: Repeat

Move to the next clip in the shot list.

---

## Slot Reallocation by Shot Type

The 3-3-2-1 default is not sacred. Adjust based on what the shot needs:

| Shot type | Identity | Environment | Props | Flex |
|-----------|----------|-------------|-------|------|
| **Wide / establishing** | 2 | 4 | 2 | 1 |
| **Close-up / dialogue** | 5 | 1 | 2 | 1 |
| **Action with props** | 2 | 2 | 4 | 1 |
| **Environment transition** | 2 | 1 (old) + 3 (new) | 2 | 1 |
| **Default** | 3 | 3 | 2 | 1 |

The agent proposes the allocation. The user approves via Telegram.

---

## Frame Selection Heuristics

When choosing which frame to extract from a clip:

**Identity:**
- Face most visible, well-lit, unobstructed.
- Prefer 3/4 angle over profile.
- Avoid frames where face is motion-blurred or partially occluded.

**Environment:**
- Widest shot with most background visible.
- Prefer early frames before camera movement changes the view.
- Look for frames that show depth (foreground + background separation).

**Props:**
- Prop largest in frame and clearly separated from character's body.
- Prefer frames where prop details (texture, shape, color) are sharpest.
- If prop is held, also grab a frame showing the grip/interaction.

**Avoid across all categories:**
- Motion-blurred frames
- Frames mid-transition or mid-cut
- Frames with visible artifacts or morphing
- Extreme close-ups (lose context) unless specifically needed for identity

**Frame extraction command:**
```bash
ffmpeg -i clip.mp4 -vf "select=eq(n\,FRAME_NUMBER)" -vframes 1 -q:v 2 frame_output.png
```

---

## Reference Bank Management

The reference bank grows with each clip. For a 3-4 clip chain, this is manageable. Track it as a simple list:

```
Reference Bank:
  Clip 1:
    - identity_maya_face.png (frame 47)
    - identity_maya_body.png (frame 62)
    - env_alley_wide.png (frame 12)
    - prop_sword.png (frame 83)
  Clip 2:
    - identity_maya_action.png (frame 31)
    - env_alley_neon.png (frame 5)
    - prop_sword_held.png (frame 70)
```

When selecting references for a new clip, scan the full bank. Don't default to only the most recent clip — sometimes the anchor clip has the best reference for a given category.

---

## Platform-Specific Notes

### Replicate (images only)

This is the primary workflow the protocol is designed for.
- All 9 slots are static image frames.
- No @Video references available.
- No @Material tags.
- Seed parameter not available on all models — don't rely on it.

### Dreamina (supports @Video + @Material)

- Substitute 1-2 image slots with a `@Video` reference from the prior clip for motion/style continuity.
- `@Material` tags can replace identity slots if the character was generated on-platform.
- Net effect: frees up image slots for more environment/prop anchoring.
- The slot allocation still applies — just count @Video and @Material as occupying their respective category slots.

### Volcengine API

- Same constraints as Replicate (images only).
- `seed` parameter available — use the same seed across clips in a chain for additional style reproducibility.
- Set seed on the anchor clip, reuse for extensions.

---

## Multi-Character Chains

With multiple characters, identity slots need to be split:

| Characters | Identity slots | Per character |
|------------|---------------|---------------|
| 1 | 3 | 3 |
| 2 | 4 (steal from flex + env) | 2 each |
| 3+ | Not recommended — identity will drift. Split into separate shot chains per character and composite in post. |

For 2-character scenes, every reference frame that shows a character should show ONLY that character. Mixed frames confuse identity anchoring.

---

## Agent Checklist

Per clip, the agent must:

- [ ] Check shot list for the current clip's description
- [ ] Determine shot type and select slot allocation
- [ ] Scan reference bank for best frames per category
- [ ] Propose allocation with specific frames to Telegram
- [ ] Wait for user approval
- [ ] Generate the clip
- [ ] Extract anchor frames from the new clip
- [ ] Send anchor frames to Telegram for approval
- [ ] Update reference bank with approved frames
- [ ] Confirm ready for next clip

Never generate a clip without user-approved references. Never skip the Telegram verification step.

---

## Agent Gotchas

1. **Don't burn all identity slots on the same angle.** Variety (face, 3/4, full body) gives the model more to work with than three copies of the same close-up.
2. **Environment frames from early in a clip are usually better.** Camera movement degrades background clarity over time.
3. **Props drift the most.** If a prop is critical to the scene, consider dedicating 3 slots to it and reducing environment.
4. **The anchor clip's frames are gold.** They were generated from the user's original creative input with no accumulated drift. Prefer them when possible.
5. **Style/mood is the most forgiving category.** It can usually be maintained through prompt text alone without burning a reference slot. That's why flex is only 1 slot.
6. **Never rely solely on the last frame of a prior clip for identity.** Re-read [skill:seedance-characters] — last frames accumulate drift. Go back to the reference bank.
