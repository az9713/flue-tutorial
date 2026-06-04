# Flue, Demystified

> A standalone explainer written for someone reading the `flue-main` download to
> understand what it *is*. It does not replace the project's own docs site (that
> lives under `flue-main/apps/docs/` and is excellent); it answers the one
> question that site assumes you already understand: **what is this thing, and is
> it a harness itself?**

---

## The one-sentence answer

**Yes — Flue is a harness. More precisely, it is a _framework for building agent
harnesses_: the "Astro/Next.js, but for agents" of its own tagline.** It gives a
language model the surrounding machinery — sessions, tools, skills, a filesystem,
a sandbox, persistence — that turns "a model you call over an API" into "an agent
that does work." The actual model-calling loop lives one layer deeper, in a
dependency called `pi-agent-core`, which Flue wraps.

If that already makes sense, you're done. The rest of this document exists
because the word **"harness"** is used for three different things in and around
Flue, and that overload is the entire reason the project feels mysterious.

---

## Why it feels mysterious: "harness" means three things

Read the README and you hit the word *harness* constantly — "The Agent Harness
Framework," "your initialized harness," `const harness = await init(agent)`. It
refers to three distinct layers, and nobody tells you they're different.

| # | What "harness" refers to | Where it lives | What it actually is |
|---|--------------------------|----------------|---------------------|
| 1 | **The LLM loop** | `pi-agent-core`'s `Agent` class | The read → call-a-tool → observe → repeat loop. The literal agentic engine. |
| 2 | **Flue's `Harness`** | `@flue/runtime`, returned by `init(agent)` | The programmable boundary: a handle to sessions, filesystem, sandbox, tools, model defaults. |
| 3 | **Flue, the framework** | The whole project | "The Agent Harness Framework." Markets itself as supplying the harness half of `agent = model + harness`. |

Here is the part that makes it concrete. Inside Flue's own `Session` class, when
it starts the engine, it writes:

```ts
// packages/runtime/src/session.ts:679
this.harness = new Agent({ ... }); // `Agent` is imported from @earendil-works/pi-agent-core
```

So Flue takes `pi-agent-core`'s `Agent` (layer 1, the real loop) and stores it in
a private field it *also* calls `harness`. Meanwhile the thing **you** get back
from `init(agent)` is a different object — Flue's own `Harness` class (layer 2) —
which holds sessions and a sandbox and creates those `Agent` loops on demand.

Once you see that the same word is doing three jobs, the mystery mostly
evaporates. The three are nested, not competing:

```
Flue the framework (layer 3)
  └─ gives you a Harness handle (layer 2)         ← what `init(agent)` returns
       └─ which opens Sessions
            └─ each Session runs a pi-agent-core Agent loop (layer 1)
```

---

## The mental model: `agent = model + harness`

Flue's design rests on one equation, stated plainly in its philosophy doc:

> **A model alone is not an agent.** A model is "a brain in a jar" — great at
> reasoning, but with no memory once a response ends, no hands to run code, no
> senses to read a file. The **harness** is everything you bolt on to fix that.

Each piece of the harness exists to remove one of those limitations:

- Want the agent to **keep its work**? Give it a **filesystem**.
- Want it to **act, not just describe**? Give it **tools**.
- Want it to act **safely**? Run those tools in a **sandbox**.
- Want it to **stay sharp over long tasks**? Manage its **context** (instructions, skills).
- Want it to **do several things at once**? Let it spawn **subagents**.

Flue's pitch is: *all five of those, in TypeScript, headless, runnable anywhere.*
You don't script the agent's steps. You fill the harness with context and tools,
point a model at it, and tell it to go.

### "It's like Claude Code, but…"

The README leans on this comparison, so it's worth making the difference exact:

| | Claude Code / Codex | Flue |
|---|---|---|
| Interface | A TUI / CLI for a human operator | **Headless** — no UI of its own |
| Who drives it | A person typing | **Your code** — HTTP, WebSocket, `flue run`, CI |
| Where logic lives | Largely in the tool | **In your TypeScript + Markdown** (`AGENTS.md`, skills) |
| Assumed operator | A human in the loop | **None** — built for autonomous/background work |

