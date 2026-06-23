# Complex plan format

Use this when a plan has several interdependent items, parallel tracks, or real
uncertainty about ordering and payoff. It layers scoring, a dependency diagram,
and waves on top of the simple format. Everything stays plain-text and
terminal-readable — the rubric adds structure, not rendering.

## Document shape

Three tiers, top to bottom: the scannable plan first, reference notes next,
per-item detail last.

**At a glance (top)** — the state, the next move, and what's still open.
1. **Header** — what the plan explores, what's in and out of scope, how it
   relates to sibling plans. One short paragraph.
2. **Classification** — the kinds of work the plan covers (tooling, features,
   visualizations, evaluation/testing, infra…) and their ID prefixes.
3. **Status legend** — the glyph vocabulary (✅ / 🔄 / 💡 / ⏸️ / unmarked).
4. **Phase diagram** — the dependency graph in ASCII, grouped into waves, with
   the critical path marked.
5. **Implementation order** — the single scored, ranked table.
6. **Next steps** — the current working set: what's in flight and the immediate
   next move, flushed into the sections above as work completes.
7. **Open design questions** — unresolved choices, each with a default; sits with
   Next steps because both are read each turn and flushed as decisions land.

**Notes (middle)** — the reference the top tier leans on; read for the why.
8. **Scoring rubric** — how the scores in the table are assigned.
9. **Production safety** — what a change touches, what can break, and the
   data-safety lines it must not cross.
10. **Empirical findings** (optional) — directional evidence that steers
    priorities, added as work produces data.

**Detail (bottom)** — long-form, read per item while working it.
11. **Item details** — one block per item: what/why, files, safety.
12. **Deferred & future** — parked work (deferred, out-of-scope, or
    future/aspirational), each ⏸️ with a re-open trigger.

Few plans need all twelve. The top tier plus the scoring rubric are load-bearing;
the rest earn their place as a plan gets riskier or more contested.

## Classification

Decide up front what kinds of work the plan covers, because they roll out and get
validated differently — tooling, features, visualizations, evaluation/testing,
infra, docs. Give each track a short ID prefix and tag every item with it (e.g.
T-1 tooling, F-2 feature, V-1 visualization, E-3 evaluation). The prefixes make
the phase diagram and the table scannable by track and keep parallel work legible.

Call out evaluation/testing explicitly — it's the track most often dropped. An
eval or test item is real work with its own slot, distinct from the empirical
findings it later produces. If the right set of tracks isn't obvious yet, start it
as an open design question and settle it here once decided.

## Phase diagram (waves + dependencies)

Draw the items as a dependency graph: nodes are items, arrows (──→) mean "must
finish before". Tag each node with its status glyph so the diagram reads as a
state map at a glance, not just dependencies. The groupings ARE the execution
waves — reviewable batches you ship in order, each grouped by what unblocks what.
Mark the critical path: the chain that gates everything else.

    Wave 1 — fix + validate:
      ✅ A ──→ ✅ B ──→ 🔄 C      (critical path)
      💡 D ─────────              (independent, parallel)

    Wave 2 — features (after Wave 1):
      E ──→ F ──→ G
      H ──────────→ G            (H parallel with E; both feed G)

This is a PERT / critical-path view, kept in plain text so it diffs and reads in
a terminal — the ASCII is the artifact, not a rendering tool. A wave ships as one
or more **tranches**: the smallest chunk that rolls out cleanly on its own. After
each pass, name the next tranche and check in.

Once a wave is fully ✅, retire it to a brief summary — collapse its nodes and
rows down to a short note of what shipped, with a link to the PR or commits — so
the plan stays focused on what's live and ahead. The detail lives in git; the
summary is just enough to remember what happened.

## Implementation order

One scored, ranked table — this is both the order of work and the score rollup,
so there's no separate summary. Suggested columns:

| Wave | Item | Prereqs | Diff. | Imp. | Priority | Touches prod? | Status |

Add a **Risk** column for infra-heavy plans (see the rubric). The "Touches prod?"
column earns its place when some items are staging-only and others change live
behavior — it makes the dangerous rows visible without reading each one. Scores
live only in this table — the item blocks don't repeat them, so a re-score can't
drift. Status shows as a glyph in both this table and the phase diagram for quick
reference; the table's Status cell adds the short note.

## Next steps

The working set: what's in flight right now and the immediate next move, placed
just below the diagram and table because it's read off them. Treat it as a
scratch-pad, not a log — hold only the current tranche.

