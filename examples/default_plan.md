# Plan: `termloom` — a local-first terminal agent

## Context

We're building a terminal agent, installable via `pip`/`uv`, written in **Python + Rust**, that
runs Linux/terminal-style commands scoped to the current repo or directory. The core idea is a
**local-first** experience: a fast Rust engine (tries + trees) handles command completion and
ranking instantly, and only **escalates to Claude** for the hard questions — plain-English
requests ("find the biggest files", "summarize this directory"), fuzzy intent, or completions the
local model can't confidently resolve.

It ships as a **standalone REPL/TUI** (not a shell replacement or shell hook) with: a
command-completion dropdown, a recent/most-used panel, a model-health readout, syntax
highlighting, progress indicators, and a **confirm-before-run** prompt for risky commands.

This plan delivers value fastest by building the **fast local path first** (completion engine +
learning/ranking + the TUI), then layering Claude escalation, plain-English translation, and risk
gating on top. The environment has Python 3.12 and `uv` 0.11; **the Rust toolchain is not yet
installed** — Step 0 installs it.

Decisions locked with the user:
- **Integration:** standalone REPL/TUI.
- **Claude escalation:** tiered — `claude-haiku-4-5` for routine NL→command and fallback
  completion, escalate to `claude-opus-4-8` for hard/ambiguous queries; prompt-cache the stable
  system+repo-context prefix.
- **MVP focus:** fast local completion + learning, then escalation/NL/gating.

## Architecture

A single installable package with a Rust extension module and a Python application layer.

```
termloom/                      (uv/maturin workspace)
├─ pyproject.toml              # maturin build backend; entry point `termloom`
├─ rust/                       # PyO3 extension: termloom._engine
│  ├─ Cargo.toml
│  └─ src/
│     ├─ lib.rs                # PyO3 module surface
│     ├─ trie.rs               # prefix trie for command + arg completion
│     ├─ ranker.rs             # frequency/recency (frecency) scoring tree
│     ├─ history.rs            # persistent usage store (append log + snapshot)
│     └─ risk.rs               # fast risky-pattern matcher (rule table)
└─ src/termloom/               # Python layer
   ├─ __main__.py              # `python -m termloom` / console_script
   ├─ repl.py                  # event loop, key handling, dispatch
   ├─ ui/                      # TUI widgets (Textual)
   │  ├─ dropdown.py           # completion dropdown
   │  ├─ panel.py              # recent/most-used panel
   │  ├─ health.py             # model-health readout
   │  ├─ confirm.py            # confirm-before-run modal
   │  └─ highlight.py          # syntax highlighting
   ├─ completion.py            # orchestrates local engine ↔ Claude fallback
   ├─ scope.py                 # repo/dir scoping + activated-env detection
   ├─ executor.py              # runs commands in the scoped env, streams output
   ├─ risk.py                  # risk policy on top of engine rules; gating
   └─ claude/
      ├─ client.py             # Anthropic SDK wrapper, tiered routing, caching
      ├─ router.py             # decide local-vs-Claude and Haiku-vs-Opus
      └─ prompts.py            # NL→command + completion-assist prompts
```

**Rust ↔ Python split.** Rust owns the **hot path** that must feel instant: the completion trie,
the frecency ranker, the persistent usage store, and the first-pass risk pattern matcher. Python
owns **orchestration and everything I/O- or LLM-bound**: the TUI, command execution, Claude calls,
scoping, and policy. Binding via **PyO3**, built with **maturin** (works cleanly under `uv`).
Latency-critical calls (`complete(prefix) -> ranked candidates`, `record(command)`,
`risk_of(command)`) cross into Rust and return plain Python objects.

**TUI framework: Textual** (Rich-based). It gives us a real widget tree (dropdown overlay, side
panel, modal confirm, status bar), built-in syntax highlighting via Rich/Pygments, and progress
indicators — far less work than hand-rolling against `prompt_toolkit`/curses for the layout in the
approved mockup. `repl.py` drives a Textual `App`; keystrokes feed `completion.py`, which queries
Rust synchronously and (only when needed) kicks an async Claude task that streams into the UI.