Same *architecture* (a model in a harness with sessions, tools, skills). Flue
just strips out the human-facing shell and exposes the harness as a programmable
API you build products on top of.

---

## What you actually write: Agents vs. Workflows

The second thing that makes Flue confusing is that it has **two** top-level
authoring surfaces, and they look similar but mean different things. This
distinction comes straight from the project's `AGENTS.md` terminology table.

### Agents — addressable and persistent

An **agent** is a file at `agents/<name>.ts` (or `.flue/agents/<name>.ts`) that
default-exports a `createAgent(...)`. It is **addressable**: callers reach a
specific *instance* of it by URL.

```txt
POST /agents/<agent-name>/<id>          ← send a message
GET  /agents/<agent-name>/<id>          ← (Upgrade: websocket) open a live session
```

The `<id>` segment is the **agent instance** — the durable scope for one
customer, one repo, one conversation. Reuse the same `<id>` to continue where you
left off (on Cloudflare, backed by Durable Objects; on Node, in memory by
default). Use a new `<id>` to start fresh. Agents are for **ongoing, stateful**
things: a support thread, a chat assistant, a long-lived coding session.

### Workflows — bounded jobs with run history

A **workflow** is a file at `workflows/<name>.ts` that exports a `run(...)`
function. It is **not** addressable and **not** persistent in the same way — it's
a job: clear input → finished result. Each invocation is a **run** with a unique
`runId`, and runs are inspectable (`/runs`, `flue logs`).

```ts
// workflows/hello.ts — a workflow exports run()
export async function run({ init, payload }: FlueContext) {
  const harness = await init(translator); // YOU create the harness here
  const session = await harness.session();
  const { data } = await session.prompt(`Translate: ${payload.text}`, { result: schema });
  return data;
}
```

The critical asymmetry, and a frequent point of confusion:

- In a **workflow**, *you* call `init(agent)` to build the harness yourself.
- In an **agent module**, the **runtime** builds the harness for you automatically
  when a message arrives at the agent's URL.

> **Rule of thumb:** reach for a **workflow** for a finite task you trigger
> (CI job, one-shot script, a `/runs`-tracked batch). Reach for an **agent** when
> something or someone needs to hold a *conversation* with a persistent instance
> over time. Runs belong to workflows only — never describe an agent prompt as a
> "run."

---

## The full hierarchy (what nests inside what)

Flue's own `AGENTS.md` defines this spine. Internalize it and the API stops
feeling arbitrary — every method name maps to a level here.

```
Agent profile        — one reusable defineAgentProfile(...) value
Created agent        — one runtime initializer from createAgent(...)
Agent module         — agents/<name>.ts; default-exports a created agent
└─ AgentInstance     — the URL <id>; one durable caller-defined boundary
   └─ Harness        — init(agent) → the programmable handle (defaults to name "default")
      └─ Session     — harness.session(name?) → one conversation (defaults to "default")
         └─ Operation — one prompt() / skill() / task() / shell() call
            └─ Turn   — one LLM round-trip inside pi-agent-core (layer 1!)
```

Mapping the words to the layers from the table at the top:

- **Harness** here is layer 2 — the thing `init()` returns.
- **Turn** here is layer 1 — a single round-trip of the `pi-agent-core` loop.
- An **Operation** (`prompt`, `skill`, `task`, `shell`) is the unit *you* invoke;
  it may take several Turns internally as the model calls tools and observes
  results.

```ts
// Putting it together — this is the whole API surface in five lines:
const agent   = createAgent(() => ({ model: 'anthropic/claude-sonnet-4-6', tools, skills }));
const harness = await init(agent);          // layer 2: the harness handle
const session = await harness.session();     // a conversation inside it
const { text } = await session.prompt('Triage this issue.'); // an Operation → N Turns
```

---

