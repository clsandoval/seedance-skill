---
name: seedance-20
description: "Generate, direct, and troubleshoot Seedance 2.0 cinematic AI video — executed on Replicate via the bytedance/seedance-2.0 model (the main model, never the -fast variant). Covers text/image/video/reference-to-video prompts, first/last frame, dialogue, lip-sync and audio, multi-character scenes, IP-safe rewrites, and zh/ja/ko/es/ru prompt work. Use when making AI video, writing Seedance prompts, directing a scene, fixing generation errors, or building an AI short film, ad, or music video. Not for non-Seedance models (Sora, Veo, Kling, Runway Gen) or image-only prompting."
license: MIT
user-invocable: true
user-invokable: true
tags: [ai-video, filmmaking, bytedance, seedance, replicate, multimodal, lip-sync]
metadata:
  version: "6.0.0"
  updated: "2026-06-16"
  author: "clsandoval"
  repository: "https://github.com/clsandoval/seedance-skill"
  execution-backend: "replicate"
  model: "bytedance/seedance-2.0"
---

# seedance-20

Seedance 2.0 operating loop for agent-directed video work. This is **one skill**: `SKILL.md` routes, and every specialist module and dense table lives under `references/` and is loaded on demand (progressive disclosure). Use this root to route, check facts, protect references, and keep prompts compact before loading the reference a task needs.

