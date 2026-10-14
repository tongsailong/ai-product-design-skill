# Security & Data-Flow Statement

This document exhaustively declares every network destination, credential touchpoint, subprocess, and deletion this skill can perform, so that users and automated security reviewers can verify the claims against the code.

**TL;DR: the core pipeline (HTML design → render → MP4/PDF/PPTX export) runs 100% locally with zero network calls and zero API keys. All cloud-touching code is isolated in `scripts/cloud/`, is strictly optional, uses only keys you supply, sends data only to the corresponding vendor's official API, and refuses to run without explicit consent (`--yes` flag or `HUASHU_CLOUD_OK=1`). There is no telemetry. No data is ever sent to any server controlled by the skill author.**

## Complete list of network destinations

| Host | Where | What is sent | When |
|---|---|---|---|
| `ark.cn-beijing.volces.com` (Volcengine Ark, ByteDance official API) | `scripts/cloud/ai-review-video.py` | Compressed segments of **your own rendered video**, for AI quality review, authenticated with **your own** `ARK_API_KEY` | Only when you run it, and only after the consent gate |
| `openspeech.bytedance.com` (ByteDance official TTS API) | `scripts/cloud/tts-doubao.mjs` (also invoked by `scripts/narrate-pipeline.mjs`) | The narration text you want synthesized, with **your own** key. The endpoint is validated against a hardcoded hostname allowlist (`*.bytedance.com` / `*.volces.com`) — a tampered `.env` cannot redirect your key or text elsewhere | Only when you run it, and only after the consent gate |
| `commons.wikimedia.org` (official Wikimedia API) | `scripts/fetch_images.py` | Image search keywords; downloads CC/public-domain images with license info printed for review | Only when the agent fetches stock imagery for a content design |
| Brand official websites, `simpleicons.org`, Google favicon service | `references/brand-asset-protocol.md` (instructions, no script) | Plain GET requests to download publicly served logos/brand assets | Only when you ask for a brand-specific design |
| `fonts.googleapis.com`, `unpkg.com` and similar CDNs | Static `<link>`/`<script>` tags inside demo/output HTML | Standard browser font/library fetches when *you* open a generated HTML file | Browser-side only; render scripts work offline-first |
| **OpenAI image generation** — *via the separate `huashu-gpt-image` skill; this repo ships no image script* | `references/gpt-visual-workflow.md` (instructions only, no script) | Your image prompts, sent by that external skill to an OpenAI-compatible images endpoint (locally brokered at `127.0.0.1` in the tested setup) | Only in the optional GPT visual-concept path, on the same principle as any normal paid-LLM call made within an agreed design task (see *Image generation* below) |

That is the entire list. `grep -rn "https://" --include="*.py" --include="*.mjs" --include="*.js" --include="*.sh" scripts/` to verify.

## API keys

- No key is hardcoded anywhere; the repo ships only `.env.example` placeholders (`.env` is gitignored).
- Keys are read from the **skill's own root `.env`** or process environment — never from files elsewhere on your machine. `ai-review-video.py` extracts only the single `ARK_API_KEY` variable; it does not load the rest of the file into the environment.
- Keys are transmitted exclusively to the corresponding vendor's official endpoint listed above, over HTTPS, as auth headers.
- `references/react-setup.md` option B (pasting an Anthropic key into a demo page input) is explicitly marked local-demo-only and not recommended; the default options require no key at all.

## Explicit consent gate

Both cloud scripts print exactly what will be sent to which host and exit before any network call unless you pass `--yes` or set `HUASHU_CLOUD_OK=1`. Everything else in this skill never needs the gate because it never leaves your machine.

## Image generation (GPT visual-concept workflow)

`references/gpt-visual-workflow.md` adds an **optional** GPT visual-concept path that generates concept images, HTML validation assets, and production image assets with an OpenAI image model. **This repo ships no image-generation script** — the actual call is performed by the separate `huashu-gpt-image` skill; this skill only defines *when* it may be used. Three boundaries apply:

- **Scope & cost.** Image generation is a **normal model capability**, used on the same principle as an everyday coding agent calling a paid LLM: when the task is a `needs-new-visual-direction` design task (or a verification/production-asset need within an already-agreed task), the agent **generates directly — no per-call paid approval is required**. It states which model, which route and whether it is billed (a *notice*, not an *approval*), records the model / image count / usage for traceability, and keeps cost sane: one image per direction by default, no "just to probe" generation, no unconditional retries, reference-image editing (`--ref`) for tweaks, reuse of already-generated assets. A genuinely larger scope (e.g. batch production assets) is handled as a **normal task-scope change** with the user — **not** by restoring per-image approval. The zero-cost probe (`--health`) still runs freely. If the generation path is unavailable the skill must **report the technical limit and pause that step** (and may hand over the finished prompts to run elsewhere) — it must **not** silently substitute placeholder images or code-drawn stand-ins for visuals the design depends on (which would silently corrupt the visual review).
- **Credentials.** This skill never stores, reads, writes, echoes, or commits image-generation keys. They are supplied to the external skill through its own environment / local proxy configuration, are never printed in reports, and are never embedded in generated HTML, images, or deliverables.
- **What is NOT relaxed.** Removing per-call approval does **not** permit changing CPA credentials, model routing or account-billing configuration, opening new paid services, widening file/system/network permissions, or bypassing any other authorization rule (Git push, release, production changes still require their own authorization — see the repo's Git and high-risk rules).

## Subprocesses

All subprocess calls invoke local media tools only: `ffmpeg`, `ffprobe`, `ffplay`, Playwright/Chromium for HTML rendering and screenshots. No shell-to-network combinations, no curl-pipe-sh patterns.

## File deletion

Recursive deletion is limited to temp directories the scripts themselves create with unique timestamp+PID names (`.video-tmp-*`, `.seek-tmp-*`, `_narration/.tmp`, Python `tempfile.TemporaryDirectory`). No script ever deletes user data or anything outside its own scratch space.

## Dependencies

Mainstream registry packages only (`playwright`, `sharp`, `pptxgenjs`, `pdf-lib`, `requests`), installed via standard `npm`/`pip`/`uv` — no binary downloads from arbitrary URLs. One documented exception to be aware of: `npx hyperframes init` (optional animation backend, see `references/hyperframes-backend.md`) installs 19 hyperframes documentation skills into `~/.claude/skills/`. This is called out with a warning in the docs before the command.

## Hooks

`scripts/design-gate-hook.sh` is **never installed automatically** — nothing in this skill writes to `settings.json`. If you manually opt in, its entire behavior is: block long-video render commands (exit 2) until a design-approval file exists. It makes no network calls, writes nothing, deletes nothing.

## Proxy handling note

`fetch_images.py` and `ai-review-video.py` disable inheriting proxy environment variables (`trust_env = False` / clearing `ALL_PROXY` etc.) for their own requests. This exists to survive stale local proxy configurations that break TLS — not to evade monitoring. If you need these requests to go through your proxy, set it explicitly in the script invocation.

## Reporting

Found something that contradicts this document? Please open an issue — a mismatch between this file and the code is treated as a bug.
