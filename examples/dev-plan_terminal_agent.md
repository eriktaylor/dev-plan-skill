# Terminal Agent — plan

Shape: **complex** — 5 interdependent tracks, a hot-path/escalation split, and a
load-bearing safety gate (it executes shell commands), so ordering and risk matter.

A terminal agent ("TA"), pip/uv-installable, Python orchestration over a Rust hot
path. Completes and ranks shell commands from a local trie + frecency store
(fast, offline), escalates harder questions and plain-English requests to Claude.
In-terminal UI: completion dropdown, recent/most-used panel, model-health readout,
confirm-before-run gate, syntax highlighting, progress. Commands are scoped to the
current repo/dir and the activated environment.

**In scope:** local completion engine, NL→command via Claude, the TUI elements,
risky-command gating, and the eval harness for all four.
**Out of scope (for now):** multi-shell daemon, remote/SSH targets, plugin API,
team-shared history sync — see Deferred.

## Classification

- **T — tooling / packaging:** the installable Python+Rust skeleton, config, CLI.
- **F — features:** the engine — execution, trie/frecency, completion, escalation, NL.
- **V — visualizations / TUI:** the in-terminal widgets.
- **S — safety:** risky-command classification and gating.
- **E — evaluation / testing:** accuracy, latency, escalation cost/quality, gate tests.

## Status legend

✅ complete · 🔄 partial / in-progress · 💡 proposed (recommended next) ·
⏸️ deferred (with re-open trigger) · unmarked = not started

Scales: Difficulty **S** (<1h) / **M** (hours) / **L** (>1d) · Impact & Risk
**Low / Med / High** · Priority is the High/Med/Low rollup of Impact÷Difficulty,
broken by Risk.

## Phase diagram (waves + dependencies)

    Wave 1 — installable local skeleton (MVP foundation):
      T-1 ──→ F-2 ──→ F-3                      (critical path)
      T-1 ──→ F-1
              S-1                              (rules-only, parallel)

    Wave 2 — usable LOCAL MVP (no Claude required):
      F-3 ──→ V-1 (dropdown)
      F-2 ──→ V-2 (recent/most-used panel)
      S-1 ──→ V-4 (confirm-before-run) ──→ F-1
      F-3 ──→ V-5 (syntax highlight)
      F-3 ──→ E-2 (local latency bench)
      S-1 ──→ E-4 (gate test suite)

    Wave 3 — Claude escalation + plain-English:
      F-3 ──→ F-4 (escalation client) ──→ F-5 (NL→command)   (critical path)
      F-4 ──→ V-3 (model-health)
      F-4 ──→ V-6 (progress indicators)
      F-5 ──→ E-1 (NL→cmd accuracy)
      F-4 ──→ E-3 (escalation cost/quality)

Critical path: **T-1 → F-2 → F-3 → F-4 → F-5**. Wave 2 ships a genuinely useful
offline tool before any Claude dependency exists; Claude is additive, not required.

## Implementation order

| Wave | Item | Prereqs | Diff. | Imp. | Risk | Priority | Touches user shell? | Status |
|------|------|---------|-------|------|------|----------|---------------------|--------|
| 1 | T-1 packaging (maturin, pip/uv) | — | M | High | Low | High | no | 💡 |
| 1 | F-1 scoped execution + history capture | T-1 | M | High | High | High | **yes** | |
| 1 | F-2 Rust trie + frecency store | T-1 | M | High | Low | High | no | |
| 1 | S-1 risky-command classifier (rules) | — | M | High | High | High | gates exec | |
| 1 | F-3 prefix completion engine | F-2 | M | High | Low | High | no | |
| 2 | V-1 completion dropdown | F-3 | M | High | Low | High | no | |
| 2 | V-4 confirm-before-run prompt | S-1,F-1 | S | High | High | High | **yes** | |
| 2 | V-2 recent/most-used panel | F-2 | S | Med | Low | Med | no | |
| 2 | V-5 basic syntax highlighting | F-3 | S | Med | Low | Med | no | |
| 2 | E-2 local-path latency bench | F-3 | S | High | Low | High | no | |
| 2 | E-4 gate test suite | S-1 | M | High | High | High | no | |
| 3 | F-4 Claude escalation client + health | F-3 | M | High | Med | High | no | |
| 3 | F-5 NL→command translation | F-4 | L | High | Med | High | feeds gate | |
| 3 | V-3 model-health readout | F-4 | S | Med | Low | Med | no | |
| 3 | V-6 progress indicators | F-4 | S | Med | Low | Med | no | |
| 3 | E-1 NL→command accuracy harness | F-5 | L | High | Med | High | no | |
| 3 | E-3 escalation cost/quality eval | F-4 | M | High | Med | High | no | |