> **Execution backend: Replicate.** Every generation runs through the Replicate API against the **`bytedance/seedance-2.0`** model — the main model, **never** `bytedance/seedance-2.0-fast` (the fast variant degrades venue/reference fidelity). Load `references/replicate-api.md` for the auth, schema, upload, polling, retry, and post-processing playbook before executing any generation. Other surfaces (Dreamina, Jimeng, Volcengine/Ark, fal, Runway's Seedance route) remain documented in the references as background and for cross-checking capabilities, but they are not the execution path here.

## Soul

This skill exists so that a person who arrives with a feeling leaves with a film. Three principles govern everything below:

1. **Hear the intent behind the words.** Users describe outcomes ("make it feel like home"), not parameters. Every gate and sub-skill translates feeling into craft; none of them may hand the translation work back to the user.
2. **Keep the story alive.** Hold a story state across the conversation: subject, mode, look, references, decided constraints, and what failed before. Every skill reads it before asking anything and updates it after acting. A user should never have to repeat a decision, and a new request inherits the world already built.
3. **Evolve with the user.** Speak plainly to a beginner and in director language to a professional - and notice when the same user grows from one into the other across a project. The register adapts; the standards never do.

## Operating Loop

1. Intake: identify the user's goal, production phase, target surface, mode, duration, aspect ratio, references, audio needs, deliverables, and safety/IP risks. If intake surfaces a clear safety, IP, likeness, or evasion risk, jump straight to the safety gate (step 8) before any planning.
2. Source gate: before platform claims, load `references/api-status.md` and `references/source-registry.md`. Before executing a generation, load `references/replicate-api.md` (the `bytedance/seedance-2.0` main-model backend). For Runway, Volcengine, or fal background, also load `references/platform-surface-matrix.md`.
3. Professional gate: if the user asks for film, ad, campaign, client, delivery, localization, color, sound, subtitle, post, QC, or multi-shot work, load `references/pro-filmmaking-standards.md` before drafting.
4. Mode gate: choose T2V, I2V, V2V, R2V, FLF2V, edit, extend, or troubleshoot before writing prose.

   Mode availability is surface-specific: edit and extend exist on Dreamina and Ark routes; fal has no dedicated extend endpoint - to continue a clip on fal, prefer reference-to-video with the previous clip as a video reference (keeps motion and audio context), and chain image-to-video from its last frame as the fallback.

5. Capability check: when planning any shot, mode, or budget, load `references/capability-map.md` to design into model strengths and around known limits, and `references/allocation-model.md` to decide where the prompt spends its fidelity budget before drafting.
6. Reference map: assign every asset one primary role: identity, first frame, last frame, product, environment, motion, camera, timing, audio, or style. State what must not transfer.
7. Multilingual gate: if the prompt uses Chinese, Russian, Japanese, Korean, Spanish, or code-mixed wording, load `references/multilingual-community-examples.md` and preserve reference tags exactly.
8. Safety gate: route IP, likeness, voice, brand, real-person, graphic, or evasion-like wording through `references/seedance-copyright.md` or `references/seedance-filter.md`.
9. Prompt build: route to `references/seedance-interview.md`, `references/seedance-prompt.md`, `references/seedance-prompt-short.md`, or a domain skill for camera, motion, audio, characters, VFX, style, recipes, or pipeline.
10. Quality pass: run anti-slop, check one visible beat, one primary camera move, physical light, sound intent, continuity anchors, constraints, delivery caveats, and source-date caveats.
11. Repair loop: when a take returns, triage it with `references/retake-protocol.md` (keep / fix in post / edit / re-roll / rewrite, one variable per retake, inside an attempt budget); if it fails outright, diagnose root cause before adding adjectives via `references/seedance-troubleshoot.md`.

## Load Map

| Situation | Load |
|---|---|
| Vague idea or missing brief | `references/seedance-interview.md` or `references/seedance-interview-short.md` |
| Production prompt | `references/seedance-prompt.md`, `references/quick-ref.md`, `references/prompt-examples.md` |
| Planning any shot, mode, or budget | `references/capability-map.md` |
| Where the prompt spends fidelity: identity vs motion vs scene density | `references/allocation-model.md`, `references/intent-vs-precision.md` |
| Multi-shot prompt, cuts inside one clip, or shots-per-duration budget | `references/multishot-grammar.md` |
| 2D, anime, or cel-style motion | `references/2d-anime-grammar.md`, `references/seedance-style.md` |
| Professional film, commercial, campaign, or delivery workflow | `references/pro-filmmaking-standards.md`, `references/shot-list-continuity.md`, `references/delivery-qc.md` |
| Compact prompt or Chinese compression | `references/seedance-prompt-short.md`, language vocab reference |
| Camera, lens, blocking, shot contract | `references/seedance-camera.md`, `references/cinematography-shot-language.md` |
| Image reference / first frame | `references/i2v-guide.md`, `references/reference-workflow.md` |
| First and last frame | `references/first-last-frame-guide.md` |
| Execute a generation (auth, schema, upload, poll, retry, post) | `references/replicate-api.md` (main `bytedance/seedance-2.0`, never `-fast`) |
| Platform background — Runway, Volcengine, fal, pricing, model IDs | `references/seedance-pipeline.md`, `references/api-workflow.md`, `references/model-name-map.md` |
| Color, ACES, HDR/SDR, aspect ratio, subtitles, audio post, or QC | `references/color-pipeline-aces.md`, `references/aspect-ratio-delivery.md`, `references/subtitles-localization.md`, `references/audio-post-delivery.md`, `references/delivery-qc.md` |
| Genre template or examples | `references/seedance-recipes.md`, `references/examples-by-mode.md`, `references/genre-guides.md` |
| Chinese/Russian/Japanese/Korean/Spanish or mixed-language examples | `references/multilingual-community-examples.md`, language vocab reference |
| Slop-heavy or filter-tripping English wording | `references/seedance-vocab-en.md`, `references/seedance-antislop.md` |
| Bad result | `references/seedance-troubleshoot.md` |
| A take came back: keep, fix in post, edit, re-roll, or rewrite | `references/retake-protocol.md` |
| Why a rule works, or a novel case no rule covers | `references/model-mechanics.md` |

Preserve reference tags exactly, keep prompts short, and never convert field-observed community tricks into official platform guarantees. For professional filmmaker requests, deliver the workflow object the role needs: shot list, shot contract, continuity ledger, prompt, post handoff, localization plan, or QC checklist.
