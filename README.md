# ʊ-Net    HydraNet

> A local-first, multi-agent coding assistant that stops AI agents from looping on the same broken fix. Runs entirely on your own GPUs — no cloud calls, no token bills, no black box.

Built by **M. Yadav**, **Y. Jangra**, and **B. Kataria** — three devs, three laptops, six months.

License: MIT &nbsp;·&nbsp; Status: Pre-alpha / building in public &nbsp;·&nbsp; Stack: Python + TypeScript

---

## What this actually is

Most AI coding agents — cloud or local — share one failure mode. When a fix doesn't work, the agent tweaks one line and retries the *exact same broken approach*, over and over, until it burns through its budget. A human would stop, think, and try something genuinely different.

Loopcutter exists to fix that one thing well: detect when an agent is stuck, halt it, roll the workspace back to a known-good state, and force a real change in approach — instead of letting it spiral for 20 attempts.

Everything else here — the AST-aware code search, the three-laptop setup, the dashboard — exists to support that core loop. This is **not** trying to be a free, smaller Devin. Read [Honest Scope](#honest-scope--read-this-before-you-commit-six-months) before committing six months to it.

---

## How it works

```
                     [ Issue / bug description comes in ]
                                     │
                                     ▼
        ┌─────────────────────────────────────────────────────────┐
        │         NODE A — SUPERVISOR (M. Yadav · RTX 3050, 6GB)   │
        │     state machine · loop detector · git rollback engine  │
        └───────────────┬───────────────────────┬──────────────────┘
                         │                       │
               (plan / write code)      (run tests / sandbox)
                         │                       │
                         ▼                       ▼
        ┌────────────────────────────┐  ┌─────────────────────────────┐
        │  NODE B — INFERENCE WORKER │  │  NODE C — SANDBOX WORKER     │
        │  (Y. Jangra · RTX 4060,8GB)│  │  (B. Kataria · RTX 4060,8GB) │
        │  quantized coder model      │  │  Docker test execution       │
        │  AST-aware code retrieval   │  │  checkpoint / diff tooling   │
        └────────────────────────────┘  └─────────────────────────────┘
```

Each laptop runs its **own** model and its **own** job, and they coordinate over your local network. This is a distributed multi-agent system — not one model split across three machines. See [Honest Scope](#honest-scope--read-this-before-you-commit-six-months) for why that distinction is worth being precise about.

### The loop-breaking flow — the actual point of this project

```
attempt 1 fails ──► attempt 2 (near-identical) fails ──► attempt 3 (still near-identical) fails
                                                                       │
                                                                       ▼
                                               [ SUPERVISOR INTERVENES ]
                                     git checkpoint → rollback → forced re-plan
                                        (failure history injected into prompt)
                                                                       │
                                                                       ▼
                                                attempt 4: genuinely different approach
```

### AST-aware retrieval, in one picture

```
[auth.ts] ──(imports)──► [db.ts] ──(mutates)──► [users schema]
    │                                                  ▲
    └──────────(referenced by the error log)───────────┘
```
A flat embedding search matches on wording and would likely miss `db.ts` entirely. Walking the actual import/call graph doesn't.

---

## Team

| Person | Role | Owns | Machine |
|---|---|---|---|
| **M. Yadav** | Project Lead / Systems Architect | Orchestration, state machine, networking, rollback engine | Node A — RTX 3050, 6GB |
| **Y. Jangra** | Core Contributor — Inference Lead | Model serving, prompting, AST parsing, code retrieval | Node B — RTX 4060, 8GB |
| **B. Kataria** | Core Contributor — Infra Lead | Sandboxing, Docker, checkpoint/rollback tooling, dashboard plumbing | Node C — RTX 4060, 8GB |

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Orchestration | Python + LangGraph | Cyclic graphs map naturally onto retry/backtrack logic |
| Model serving | Ollama or vLLM | Quantized 7–8B coder models (Qwen2.5-Coder, DeepSeek-Coder) — what actually fits in 6–8GB |
| Networking | gRPC or WebSocket + TLS | Use existing, audited TLS. Don't hand-roll crypto handshakes |
| Sandboxing | Docker | Ephemeral, per-attempt containers; only the relevant repo slice gets mounted |
| Code parsing | Tree-sitter | Real AST extraction — not regex, not flat embeddings |
| Vector store | pgvector (Postgres) | One database, not two |
| Dashboard | Next.js + WebSockets | Live state stream, deliberately minimal in v1 |
| Version control | git (GitPython / CLI) | Branch-per-attempt, checkpoint, rollback |

## Six-month roadmap

| Month | Objective | What "done" looks like | Primary owner |
|---|---|---|---|
| 1 | Single-machine agent loop | CLI fixes a small Python bug end-to-end inside Docker, on one laptop | All three — shared foundation |
| 2 | Loop detection & backtracking | Demo: agent backtracks after 3 failed attempts instead of looping 20+ | M. Yadav |
| 3 | AST-aware code RAG | Side-by-side proof structural search finds a bug flat embeddings miss | Y. Jangra |
| 4 | Multi-machine coordination | The Month-1 CLI command now runs across all 3 physical laptops | M. Yadav |
| 5 | Minimal live dashboard | Screen-recordable UI showing live agent state | Y. Jangra |
| 6 | Benchmark + honest release | Public repo with real SWE-bench Lite numbers, failures included | B. Kataria |

## Honest scope — read this before you commit six months

| The pitch | The reality | Why it matters |
|---|---|---|
| "Pools VRAM across 3 laptops into one bigger model" | Three independent models that coordinate over the network | True tensor-parallel pooling across machines exists (exo, llama.cpp's `rpc-server`), but WiFi latency makes it slower than running models independently. A coordinated multi-agent mesh is the right call — just don't market it as something it isn't. |
| "Beats Devin / OpenHands / Cline in 6 months" | "Does one narrow thing — loop-breaking — better than they do, on hardware you already own" | Those tools have years of work, funded teams, and tens of thousands of GitHub stars behind them already. Competing head-on isn't a fair fight. Owning one edge is. |
| "10k–50k GitHub stars" | No star target | Stars follow usefulness, timing, and luck — not README copy. Treat them as a side effect, not a goal. |
| Custom RSA handshake + custom TCP framing, built from scratch | gRPC / WebSocket + TLS + a shared token | Don't spend a month reinventing security-critical networking primitives that are already audited and one `pip install` away. |
| "Boss" and "colleagues who bow to the host machine" | One architecture owner, three core contributors, shared credit | Fine for a README to show clear ownership. Not how to run an unpaid six-month project with two friends. |

---

## Getting started *(target interface — build this across Month 1 / Month 4)*

```bash
git clone <repo-url>
cd loopcutter

# Node A — orchestrator (Mohit's machine)
docker compose up -d                        # postgres + pgvector
python -m orchestrator.cli serve --port 8080

# Node B / Node C — worker machines
python -m worker.cli connect --host <orchestrator-ip>:8080 --role inference
python -m worker.cli connect --host <orchestrator-ip>:8080 --role sandbox

# run it
python -m orchestrator.cli fix --repo ./target_repo --issue "auth fails on logout"
```

---

## Month-by-month task list

Checkboxes below track progress. Heads up: GitHub only makes `- [ ]` boxes *clickable* inside Issues, PRs, and Discussions — in a plain repo file like this one they render but aren't toggleable by clicking. Either edit `- [ ]` → `- [x]` and commit, or mirror this list into a GitHub Project board / Issues if you want literal click-to-check.

### Month 1 — Single-machine agent loop

**M. Yadav — Orchestration**
- [ ] Scaffold the repo (package layout, linting/formatting config)
- [ ] Build the core agent state machine in LangGraph: read → plan → edit → test → reflect
- [ ] Build the CLI entrypoint (`loopcutter fix --repo --issue`)
- [ ] Add structured logging / run tracing so every step is inspectable later
- [ ] Write the config system (model choice, paths, retry limits) as one YAML/JSON file
- [ ] Get one real bug, in one real small repo, fixed end-to-end — this is the Month-1 finish line

**Y. Jangra — Inference**
- [ ] Set up Ollama or vLLM serving a quantized 7–8B coder model on a 4060
- [ ] Benchmark tokens/sec and usable context length at 4-bit on 8GB VRAM
- [ ] Build an inference client wrapper with timeouts, retries, streaming
- [ ] Write the first prompt templates: plan, edit, test-result-interpretation
- [ ] Add structured-output parsing so responses come back as reliable JSON, not prose
- [ ] Document exact VRAM usage at different context lengths — this becomes your scoping data later

**B. Kataria — Sandboxing**
- [ ] Write a base Dockerfile for a Python sandbox (pinned versions, no host leakage)
- [ ] Build host-to-container file sync for the working repo
- [ ] Add a command-safety guard (block `rm -rf`, `chown`, stray network calls, etc.)
- [ ] Capture stdout/stderr from the sandbox cleanly and separately
- [ ] Build container lifecycle management (spin up, kill, clean up on crash)
- [ ] Wire `pytest` execution inside the sandbox with a clear pass/fail signal back out

### Month 2 — Loop detection & backtracking *(the signature feature)*

**M. Yadav**
- [ ] Build a repetition detector comparing shell-command + diff history across attempts
- [ ] Define and tune the "stuck" threshold (start at 3 near-identical attempts, make it configurable)
- [ ] Build the intercept hook that halts execution once the threshold is hit
- [ ] Build a git-based checkpoint/rollback engine (branch-per-attempt, not raw `--hard` resets)
- [ ] Write the policy logic: retries-before-rollback, what gets logged when it fires
- [ ] Build a test harness that simulates a "death loop" so you can test the breaker on demand

**Y. Jangra**
- [ ] Design the self-reflection prompt: model states *why* the last attempt failed before retrying
- [ ] Inject failure history into the next prompt ("you tried X, it failed because Y, try differently")
- [ ] Make reflection + plan output reliable structured JSON
- [ ] Parse stack traces for the actual file/line instead of feeding the model a raw wall of text
- [ ] Run a controlled comparison: loop-breaking agent vs. baseline, on the same 5 seeded bugs
- [ ] Write up the numbers — attempts-to-resolution, not vibes

**B. Kataria**
- [ ] Build cheap workspace snapshotting (don't re-clone the whole repo every checkpoint)
- [ ] Build the diff calculator showing exactly what changed between attempts
- [ ] Implement branch-per-attempt so failed tries don't pollute the main working branch
- [ ] Add automated cleanup of dead branches and orphaned containers
- [ ] Wire test results back into the loop-detector as a structured pass/fail/error signal
- [ ] Record the demo: baseline agent looping forever vs. yours backtracking in 3 attempts

### Month 3 — AST-aware code RAG

**M. Yadav**
- [ ] Set up pgvector on the Node A Postgres instance
- [ ] Design the schema: code chunks, function/class nodes, import edges
- [ ] Build incremental re-indexing on file change (no full re-embed per run)
- [ ] Add query-time context-compression so retrieved code fits the model's context window
- [ ] Build a retrieval API the agent calls, instead of dumping raw results into the prompt
- [ ] Load-test indexing on a real medium-sized open-source repo, not a toy example

**Y. Jangra**
- [ ] Wire up Tree-sitter parsing for Python first — expand languages later, not now
- [ ] Extract functions, classes, imports, and call relationships into a graph
- [ ] Chunk code by logical unit (function/class), not arbitrary line counts
- [ ] Build the import/call graph that lets retrieval "hop" to related files
- [ ] Run the same seeded bug set through flat-embedding search vs. AST-aware search
- [ ] Write up the recall comparison — your strongest, most concrete README evidence

**B. Kataria**
- [ ] Build hybrid retrieval: vector similarity + keyword match + graph-neighbor expansion
- [ ] Add filters to skip `node_modules`, `venv`, build artifacts, lockfiles
- [ ] Build the context assembler turning retrieved chunks into a clean prompt block
- [ ] Add a small eval harness measuring retrieval precision/recall on the seeded bug set
- [ ] Cache high-value retrieval results to avoid re-querying on every retry
- [ ] Document what AST-aware retrieval still misses (dynamic imports, cross-language refs)

### Month 4 — Multi-machine coordination

**M. Yadav**
- [ ] Stand up the supervisor process on Node A, routing tasks to Node B and Node C
- [ ] Use gRPC or an authenticated WebSocket (TLS + shared token) — not custom crypto
- [ ] Add a heartbeat/health-check so a dropped worker doesn't silently stall a run
- [ ] Build a simple task queue so the supervisor can hold work if a worker is busy
- [ ] Explicitly handle "worker disconnects mid-task" — decide and implement the recovery path
- [ ] Get the exact Month-1 CLI command running across all three physical laptops, end to end

**Y. Jangra**
- [ ] Wrap the inference engine as a network-callable service on Node B
- [ ] Stream model output back to the supervisor instead of waiting for full completion
- [ ] Handle "supervisor unreachable" gracefully on the worker side
- [ ] Add request queuing so Node B can't be overwhelmed by parallel requests
- [ ] Measure round-trip latency added by the network hop vs. running locally — this number matters
- [ ] Tune what gets sent over the wire (full repo vs. relevant slice) to keep latency sane

**B. Kataria**
- [ ] Wrap the Docker sandbox as a network-callable service on Node C
- [ ] Build secure-enough transfer of only the relevant repo slice to the worker
- [ ] Handle sandbox cleanup if the network connection drops mid-test
- [ ] Add basic auth so a random device on the WiFi can't submit jobs to your sandbox
- [ ] Test what happens when Node C handles two things at once — document the limit honestly
- [ ] Screen-record the three-laptop run for the eventual launch video

### Month 5 — Minimal live dashboard

**M. Yadav**
- [ ] Build a WebSocket event stream from the supervisor to a frontend
- [ ] Add basic auth so the dashboard isn't open to anyone on the local network
- [ ] Stream state transitions (thought → action → observation → reflection) as structured events
- [ ] Add a pause/abort endpoint that actually halts execution, not just the UI
- [ ] Log every run to disk (SQLite is fine) so past sessions are reviewable
- [ ] Load-test the WebSocket connection across a long-running multi-hour session

**Y. Jangra**
- [ ] Scaffold the Next.js app with a minimal, clean layout (function over decoration)
- [ ] Build the live log/state stream component
- [ ] Build a basic diff viewer for the current edit
- [ ] Add a hardware panel showing VRAM/CPU usage per node
- [ ] Wire the pause/abort button to the backend endpoint
- [ ] Make sure it doesn't fall over if two people open it at once

**B. Kataria**
- [ ] Build session history storage (past runs, searchable)
- [ ] Add session export (download a run's full log as a file)
- [ ] Add input validation on anything the dashboard sends back to the backend
- [ ] Build a simple replay view for past runs
- [ ] Test the dashboard against a deliberately flaky WiFi connection — fix what breaks
- [ ] Production-build it and confirm it loads on a machine that isn't yours

### Month 6 — Benchmark, honest docs, ship it

**M. Yadav**
- [ ] Run a 24–48 hour stability soak test across all three nodes
- [ ] Write a setup doc and test it on a machine you've never touched before
- [ ] Pin dependency versions so a fresh clone doesn't break in a month
- [ ] Write basic CI: lint + a couple of integration tests on every PR
- [ ] Do a security pass on the sandbox boundary specifically — this is the part that can hurt someone if it's wrong
- [ ] Final repo cleanup: strip secrets, dead code, draft files

**Y. Jangra**
- [ ] Run Loopcutter against a real slice of SWE-bench Lite (even 20–30 issues is meaningful)
- [ ] Publish the raw numbers, including the failures — this is what makes the README credible
- [ ] Write the architecture doc, expanding on the diagrams already in this README
- [ ] Write the install/setup tutorial, tested against Month-6's fresh-machine setup
- [ ] Document hardware requirements honestly — what actually fits in 6GB vs. 8GB vs. more
- [ ] Compare your numbers to OpenHands/Aider/Cline's published numbers, plainly, without spin

**B. Kataria**
- [ ] Rewrite the README's intro once there's a real demo to point to — replace claims with evidence
- [ ] Record a short (1–2 minute), unscripted demo video — raw footage beats over-produced clips here
- [ ] Set up `CONTRIBUTING.md` and issue templates
- [ ] Lock in the `LICENSE` file
- [ ] Write the Show HN / r/LocalLLaMA post — lead with what it does, not a star count you don't control
- [ ] Publish

---

## License

MIT. Swap for Apache-2.0 if you want an explicit patent grant — pick before you publish, not after.

## Contributing

Fork → branch → PR. Add real contribution guidelines once the Month-6 docs pass is done — don't write rules for a project that doesn't run yet.

---

*None of the above means don't build this. It means build the one true thing first, demo it honestly, and let the rest follow.*
