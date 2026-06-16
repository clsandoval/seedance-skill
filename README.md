# seedance-skill

A single Claude Code skill for generating and directing cinematic **Seedance 2.0** AI video, executed on **Replicate** via the **`bytedance/seedance-2.0`** model — the main model, never the `-fast` variant.

Text-to-video, image-to-video, video-to-video, and reference-to-video workflows with `[Image1]`/`[Video1]`/`[Audio1]` reference tags, multi-character scenes, lip-sync/audio, IP-safe rewrites, and multilingual (zh/ja/ko/es/ru) prompting.

## One skill, progressive disclosure

This repo registers as **one skill**. The root `SKILL.md` is the only skill file; it routes. Everything else — the specialist modules (prompt, camera, motion, lighting, characters, style, VFX, audio, recipes, troubleshoot, copyright, antislop, filter, interview, vocab) and the dense reference tables — lives under `references/` and is loaded on demand.

```
SKILL.md                 # the single skill: operating loop + load map (always loaded)
references/              # everything else, loaded on demand
  replicate-api.md       # execution backend: auth, schema, upload, poll, retry, post
  seedance-prompt.md     # folded-in specialist modules (formerly separate skills)
  seedance-camera.md
  ...
  capability-map.md      # dense tables / volatile facts
  quick-ref.md
  ...
assets/                  # README imagery
data/                    # community/source snapshots used by source-registry.md
```

The content was adapted from [Emily2040/seedance-2.0](https://github.com/Emily2040/seedance-2.0), repackaged from 20+ separately-registering sub-skills into a single skill with reference-based progressive disclosure, and repointed at Replicate as the execution backend.

## Execution backend: Replicate

All generations run against the flagship **`bytedance/seedance-2.0`** model on Replicate. The `-fast` variant is never used — it degrades venue and reference fidelity. See `references/replicate-api.md` for the full playbook (env, asset upload to public URLs, prediction creation, polling, the E005 content-filter retry policy, and ffmpeg post-processing).

Set your token before generating:

```bash
export REPLICATE_API_TOKEN=...
```

## Install (Claude Code)

```bash
claude skills install https://github.com/clsandoval/seedance-skill
```

## License

MIT