**Scoping.** `scope.py` resolves the working root (git toplevel via `git rev-parse
--show-toplevel`, else cwd) and detects an activated environment (`$VIRTUAL_ENV`, `$CONDA_PREFIX`,
`pyproject`/`package.json` markers). All execution and Claude context are scoped to this root;
completions for paths/files are drawn from within it.

## Claude integration (escalation path)

Use the **official Anthropic Python SDK** (`anthropic`), per the claude-api skill — never raw
`requests`. Key choices, all grounded in the skill:

- **Tiered routing** (`claude/router.py`): local engine answers first. Escalate to Claude only
  when local confidence is low or the input is natural language (heuristic: input doesn't parse as
  a known command/flag, or starts with an interrogative/imperative phrase). Route routine
  translation to `claude-haiku-4-5`; escalate to `claude-opus-4-8` when Haiku returns low
  confidence or the request is multi-step/ambiguous.
- **Prompt caching** (`claude/client.py`): put the **stable system prompt + repo-context summary**
  first with `cache_control: {"type": "ephemeral"}`, volatile user query last. Verify hits via
  `usage.cache_read_input_tokens`. (See `shared/prompt-caching.md` — prefix-match invariant.)
- **Streaming** for any NL→command answer so the UI shows progress; use `client.messages.stream(...)`
  and `get_final_message()`.
- **Thinking/effort:** adaptive thinking with `output_config={"effort": "low"}` for Haiku routine
  calls (latency), `"high"` for Opus hard calls. Do **not** use `budget_tokens` (400s on these models).
- **Structured output:** NL→command returns a JSON object via `output_config={"format": {...}}`
  (`{command, explanation, risk, confidence, needs_confirm}`), so the UI never has to scrape prose.
  Do not use assistant prefills (400 on these models).
- **Model-health readout** reflects: API reachability, last-call latency, rolling error rate, and
  rough session token spend (from `usage`), surfaced by `ui/health.py`.
- Auth via `ANTHROPIC_API_KEY` env (bare `anthropic.Anthropic()`); never hardcode. Offline mode:
  if no key/health fails, the REPL stays fully usable on the local path and the readout shows
  "Claude: unavailable (local-only)".

## Risk gating (`risk.rs` + `risk.py`)

- Rust `risk.rs` holds a fast rule table flagging destructive/irreversible patterns: `rm -rf`,
  `dd`, `mkfs`, `:(){ :|:& };:`, force-push, `chmod -R` on broad paths, redirection over existing
  files, `sudo`, writes outside the scoped root, piping remote content to a shell, etc. Returns a
  risk level (`safe`/`caution`/`danger`) + matched reason(s).
- `risk.py` adds policy: `danger`/`caution` → `ui/confirm.py` modal (shows the command,
  highlighted, with the matched reason and a dry-run/explain option) that **blocks execution**
  until the user confirms. Claude-suggested commands carry their own `risk` field which is unioned
  with the local classifier (max severity wins — never downgrade).
- Anything Claude proposes that would run is **always** routed through the same gate; the LLM never
  executes directly.

## MVP and growth phases

**Phase 0 — Scaffold.** Install Rust (`rustup`), set up the maturin/uv workspace, stub PyO3 module,
`uv run termloom` launches an empty Textual app. Verify the Rust↔Python boundary imports.

**Phase 1 — MVP: fast local path (the usable core).**
- `trie.rs` completion over a seeded command corpus + live `$PATH` scan + in-repo paths.
- `ranker.rs` frecency scoring; `history.rs` persistent usage store (records every accepted command).
- `ui/dropdown.py` + `ui/panel.py` + `ui/highlight.py` + status bar (repo/branch + a static health
  chip for now).
- `executor.py` runs commands in the scoped env and streams stdout/stderr with a progress indicator.
- `risk.rs` + `ui/confirm.py`: basic risky-command gating wired in from day one (cheap, high-value).
- Outcome: a daily-usable completing REPL that learns and ranks your commands and guards dangerous ones.

**Phase 2 — Claude escalation + plain-English.**
- `claude/client.py` + `router.py` + `prompts.py`: NL→command translation, completion assist on
  low local confidence, "summarize this file/directory" (reads scoped files, sends to Claude).
- Live `ui/health.py` readout; streaming answers; tiered Haiku→Opus routing; prompt caching.

