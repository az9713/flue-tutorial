# Project Context & Working Memory

> Durable context for continuing development on this repo after a context clear.
> Pointed to by the root `CLAUDE.md`. Read this first, then `AGENTS.md` (upstream
> dev/test conventions) and `docs/FLUE-DEMYSTIFIED.md` (the architecture deep-dive).

---

## 1. What this repo is

A **tutorial clone / enhanced fork of [`withastro/flue`](https://github.com/withastro/flue)**.

- **Upstream:** https://github.com/withastro/flue — "The Agent Harness Framework" (TypeScript, headless, programmable).
- **This fork:** https://github.com/az9713/flue-tutorial (public, default branch `main`).
- **Our additions are confined to `docs/`** — all upstream source/docs below the README banner are preserved unmodified.
- **GitHub Pages** is live: source `main` + `/docs` folder (`docs/.nojekyll` disables Jekyll). Interactive page → https://az9713.github.io/flue-tutorial/flue-demystified.html

What we added:
| Path | What |
| --- | --- |
| `docs/FLUE-DEMYSTIFIED.md` | The deep-dive write-up (the canonical explainer; ~330 lines) |
| `docs/flue-demystified.html` | Standalone interactive single-file page (vanilla JS, no build) |
| `docs/flue-demystified.mp4` | 34s / 1080p video explainer (curated copy) |
| `docs/video/` | HyperFrames project that renders the video (see §5) |
| `docs/CONTEXT.md` | This file |
| `README.md` banner | Front-and-center credit to upstream + links + inline video embed |

The video is also hosted at a GitHub user-attachments URL (in the README banner) and embedded inline on the repo homepage.

---

## 2. What Flue actually is (the 60-second model)

The whole point of the deep-dive: **the word "harness" is overloaded across three layers**, which is what makes Flue confusing.

1. **The LLM loop** — `pi-agent-core`'s `Agent` class (`@earendil-works/pi-agent-core`). read → call tool → observe → repeat. The real agentic engine. **Flue wraps it.**
2. **Flue's `Harness`** — the object `init(agent)` returns. A handle to sessions, fs, sandbox, tools. *Not* the loop.
3. **Flue, the framework** — supplies the harness half of **`agent = model + harness`**. "Astro/Next.js, but for agents."

The smoking gun: inside Flue's `Session`, `packages/runtime/src/session.ts:679` does `this.harness = new Agent({...})` importing `Agent` from pi-agent-core. So Flue stores the layer-1 loop in a field it *also* calls `harness`.

**Agents vs Workflows** (the other confusion):
- **Agent** = `agents/<name>.ts`, default-exports `createAgent()`. Addressable by URL (`POST /agents/<name>/<id>`), persistent, runtime builds the harness for you. For conversations.
- **Workflow** = `workflows/<name>.ts`, exports `run()`. Bounded job, you call `init(agent)` yourself, each run has a `runId` (`/runs`, `flue logs`). For triggers/CI.
- **Runs belong to workflows only** — never call an agent prompt a "run".

**Hierarchy:** `AgentInstance → Harness → Session → Operation → Turn` (one Operation = one `prompt()`/`skill()`/`task()`/`shell()`; one Turn = one LLM round-trip inside pi-agent-core).

**`prompt()` trace** (anchors in `packages/runtime/src/session.ts`): `prompt()` :839 → `runOperation` (exclusive lock) :1486 → `runModelTurnWithRecovery` :1666 (Flue's loop — **recovery/persistence only**, not the agent loop) → kicks pi-agent-core via `harness.prompt()`/`waitForIdle()` → Turns stream through `emitTurnRequestAndStream`/`streamSimple` :612 → tools run parallel in sandbox :690/:729 → `syncHarnessMessagesSince` :1632 + `save()` :1639 → returns `{text,usage,model}` :2136. Structured output (`result:` schema) = a synthetic `finish` tool + retry loop (`runWithResultTools` :2156, `MAX_FOLLOWUPS=32` :2165).

---

## 3. Repo layout (upstream)

Monorepo, pnpm workspaces, Turbo.

- `packages/runtime/` — **`@flue/runtime`**: harness, sessions, tools, sandbox, routing. Wraps pi-agent-core. Key file: `src/session.ts` (the engine wrapper), `src/harness.ts` (layer-2 Harness), `src/agent.ts`, `src/index.ts` (public exports).
- `packages/cli/` — **`@flue/cli`** (`flue` binary): Vite build graph, dev/build/run/connect/add, targets node + cloudflare.
- `packages/sdk/`, `packages/opentelemetry/`, `packages/connectors/`.
- `apps/docs/` — **Astro Starlight docs site** (the real Flue docs, ~64 `.md`/`.mdx` files under `src/content/docs/`: introduction, concepts, guide, api, cli, sdk, config, ecosystem, getting-started, reference). This is the source for flueframework.com. To read as a site: `cd apps/docs && pnpm install && pnpm dev`.
- `apps/www/` — marketing site.
- `examples/` — assistant, braintrust, chat-sdk, cloudflare(-websocket), hello-world, imported-skill, node-websocket, sentry.
- `connectors/` — markdown install instructions (sandbox providers: daytona, e2b, modal, vercel, cloudflare, etc.). Not npm packages.
- `AGENTS.md` (root) — **authoritative upstream terminology table + dev/test conventions**. Read it before non-trivial work.

Agent/workflow sources live in `<root>/.flue/` OR `<root>/` (when `.flue/` exists, bare `agents/`/`workflows/` is ignored). This repo's `.flue/` has `lib/github.ts` + `workflows/pr-redirect.ts`.

---

## 4. Dev workflow & commands

From upstream `AGENTS.md`:
- **Build order matters:** build runtime before CLI/examples. In `packages/runtime/`: `pnpm run build`; then in `packages/cli/`: `pnpm run build`. Type-check runtime: `pnpm run check:types`.
- Tests: active suite in `<package>/test/`, archived in `<package>/test-legacy/` (don't add there). Design tests from observable contracts; `describe('fn()')` / `it('X when Y')`. Don't over-test.
- Plans go in `plans/` (gitignored).
- `task` subagents must be told NOT to spawn their own subagents. Treat `review` feedback as input, not requirements.

Flue CLI (once built): `flue dev --target node|cloudflare` (port 3583), `flue build --target ...` (→ `./dist`), `flue run <workflow> --payload '...'`, `flue connect <agent> <id>`, `flue add <connector> | <agent>`.

Toolchain present on this machine: **Node v22**, pnpm/npm, **FFmpeg 7.1**, git 2.54, gh (authed). Models referenced in code use `provider/model` strings (e.g. `anthropic/claude-sonnet-4-6`).

---

## 5. The HyperFrames video pipeline (`docs/video/`)

**HyperFrames by HeyGen** ("write HTML, render video") — catalogued in the plugin marketplace but NOT installed as a plugin; driven entirely via `npx hyperframes@0.6.70 <cmd>`. Fully local, no API key.

Commands (from `docs/video/`): `npm run dev` (preview, long-running — background it), `npm run check` (lint+validate+inspect), `npm run render` (→ MP4 in `renders/`).

**Composition rules** (learned + from generated `docs/video/CLAUDE.md`):
1. Every timed element needs `class="clip"` + `data-start` + `data-duration` + `data-track-index`.
2. One **paused** GSAP timeline registered on `window.__timelines["main"]`; the renderer seeks it frame-by-frame.
3. **Deterministic only** — no `Date.now()`, `Math.random()`, or network logic.
4. **Fonts MUST be literal compiler-mapped names** (e.g. `Playfair Display`, `JetBrains Mono`, `Outfit`, `EB Garamond`, `Montserrat`, `Inter`...). The compiler does **static** analysis of `font-family` and **cannot resolve CSS `var()`** — using `var(--serif)` silently falls back to a generic font. List of mapped names appears in render logs.
5. Add `tl.set(target, {opacity:0}, boundary)` **hard-kills** at each scene-out boundary, or non-linear seeking leaves stale visibility.

**Hard-won lessons:**
- The `validate` (headless-Chrome) pass can report 0 errors while layout is genuinely broken. **`inspect` is stricter, and real pixels are truth.** Verify by extracting frames: `ffmpeg -ss <t> -i <mp4> -frames:v 1 out.png` and actually look at them. (S4 "Workflows" card overflow was a real bug `validate` missed; fixed with a CSS grid + `overflow:hidden`.)
- `renders/` is gitignored (timestamped output); the curated copy is committed at `docs/flue-demystified.mp4`.
- Current composition: 6 scenes / 34s / 1920×1080 / 30fps, palette near-black + ember `#ff6a3d` / teal `#57e0c4`, mirroring the HTML.

The interactive **HTML** (`docs/flue-demystified.html`) is intentionally a separate, dependency-free single file (Instrument Serif + JetBrains Mono + Spline Sans via Google Fonts; vanilla JS for the 3-layer decoder, equation, and the `prompt()` trace player). It doubles as HyperFrames' "website-to-video" capture input if ever needed.

---

## 6. Git / GitHub / Pages state

- Remote `origin` = https://github.com/az9713/flue-tutorial (public), default branch **`main`**.
- `gh` authed as **az9713** (a second account `ai-java-kid` also exists in `gh auth` — make sure `az9713` is active). git local config set to `az9713` / `az9713@yahoo.com`.
- Commit trailer used: `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`.
- **GitHub Pages:** enabled via `POST /repos/az9713/flue-tutorial/pages` with `{"source":{"branch":"main","path":"/docs"}}`. Served from `/docs`, `docs/.nojekyll` present. Live (HTTP 200): https://az9713.github.io/flue-tutorial/flue-demystified.html . After pushing doc changes, Pages rebuilds automatically (~1 min; poll `gh api repos/az9713/flue-tutorial/pages --jq .status` until `built`).
- `.gitignore` additions by us: `docs/video/renders/`.

---

## 7. Environment gotchas (Windows)

- Repo path: `C:\Users\simon\Downloads\astro_flue_better_stack\flue-main`. It **was double-nested** (`flue-main/flue-main/`); the user flattened it. The real project root is where `package.json`/`README.md`/`AGENTS.md` live.
- The **Bash tool's CWD can reset** to the outer dir between calls — use absolute paths or `cd` at the start of each command.
- Git prints harmless `LF will be replaced by CRLF` warnings on add/commit; ignore them.
- Prefer the dedicated Read/Edit/Write/Grep/Glob tools over shell `cat`/`sed`/etc.

---

## 8. User preferences (for how to work here)

- **Write non-trivial explanations to markdown files on disk, not the terminal.** Long traces, architecture explanations, deep dives → a `.md` artifact; the terminal gets only a short pointer + key takeaway. (This is why the deep-dive lives in `docs/` rather than chat.)
- The user is exploring/understanding this codebase and values durable, reusable artifacts and honest flagging of what was verified vs. assumed.

---

## 9. Open threads / possible next tasks

- README MP4 link uses the hosted `user-attachments` URL (set) + inline embed + in-repo copy. Done.
- Pages root (`https://az9713.github.io/flue-tutorial/`) has **no `docs/index.html`** → bare root may 404. A landing `docs/index.html` (or redirect to `flue-demystified.html`) would polish it.
- Possible video enhancements: burned-in captions, a voiceover track (HyperFrames supports local Kokoro TTS via `/hyperframes-media`).
- **Upstream sync strategy** not yet defined: there's no `upstream` remote. To pull future Flue changes, add `git remote add upstream https://github.com/withastro/flue` and merge/rebase, keeping `docs/` + the README banner.
- Building actual Flue agents/workflows for the tutorial (none added yet beyond upstream examples).
