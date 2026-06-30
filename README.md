# dev-plan-skill

*A Claude Code skill for structured, scored, terminal-readable development plans — the same format every session.*

![Development Plan Skill Chart](dev-plan_skill_image.jpg)

`/dev-plan` turns a one-off planning request into a plan that lives in your repo and survives across sessions: work split into tracks, scored and ranked, dependencies drawn as plain-text arrows, status tracked with glyphs. The skill holds the scaffold; you keep the thinking.

## Install

Clone the repository to get started:
```bash
git clone https://github.com/eriktaylor/dev-plan-skill.git
```

For a global install, available in every project:
```bash
mkdir -p ~/.claude/skills
cp -r ~/dev-plan-skill/dev-plan ~/.claude/skills/dev-plan
```

Or scope it to a specific project directory:
```bash
mkdir -p .claude/skills
cp -r ~/dev-plan-skill/dev-plan .claude/skills/dev-plan
```

The resulting folder layout will look like this:
```text
.claude/
└── skills/
    └── dev-plan/
        ├── SKILL.md                  # Router + simple-plan format
        └── references/
            └── complex-plan.md       # Full complex format
```

Now the skill is installed and ready to use. To run it in the CLI:
```bash
claude
# Inside the session, type: /dev-plan
```

## What a complex plan looks like

Status is a single glyph per item, flipped in place:

```
✅  complete
🔄  in-progress
💡  proposed (recommended next)
⏸️  deferred (with re-open trigger)
·   not started
```

Every item is scored, and ranked by a simple rule:

```
Priority = Impact ÷ Difficulty        (ties broken by Risk)
```

Dependencies are plain text, grouped into waves, with the critical path called out:

```
Wave 1 — installable local skeleton (MVP):
  T-1 ──→ F-2 ──→ F-3                  (critical path)
  T-1 ──→ F-1
          S-1                          (rules-only, parallel)

Wave 2 — usable LOCAL MVP (no Claude required):
  F-3 ──→ V-1 (dropdown)
  S-1 ──→ V-4 (confirm-before-run) ──→ F-1
```

## Simple or complex

The skill routes on one decision. A **simple** plan is a short, sequential change — goal, steps, done-when. A **complex** plan — interdependent tracks, anything that touches production — gets the full format above, loaded from `references/complex-plan.md` only when it's needed.

## Repo layout

```text
dev-plan-skill/
├─ README.md
├─ dev-plan/                          # the skill — copy into ~/.claude/skills/
│  ├─ SKILL.md                        # router + simple-plan format (always loaded)
│  └─ references/complex-plan.md      # full complex format (loaded on demand)
├─ method/dev-plan-testing-methods.md # reproduce the Plan Mode vs skill test
└─ examples/
   ├─ default_plan.md                 # Arm A — Claude /plan mode
   └─ dev-plan_terminal_agent.md      # Arm B — /dev-plan skill
```

## Learn more

The story, the design decisions, and the head-to-head test against Claude Code's Plan Mode are in the [Medium article](https://medium.com/@erikntaylor/making-better-development-plans-into-a-skill-for-claude-code-093db647b404). The two plans in `examples/` are the real outputs from that test.