As work lands, flush it: push the results into the durable sections above (flip
status glyphs in the table and diagram, add a finding, update an item block), then
clear the finished lines and write what's next. Done-state lives in the glyphs,
not here, so the two never drift and this section stays a few lines however long
the plan runs. History is git's job, not this section's.

## Open design questions

Sits with Next steps: read every turn, flushed as decisions land. Unresolved
choices don't have to block work if each ships with a sensible default. Capture
them in a two-column table:

| Question | Default unless overridden |

State the default you'll proceed with so the plan stays executable, and let the
human override any row. A question can carry its own re-open trigger — a condition
that, once met, reopens the decision (e.g. "if the backstop handles >50% of
events, revisit the always-on host"). When a question is settled, flush it: fold
the decision into the affected sections and drop the row.

## Scoring rubric

How the scores in the table above are assigned.

Default to two axes:
- **Difficulty** — engineering effort.
- **Impact** — expected lift to the outcome you care about, or risk reduction
  for tooling/infra items.

**Priority = Impact ÷ Difficulty.** High-impact, low-effort items rise to the
top. It's a sorting heuristic, not a law — call out when an item's real value
isn't captured by the headline objective (e.g. a config knob whose payoff shows
up in production, not in an offline score).

**Risk as a core axis.** For infra- or production-heavy plans, promote **Risk**
(blast radius if it goes wrong) to a first-class axis and score Difficulty ·
Risk · Impact. Use it to break ties and order within a wave — a high-impact,
high-risk item usually wants to ship alone and early enough to watch. Leave it
out for feature plans where nothing is load-bearing.

**Scale.** Either calibrate 1–5, or use T-shirt sizes when that's faster and
precision is false comfort: Difficulty S (<1h) / M (hours) / L (>1d); Risk and
Impact Low / Med / High. Pick one scale per plan and state it in the legend.

**Alternative: RICE.** When reach and confidence matter (user-facing features,
bets with unknown payoff): Priority = (Reach × Impact × Confidence) ÷ Effort.
Pick one scheme per plan; don't mix item to item.

**Optional extra axes** (only if they earn their keep): context availability (how
much is known vs. must be discovered first); testability (how cheaply it can be
validated before promotion).

Mark items that depend on an earlier item proving value as **Unscheduled** —
captured to avoid losing them, not committed work.

## Production safety

For any plan that can change live behavior, give safety its own real estate
rather than scattering caveats through the items. It answers three questions:
what code does a change touch, what breaks if it's wrong, and is it on an
essential path? — plus the data-safety lines the project won't cross.

**Touch-points table.** The blast-radius view: which shared modules each item
touches, and whether that path is essential.

| Module / file | Items that touch it | Essential path? | Risk surface |

It answers "if I change this, what's exposed?" without reading every item block,
and surfaces two items in the same wave touching the same file.

**Data-safety invariants.** Rules the plan must not cross, stated once as a
numbered list — e.g. "no PII in model-escalation payloads", "no raw user content
in logs", "schema migrations are additive only: guarded ADD COLUMN / CREATE TABLE
IF NOT EXISTS, never drop or rename". Per-item blocks reference an invariant by
number instead of repeating it. State the lines not to cross; don't prescribe how
a feature is built.

**Reversibility.** For anything on a live path, note how to turn the change off
and fall back to current behavior.

## Empirical findings

Optional. As the work produces data, record directional evidence that steers
priorities here — a line or two per finding, noting any item whose score or order
it changes. These are the readout of an evaluation/testing item: the item is the
work, the finding is the result. Keep them short.

## Item details

One block per item, at the bottom because it's the long-form you read while
working a specific item:

    ### <ID> — <short title>

    <what it is and why, grounded in the actual files/code where possible>

    **Files:** <paths touched>
    **Safety:** <which invariants apply; any per-item risk>

The block is static reference — what it is, where it lives, how it's guarded. Keep
prose tight; favor the specific (file paths, the one caveat that matters) over
narration.

## Deferred & future

Capture work you're consciously not doing now — deferred, out-of-scope, or
future/aspirational — so it isn't lost or quietly relitigated. Mark each ⏸️ and
give it two things:

- **Why not now** — one line on the cost/benefit, or for aspirational items, what
  would make it worth doing.
- **Re-open trigger** — the concrete condition that brings it back (e.g. "when
  the repo goes private", "when the daily run has to fire with the laptop closed",
  "if we ever want offline support").

An item with no re-open trigger is just a dropped item — give it the trigger or
cut it.
