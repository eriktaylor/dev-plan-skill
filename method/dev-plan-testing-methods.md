# Testing the dev-plan skill in Claude Code

A controlled comparison of how Claude plans the same complex task using **Claude
Code's built-in Plan Mode** versus the **dev-plan skill**. Both get the identical
prompt in fresh, isolated sessions; the planning mechanism is the only variable.

## Why this comparison

Comparing the skill against bare, unprompted Claude would be a strawman — "planning
beats no planning". The honest test is against Plan Mode, Claude Code's own
planning feature, which already produces structured plans. That answers the real
question: does the skill add anything over what's built in? (An optional third run
with neither — a plain prompt, no Plan Mode, no skill — gives the floor.)

## Setup at a glance

Two fresh Claude Code sessions in two empty directories. **Arm A** uses Plan Mode
(`/plan`); **Arm B** has the dev-plan skill installed and invokes `/dev-plan`.
Same model, same prompt, captured and compared. Claude Code sessions carry no
cross-conversation memory and an empty repo carries no project context, so each
run is isolated by construction.

## Held constant (so the test is fair)

- **Same model** in both arms — pick one and keep it identical; note which in the writeup.
- **Identical prompt**, pasted verbatim (below). No "write to a file" line — each arm saves by its own mechanism (see capture notes), so the prompt stays the single shared input.
- **Fresh, empty working directory** per arm — no leftover files, no `CLAUDE.md`; no skill in Arm A.
- **No memory / no prior turns** — a new `claude` session each time.
- **Only difference:** Plan Mode in Arm A vs the dev-plan skill in Arm B.

Run each arm once or twice. Plan Mode's output varies run-to-run; the skill's
should stay structurally consistent — that contrast is itself a result.

## Prerequisites

- Claude Code installed and authenticated (`claude --version`). `/plan` needs v2.1.0+; skill auto-loading from `.claude/skills/` needs v2.1.157+ (both well past now).
- The `dev-plan/` folder downloaded locally (contains `SKILL.md` and `references/complex-plan.md`). Commands below assume `~/Downloads/dev-plan` — adjust to where you saved it.

## The prompt

Paste verbatim in both arms. It describes the project the way a developer would
and deliberately says nothing about tracks, scoring, diagrams, or waves — so
neither arm gets structural hints from the prompt.

```
Build a terminal agent, installable via pip or uv, written in Python and Rust.
It runs Linux/terminal-style commands scoped to the current repo or directory
(including operations inside an activated environment). A local backend uses
trees and tries for fast command completion and escalates to Claude for harder
questions. It completes commands, learns and ranks your most-used ones, and
accepts plain-English commands like searching files or summarizing a document
or directory.

It has in-terminal visual elements: a command-completion dropdown, a
recent/most-used commands panel, a model-health readout, and a confirm-before-run
prompt for risky commands — with basic syntax highlighting and progress
indicators.

Plan how you'd test and evaluate it too: the accuracy of plain-English-to-command
translation, the latency of the local path, the quality and cost of Claude
escalations, and that risky commands are correctly gated.

Plan what to build first for a usable MVP and how it grows from there.
```

## Arm A — built-in Plan Mode (`/plan`)

```bash
cd ~                                    # /home/<user-name>
mkdir -p ~/devplan-test/planmode        # fresh, empty working dir
cd ~/devplan-test/planmode

ls -la                                  # expect empty; no .claude here
claude                                  # same model as Arm B
```

In the session: type `/plan` (or Shift+Tab twice) to enter Plan Mode, then paste
the prompt. Plan Mode is **read-only** — it won't write files; it produces the
plan and auto-saves it. Capture it, then exit without approving (we want the plan,
not execution):

```bash
# find the plan Claude Code just saved
ls -t ~/.claude/plans/ | head -1
# copy that file out (paste its name in place of <file>)
cp ~/.claude/plans/<file>.md ~/devplan-test/planmode-plan.md
```

If the save location differs on your version, just copy the plan text Claude
printed in the session into `~/devplan-test/planmode-plan.md`.

## Arm B — dev-plan skill (`/dev-plan`)

```bash
cd ~                                    # /home/<user-name>
mkdir -p ~/devplan-test/skill/.claude/skills
cd ~/devplan-test/skill

# install the skill (adjust the source path to where you saved it)
cp -r ~/Downloads/dev-plan ~/devplan-test/skill/.claude/skills/dev-plan

# verify structure
ls .claude/skills/dev-plan              # expect: SKILL.md  references/
ls .claude/skills/dev-plan/references   # expect: complex-plan.md

claude                                  # same model as Arm A
```

In the session, first confirm the skill loaded — ask `What skills are available?`
or check that `/dev-plan` autocompletes. Then run `/dev-plan` and paste the prompt.
The skill writes the plan to the repo as part of its behavior; collect that file:

```bash
ls ~/devplan-test/skill/*.md            # the plan the skill wrote
```

## Arm C — optional: does the nudge fire on its own?

Same install as Arm B, but **do not** type `/dev-plan` — just paste the prompt and
watch whether Claude reaches for the skill itself (it's model-invocable).

```bash
cd ~
mkdir -p ~/devplan-test/nudge/.claude/skills
cp -r ~/Downloads/dev-plan ~/devplan-test/nudge/.claude/skills/dev-plan
cd ~/devplan-test/nudge
claude
# → paste the prompt only; do NOT type /dev-plan
```

## Optional floor — neither

A third empty dir, no skill, plain prompt, no `/plan`. Shows what default Claude
gives with zero scaffolding — the baseline both Plan Mode and the skill build on.

## Compare

```bash
# Plan Mode output vs skill output
diff ~/devplan-test/planmode-plan.md ~/devplan-test/skill/*.md
```

They're structured differently, so opening both in an editor side by side reads
better than the line diff.

## What to look for

Both Plan Mode and the skill produce *a* structured plan — the question is what
the skill's durable format adds:

- Is the work classified into tracks (tooling, features, eval/testing, viz), or planned as one undifferentiated list?
- Are items scored and ordered, with a dependency view and explicit waves / an MVP-first cut — or a flat sequence?
- Does it raise the latent decisions (Rust vs not, pip vs uv, escalation threshold) as tracked open questions with defaults?
- Does it flag the production-safety surface — this tool runs shell commands in your repo?
- Is the plan a durable, updatable artifact with a status vocabulary and a single home for state — or a one-time plan meant to be consumed and discarded?
- Across repeated runs, how much does Plan Mode's shape drift versus the skill's?
