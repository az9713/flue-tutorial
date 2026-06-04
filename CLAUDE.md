# CLAUDE.md

This repository is a **tutorial clone / enhanced fork of [`withastro/flue`](https://github.com/withastro/flue)**, published at https://github.com/az9713/flue-tutorial. All original Flue source is preserved; our additions live in `docs/`.

## Start here

**Read [`docs/CONTEXT.md`](docs/CONTEXT.md) first** — it is the durable working-memory file for this project: what the fork is, where everything lives, how Flue's architecture actually works, the dev/build/test workflow, the HyperFrames video pipeline, GitHub/Pages state, environment gotchas, and open threads. It is written to let a fresh session resume development with no prior context.

@docs/CONTEXT.md

## Also reference

- **[`AGENTS.md`](AGENTS.md)** — upstream's authoritative terminology table and dev/test conventions. Defer to it for how to build, test, and structure changes in `packages/`.
- **[`docs/FLUE-DEMYSTIFIED.md`](docs/FLUE-DEMYSTIFIED.md)** — the architecture deep-dive (the three meanings of "harness", `agent = model + harness`, agents vs. workflows, and a turn-by-turn trace of `prompt()` with `file:line` anchors).

## House rules (project-specific)

- **Write non-trivial explanations to markdown on disk, not the terminal** — long traces, architecture explanations, and deep dives go in a `.md` artifact (usually under `docs/`); the terminal gets a short pointer + key takeaway.
- **Keep upstream source unmodified** unless a task explicitly targets it; put new tutorial material in `docs/`.
- **Verify rendered output with real pixels** (extract frames with `ffmpeg`) — the HyperFrames `validate` pass can miss layout bugs that `inspect` and actual frames catch. See `docs/CONTEXT.md` §5.
- On commits, use the trailer `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`. After pushing `docs/` changes, GitHub Pages rebuilds automatically.