**Phase 3 — Polish & breadth.**
- Smarter ranking (context-aware: which commands follow which), arg/flag-level completion from
  `--help` parsing, richer dry-run/explain, config file for risk rules and model tiers,
  packaging/wheels for `pip install termloom`.
- (Deferred, per earlier discussion) optional zsh/bash shell-hook front-end reusing the same engine.

## Testing & evaluation

A dedicated `tests/` tree plus an `eval/` harness with checked-in datasets.

- **Local-path latency (the headline metric).** `pytest-benchmark` (Python) + `cargo bench`
  (criterion, Rust) on `complete()`/`record()`/`risk_of()`. Assert p50 < 5 ms and p99 < 25 ms for
  completion on a 10k-command corpus, so the local path always feels instant. Regression gate in CI.
- **NL→command accuracy.** `eval/nl_dataset.jsonl` of `{prompt, acceptable_commands[]}` pairs
  (start ~100, grow). Harness runs each through the router and scores **exact/semantic match**
  against the acceptable set; report top-1 accuracy per tier (Haiku vs Opus) and the escalation
  rate. Use an **LLM-as-judge** (Opus) for semantic equivalence where string match is too strict.
- **Claude cost/quality.** The eval harness records `usage` (input/output/cache tokens) per query
  and computes **$/query** per tier and **cache-hit rate**; assert the stable prefix caches
  (`cache_read_input_tokens > 0` on the 2nd+ call). Track quality-vs-cost as a table per model so
  the Haiku→Opus threshold can be tuned with data.
- **Risk gating correctness (safety-critical).** `eval/risk_dataset.jsonl` of labeled commands
  (`danger`/`caution`/`safe`). Assert **zero false-negatives on `danger`** (a dangerous command
  must never bypass the gate) and track false-positive rate on `safe`. Includes Claude-suggested
  commands — verify the union-with-local-classifier never downgrades severity. Property test: any
  command containing a known-destructive token always gates.
- **Executor scoping.** Tests that commands run with cwd = scoped root, that writes outside the
  root are flagged, and that an activated venv/conda env is respected.
- **TUI.** Textual's `App.run_test()` / `Pilot` for snapshot/interaction tests of the dropdown,
  panel, confirm modal, and health chip (type → dropdown appears → select → command staged →
  risky command → modal blocks until confirmed).
- **Reusable fixtures:** a temp git repo, a fake `$PATH`, a seeded history store, and a **mocked
  Anthropic client** (record/replay fixtures) so the bulk of tests run offline and deterministically;
  a small **live-API** suite (gated on `ANTHROPIC_API_KEY`) runs the real escalation path in CI nightly.

## Verification (end-to-end)

1. `rustup` present; `uv run maturin develop` builds the extension; `uv run termloom` launches the TUI.
2. Type `git ch` → dropdown shows ranked `git checkout`/`git cherry-pick`/… from the corpus +
   history; accept one → it runs in the scoped repo and its frecency increments (confirm via the
   recent/most-used panel on next launch).
3. Type a destructive command (`rm -rf build`) → confirm modal blocks; cancel → nothing runs;
   confirm → runs.
4. Type plain English ("show the 5 biggest files here") with `ANTHROPIC_API_KEY` set → routed to
   Haiku, streamed JSON answer, command staged behind the confirm gate; health chip shows latency.
5. `pytest` (offline, mocked Claude) and `cargo test`/`cargo bench` pass; `uv run python -m eval`
   prints NL accuracy, $/query per tier, cache-hit rate, and the risk-gate confusion matrix with
   zero `danger` false-negatives.
6. Unset `ANTHROPIC_API_KEY` → REPL still completes/ranks/gates locally; health chip shows
   "local-only".

## Open considerations (surface during build, not blocking)

- Confirm Textual meets the exact dropdown-overlay layout from the mockup; fall back to
  `prompt_toolkit` only if a true floating completion menu proves awkward.
- Decide the seed command corpus source (curated list of common commands + `tldr`-style data) vs.
  pure `$PATH`/`--help` discovery; start curated, grow from usage.
- Packaging: maturin produces platform wheels; decide CI build matrix (Linux first, given WSL target).
