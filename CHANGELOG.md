# Changelog

## 6.0.0 — 2026-06-16

**Single-skill repackage + Replicate backend.**

- Repackaged from 24 separately-registering sub-skills (`skills/seedance-*/SKILL.md`) into **one skill**. The root `SKILL.md` is now the only registered skill; all specialist modules were folded into `references/` as progressive-disclosure docs.
- Rewrote all `[skill:…]` and `[ref:…]` cross-references to relative `references/*.md` paths so the root skill can load any module on demand.
- Switched the execution backend to **Replicate**, model **`bytedance/seedance-2.0`** (the main model — never `bytedance/seedance-2.0-fast`). Added `references/replicate-api.md` and rerouted the operating loop, load map, and platform/API references to it.
- Removed multi-skill scaffolding and author tooling (`skills/`, `scripts/`, `docs/`, `agents/`, `evals/`, release manifests, legacy `references/migrated/`).
- Content adapted from [Emily2040/seedance-2.0](https://github.com/Emily2040/seedance-2.0) (upstream v5.5.2).