## The sandbox: why "no container" is a headline feature

A harness can't *do* anything beyond emit text unless the agent has somewhere to
act. That somewhere is the **sandbox**: the filesystem + command-execution
boundary. The built-in `read`, `glob`, `grep`, and `bash` tools all operate
inside it.

Flue's notable default: every agent gets a **virtual sandbox** — an in-memory
filesystem and shell powered by [`just-bash`](https://github.com/vercel-labs/just-bash) —
with zero setup. No Docker, no VM. That's why the README keeps saying agents run
"without a container": for many agents you genuinely don't need one, which makes
them dramatically cheaper and more scalable to run at high volume.

When you need a real Linux box (a true coding agent with `git`, `npm`, a
browser), you swap the sandbox behind the same API:

- `local()` — direct access to the host machine (great for CI runners, where the
  runner itself is your isolation boundary).
- A **connector** — Daytona, E2B, Modal, Vercel, Cloudflare, etc. Connectors
  aren't npm packages; they're Markdown install instructions your coding agent
  applies to your project (`flue add daytona | claude`).

Same `session.fs` / `session.shell` API regardless of which one is mounted. That
substitutability is the point.

---

## The two-package architecture

Flue ships as a monorepo, but the runtime story is two packages:

| Package | Role |
|---------|------|
| `@flue/runtime` | The harness itself: sessions, tools, sandbox, persistence, routing. Wraps `pi-agent-core`. This is what your agent code imports. |
| `@flue/cli` | The `flue` binary + build tooling: a Vite build graph that compiles your `agents/` and `workflows/` into a deployable server artifact, plus `flue dev / run / connect / build / add`. |

The deeper dependency that does the literal LLM loop —
`@earendil-works/pi-agent-core` — is **not** something you import or think about.
Flue wraps it completely. (In this download it's pinned as `"*"` and may not even
be installed; you don't need it to read the source and understand the boundary.)

The lifecycle, end to end:

```
You author        →  agents/*.ts, workflows/*.ts, AGENTS.md, .agents/skills/*
flue build        →  Vite compiles it into ./dist (a Node .mjs, or a CF Worker)
You deploy/run    →  flue run (CI) · HTTP/WebSocket server · Cloudflare · GH Actions
At runtime        →  init(agent) → Harness → Session → Operation → pi-agent-core Turns
```

"Write once, deploy anywhere" is literal: the same agent source targets Node,
Cloudflare, GitHub Actions, GitLab CI — you change the `--target`, not the code.

---

## So, is Flue a harness itself? — the precise answer

- **As a product/framework:** yes. Flue *is* "the harness" in `agent = model +
  harness`. It supplies every piece of the harness (sessions, tools, skills, fs,
  sandbox) and nothing of the model — you bring the model by name
  (`anthropic/claude-sonnet-4-6`, `openai/gpt-5.5`, …).
- **As an object in the API:** the thing literally *called* a harness is the
  `Harness` you get from `init(agent)` — layer 2. It's a handle, not the loop.
- **The actual agentic loop** (the part most people picture when they hear
  "harness") is one layer below Flue, in `pi-agent-core`. Flue's contribution is
  everything *around* that loop that makes it deployable, persistent, sandboxed,
  and programmable without a human in the loop.

Put differently: **Flue is to an agent harness what Next.js is to a web server.**
Next.js doesn't reinvent HTTP; it gives you the framework, conventions, and build
pipeline around it. Flue doesn't reinvent the LLM loop; it gives you the
framework, conventions, and build pipeline around *that*.

---

## Is it a compiler, then?

A natural follow-up, since `flue build` clearly *compiles* something. The honest
answer: **"harness compiler" is a useful half-truth — right about one real step,
misleading as a label for the whole.**

**Where it's genuinely right.** There is a real compile step. `flue build` (in
`@flue/cli`) runs a **Vite build graph** that takes your authored sources —
`agents/*.ts`, `workflows/*.ts`, `AGENTS.md`, `.agents/skills/*` — and emits a
deployable artifact in `./dist` (a single Node `.mjs`, or a Cloudflare Worker +
Durable Objects). It does compiler-ish work along the way: validating your static
skill imports, packaging each `SKILL.md` and its permitted supporting files,
rejecting files that look like secrets, and bundling everything for the chosen
`--target`. So "Flue compiles your project" is simply a true sentence.

**Where it misleads.** The phrase mislabels the *whole* framework, because the
harness is neither the input nor the output of that compile:

- **The thing compiled** is your *project* (agents + workflows + Markdown
  context) — not a harness.
- **The thing emitted** is a *server* — not a harness. The harness is built
  later, at **runtime**, by `init(agent)` → `Harness` → `Session`. The compiler
  emits something that *manufactures* harnesses on demand; it never emits a
  harness.
- **Most of Flue's value lives at runtime**, not in the build: sessions, message
  persistence, the sandbox, routing, the `pi-agent-core` loop. "Compiler" quietly
  drops all of that.

So the strict reading is: `flue build` compiles *your agents* into a
*harness-serving server*. Neither end of that is a harness — which is why
"harness compiler" doesn't quite land.

**The sharper label.** Flue is a **harness framework with a compiler in it** —
exactly the Astro/Next.js shape. Nobody calls Next.js a "page compiler," even
though `next build` absolutely compiles pages into a server; the compiler is one
part of a framework that is mostly runtime. Same here. If you want a one-liner
that keeps the compiler insight without overclaiming:

> **Flue compiles declarative agent definitions (TypeScript config + Markdown
> context) into a deployable server that constructs harnesses at runtime.**

That credits the build step *and* names what actually comes out.

---

## Where to go next (in this download)

The project's real docs are thorough — now that the vocabulary is unlocked,
they'll read cleanly:

- `flue-main/README.md` — the worked examples (support agent, CI triage, coding agent, MCP).
- `flue-main/AGENTS.md` — the authoritative terminology table (source for the hierarchy above).
- `flue-main/apps/docs/src/content/docs/introduction/philosophy.md` — `agent = model + harness`, stated by the authors.
- `flue-main/apps/docs/src/content/docs/concepts/agents.mdx` — the canonical "what is an agent / what is a harness" deep dive.
- `flue-main/apps/docs/src/content/docs/guide/` — per-piece guides: workflows, sandboxes, tools, skills, subagents.
- `flue-main/packages/runtime/src/harness.ts` and `session.ts` — read these two to *see* layers 2 and 1 meet (the `new Agent(...)` line is `session.ts:679`).

---

## Deep dive: one `prompt()`, turn-by-turn

This is the conceptual layering from earlier, *in motion*. It traces exactly what
happens when you call `session.prompt(...)`, with `file:line` anchors into
`packages/runtime/src/session.ts`.

### Hold onto this: "turn" means two nested things

- **Operation** — your one `session.prompt()` call. Flue's code.
- **Turn** — one LLM round-trip (a provider HTTP call) inside `pi-agent-core`.

**One Operation drives 1…N Turns.** A `prompt()` where the model runs three bash
commands is ~4 Turns (3 tool-calling round-trips + 1 final answer), all inside a
single Operation. The thing most people picture as "the agent loop" is the inner
Turn loop, and it lives one layer below Flue, in `pi-agent-core`.

### A. Setup — before any model call

1. **`prompt(text, options)`** → `createCallHandle(signal, …)` wraps the work in
   an abortable handle (`:839`). Your `signal` / `.abort()` plugs in here.
2. **`runOperation('prompt', …)`** (`:1486`) takes the session's **exclusive
   lock** — only one prompt/skill/task/shell runs at a time per session — and
   wires cancellation to `harness.abort()` (`:1502`).
3. **`buildPromptText(text, schema)`** (`:843`) — if you passed a `result:`
   schema, the prompt is augmented to instruct the model to return data via a
   `finish` tool.
4. **`runPromptCall(…)`** (`:2085`): with a schema, `createResultTools(schema)`
   mints two synthetic tools — **`finish`** (success; its params *are* your
   valibot schema) and **`give_up`** (typed failure).
5. **`withCallOverrides(…)`** (`:1258`) snapshots harness state, then for *this
   call only* resolves the model + thinking level and assembles the toolset:
   `[...builtin (read/glob/grep/bash/task), ...your defineTool tools, ...result
   tools]`, assigning it to `harness.state.tools`. A `finally` restores the prior
   state afterward — which is why per-call overrides never leak.

Throughout, `harness` is the **pi-agent-core `Agent`** (layer 1), constructed at
`:679`.

### B. The turn loop — `runModelTurnWithRecovery` (`:1666`)

```text
while (true):
  beforeLength = harness.state.messages.length
  await start()                  // → harness.prompt(text, images)  ← kicks pi-agent-core
  await harness.waitForIdle()    // engine runs ALL its internal Turns until done
  await syncHarnessMessagesSince(beforeLength, source)   // append to history + persist
  latest = last message
  if not assistant            → return
  if context overflow         → compact, then start = harness.continue(); retry
  if retryable model error    → backoff, then start = harness.continue(); retry
  else checkCompaction();       return
```

The key realization: this outer loop is **recovery only** — context-overflow →
compaction, and transient-error backoff. The *normal* read → tool → observe
iteration is **not here**. It happens *inside* `harness.prompt()` /
`waitForIdle()`, down in pi-agent-core.

### C. Inside one `harness.prompt()` — the Turns

Observed via the subscribe handler (`:695`) and `emitTurnRequestAndStream`
(`:612`). For each Turn the engine runs:

1. `turn_start` → Flue assigns an `activeTurnId` and emits `turn_start` (`:700`).
2. **`emitTurnRequestAndStream`** (the `streamFn`, `:612`) emits a `turn_request`
   event carrying the *full* input (systemPrompt, messages, tools), then calls
   **`streamSimple(model, context, options)`** — **this is the literal provider
   HTTP streaming call** (pi-ai).
3. Streaming deltas arrive as `message_update`, re-emitted as `text_delta` and
   `thinking_start/delta/end` (`:708`).
4. If the assistant message contains **tool calls**, the engine (configured
   `toolExecution: 'parallel'`, `:690`) fires `tool_execution_start` and runs each
   tool's `execute(id, params, signal)` — e.g. `bash` hits the sandbox via
   `env.exec`. Results are appended to the message list.
5. The engine **loops back to step 1** with tool results in context — another
   Turn, another `streamSimple` call — until the model emits a final assistant
   message with **no tool calls**. Then it goes idle and `waitForIdle()` resolves.

### D. Persistence & return

- After the engine idles, **`syncHarnessMessagesSince`** (`:1632`) slices the new
  messages, appends them to `SessionHistory`, and calls **`save()`** (`:1639`),
  writing to the `SessionStore` — in-memory on Node by default, Durable Object
  storage on Cloudflare. **Persistence happens once per idle cycle, not per
  Turn.**
- **No-schema path:** returns
  `{ text: getAssistantText(), usage: aggregateUsageSince(beforeLeafId), model }`
  (`:2136`).
- **Schema path:** `runWithResultTools` (`:2156`) inspects the outcome —
  `finished` → returns validated `data`; `gave_up` → throws
  `ResultUnavailableError`; `pending` (model replied but never called `finish`) →
  sends a nudge follow-up and loops, up to **`MAX_FOLLOWUPS = 32`** (`:2165`).

### Two insights worth keeping

- **Structured output is forced tool-calling, not JSON mode.** Your `result:`
  schema becomes the parameters of a synthetic `finish` tool, plus a retry loop
  that nudges the model until it calls it.
- **Flue's loop ≠ the agent loop.** `runModelTurnWithRecovery` is a thin
  recovery/persistence wrapper; the real read-act-observe cycle is
  pi-agent-core's. This trace *is* the layer-1 / layer-2 boundary from the top of
  this document, shown in motion.