## Next steps

Working set: nothing in flight yet. **Proposed first tranche = Wave 1** (T-1, F-2,
S-1, then F-1 + F-3). That lands an installable package with a working local trie,
the rules gate, and scoped execution — the spine everything else hangs off.

Recommend starting with **T-1 alone** as the very first check-in: prove
`uv pip install -e .` builds the Rust extension and exposes a CLI entrypoint
before any logic is written. Want me to take T-1?

## Open design questions

| Question | Default unless overridden |
|----------|---------------------------|
| TUI framework | `prompt_toolkit` (purpose-built for line editing + completion dropdowns; Textual is heavier for a shell line). |
| Shell integration model | Standalone REPL wrapper first; readline/ZLE shell-hook deferred (see ⏸️). |
| Rust↔Python binding | PyO3 + maturin; single wheel, no separate toolchain at install. |
| Claude tiering | Haiku 4.5 for fast NL→command; escalate to Opus 4.8 only on low-confidence/complex asks. Cache system prompt. |
| History source | TA captures its own session history; optional one-time import of `~/.bash_history`/`~/.zsh_history`. |
| Risky-command list | Curated regex/AST rules (rm -rf, dd, mkfs, :(){ }, force-push, chmod -R, curl\|sh, > /dev/sd*), tunable per-repo. |
| Risk gate vs NL output | NL-suggested commands ALWAYS pass through S-1 and the confirm prompt — never auto-run. |

## Scoring rubric

Two axes plus Risk, since TA executes commands against the user's filesystem.
- **Difficulty** — engineering effort (S/M/L by wall-clock).
- **Impact** — lift to a usable, trustworthy MVP.
- **Risk** — blast radius if wrong. High = can destroy user data or run unintended
  commands (F-1, S-1, V-4, F-5). Risk breaks ties and pulls dangerous items early
  enough to watch — S-1 and its tests (E-4) ship in Waves 1–2, before NL exists.

## Production safety

"Production" here = the user's actual shell and filesystem. The gate is the
product, not a feature.

**Touch-points**

| Module / file | Items | Essential path? | Risk surface |
|---------------|-------|-----------------|--------------|
| `executor` (subprocess + cwd scope) | F-1, V-4 | yes | runs real commands; cwd/env confinement |
| `safety/classifier` | S-1, E-4, F-5 | yes | a miss = unconfirmed destructive command |
| `escalation/client` | F-4, F-5, V-3, E-3 | no | sends shell context off-box (data egress) |
| `store` (trie/frecency, on disk) | F-2, F-3, V-2 | no | local only; no secrets persisted |

**Data-safety invariants**

1. No secrets, env-var *values*, or file *contents* in Claude escalation payloads —
   send command text, cwd basename, and dir listing names only; redact tokens.
2. No model-suggested or completed command auto-executes; every run of a
   classified-risky command passes the S-1 gate + V-4 confirm (invariant cannot be
   bypassed by the completion or NL path).
3. Execution is confined to the activated cwd/repo; no implicit `cd` outside scope
   without explicit confirm.
4. History store is local-only and never transmitted; no network writes from the
   store module.

**Reversibility:** `--offline` / config flag disables F-4/F-5 entirely → pure
local tool (Wave 2 behavior). The S-1 gate is always-on and has no disable flag.

## Item details

### T-1 — packaging skeleton (maturin, pip/uv)
maturin-built mixed Python+Rust project; `pyproject.toml` exposes a `ta` console
entrypoint. CI builds wheels for linux. Goal: `uv pip install -e .` works and `ta`
launches an empty REPL.
**Files:** `pyproject.toml`, `Cargo.toml`, `src/lib.rs`, `python/ta/__main__.py`
**Safety:** none (no exec yet).

### F-1 — scoped execution + history capture
Subprocess executor pinned to cwd and the active env; records each run (command,
exit, cwd, timestamp) into the store for ranking.
**Files:** `python/ta/executor.py`, `store` (writes)
**Safety:** inv. 2, 3 — must route risky commands through S-1/V-4.

### F-2 — Rust trie + frecency store
PyO3 module: prefix trie for completion + frecency (recency×frequency) ranking,
persisted to a local file. The hot path; must answer completion queries in <10ms.
**Files:** `src/trie.rs`, `src/frecency.rs`, `src/lib.rs`
**Safety:** inv. 4 — local only.

### F-3 — prefix completion engine
Python layer over F-2: given the current line, return ranked candidates (history +
common-command corpus). Confidence score decides whether to offer locally or hint
escalation.
**Files:** `python/ta/complete.py`
**Safety:** none.

### F-4 — Claude escalation client + health
Anthropic SDK client, tiered (Haiku→Opus), prompt-cached system prompt, timeout +
retry, and a health signal (latency, error rate, last-ok) feeding V-3. Cost meter
per call.
**Files:** `python/ta/escalation/client.py`, `python/ta/escalation/health.py`
**Safety:** inv. 1 — payload redaction lives here.

### F-5 — NL→command translation
Plain-English → candidate command(s) via F-4, with a structured output contract
(command + rationale + risk self-assessment). Output is re-checked by S-1 and
always confirmed.
**Files:** `python/ta/nl.py`
**Safety:** inv. 1, 2.

### V-1/V-2/V-3/V-4/V-5/V-6 — TUI
Completion dropdown, recent/most-used panel, model-health readout, confirm-before-run
prompt, syntax highlighting, progress indicators — all in `prompt_toolkit`.
**Files:** `python/ta/ui/*.py`
**Safety:** V-4 enforces inv. 2 at the UI layer.

### S-1 — risky-command classifier
Rules engine (regex + light shell-AST) tagging commands Low/Med/High risk with a
reason; High requires explicit confirm, never silent. Per-repo overrides allowed,
but cannot disable High entirely.
**Files:** `python/ta/safety/classifier.py`, `python/ta/safety/rules.toml`
**Safety:** inv. 2 — the gate itself.

### E-1 — NL→command accuracy harness
Labeled set of (English, gold command) pairs; metrics: exact-match, semantic-match
(normalized), and execution-equivalence on a sandbox. Tracks Haiku vs Opus.
**Files:** `eval/nl_accuracy/`
**Safety:** runs in a throwaway sandbox only.

### E-2 — local-path latency benchmark
p50/p95/p99 for completion queries against a seeded store; budget: p95 < 10ms,
end-to-end keystroke→dropdown < 30ms. Regression-gated in CI.
**Files:** `eval/latency/`

### E-3 — escalation cost/quality eval
Per-call token + dollar accounting, escalation rate, and a quality score
(did the suggestion match gold / run cleanly). Watches the local-vs-Claude split.
**Files:** `eval/escalation/`

### E-4 — gate test suite
Adversarial corpus of destructive/obfuscated commands (encoded `rm`, aliased
force-push, `curl|sh`) asserting S-1 flags High and V-4 blocks auto-run. Zero
false-negatives is the bar; tracks false-positive rate as a UX cost.
**Files:** `eval/safety/`

## Deferred & future

- ⏸️ **Shell-hook integration (readline/ZLE widget into the user's real shell).**
  Why not now: standalone REPL proves the engine without per-shell quirks.
  Re-open: once local completion + gate are validated and users want it in their
  native shell.
- ⏸️ **Remote / SSH-scoped execution.** Why not now: multiplies the safety surface.
  Re-open: when single-host local UX is solid and a concrete remote need appears.
- ⏸️ **Team-shared / synced history & rankings.** Why not now: invariant 4 keeps
  the store local; sync needs a privacy model. Re-open: if teams ask to share
  command corpora.
- ⏸️ **Plugin API for custom commands/classifiers.** Re-open: after the rules
  format (S-1) stabilizes and a third use-case appears.
