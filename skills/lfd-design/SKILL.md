---
name: lfd-design
description: Design a loss function and harness for a long-running /goal optimization run (loss-function development, LFD). Use when the user wants to set up an autonomous optimization loop, distill a product from public artifacts, turn a spec into an optimization target, or asks to design a /goal. Interrogates the task, designs a blinded eval, generates the scorer/probe/log harness files, red-teams the target for cheats, and emits goal.md ready to launch.
---

# LFD Design

You are designing an optimization target, not solving a task. The agent that
receives `goal.md` is a competent, tireless, literal optimizer: it will satisfy
the target by the cheapest available path — memorizing the eval, hardcoding
answers, mining feedback channels into lookup tables. Your job is to make
genuine capability the cheapest path left.

A spec says "build this, make the tests pass." A loss function says "build
this, make the tests pass, then descend toward this bar on data you cannot
see." You are writing the second thing. It has four parts: the **target**, the
**constraints**, the **instruments**, and the **forced entropy**. Every /goal
you emit must contain all four.

## Phase 1 — Interrogate

Ask the user in ONE batched round (do not guess, do not proceed on vibes):

1. **Outcome** — what artifact or behavior, and what does "good" look like?
   Is there a reference artifact (a product, a dataset, a known-correct
   output) to score against?
2. **Eval source and size** — where do ground-truth cases come from, and how
   many are obtainable? If fewer than ~200, warn explicitly: a small eval is
   enumerable and the agent WILL memorize it. Suggest ways to widen it
   (mine public artifacts, perturb seeds, broaden the date/entity range).
3. **Budgets** — wall-clock budget for the run, dollar ceiling, and which
   paid surfaces exist (crawler credits, LLM keys). An 80% solution in 2
   hours beats a 100% one in 30 days; get the user's actual tolerance.
4. **Surface** — what the agent may touch: directories, APIs, providers,
   models, concurrency. Everything unlisted is denied.
5. **Acceptance** — the score bar, measured on held-out data only, plus a
   diminishing-returns stop ("if marginal gain ≈ 0 for N cycles, stop and
   report").

## Phase 2 — Design the loss function

**Target.**
- The metric must be mechanically computable by a script, at the right
  resolution for the claim. An LLM judge that "compares two screenshots"
  approves 12px spacing errors; a pixel-diff does not. Match the instrument
  to the precision the user actually wants.
- The metric must penalize BOTH failure directions. Recall without precision
  invites a return-everything cheat; precision without recall invites a
  return-one-thing cheat. If the user gives a one-sided metric, fix it and
  tell them why.
- Split the eval: `eval/dev` (scored freely, per-item misses allowed) and
  `eval/holdout` (scored rarely, aggregate number only, acceptance measured
  here exclusively). Holdout should live OUTSIDE the agent's working
  directory if at all possible.
- Run a leak audit on every feedback channel: bits revealed per scoring call
  × expected number of cycles — can the agent reconstruct the eval before
  the run ends? If yes, cut feedback resolution (cap the miss list, return
  aggregates) or grow the set.

**Constraints.**
- Wall-clock budget, stated in the /goal. Agents have no sense of time and
  will grind 10 hours for 2%.
- Dollar and credit ceilings per paid surface.
- Surface allowlist from Phase 1.
- Methodology rules (LLM-in-the-data-plane allowed? deterministic only?).
- **Capacity caps** on every artifact that could function as a lookup table:
  keyword lists, regex sets, seed data, special-case branches. Name the
  artifact and the cap explicitly ("keyword list ≤ 20 entries; no literal
  string in the codebase may match an eval item").

**Enumerate the cheats.** Read `references/cheat-museum.md`, then list at
least 10 ways a lazy optimizer could max THIS metric without solving THIS
task. For each, write the fence: a constraint in `goal.md` AND a way to
detect violation (a lint, a cap check, the probe gap). A constraint without
an instrument is a vibe — the agent will violate it cheerfully because it
can't tell it's violating it.

## Phase 3 — Generate the harness

Write these files now, tailored to the task. Do not ship placeholders.

- `harness/score.sh` — the task-specific scorer. Pixel-diff for a UI clone
  (deterministic rendering: frozen time, animations off, pinned fonts,
  fixed viewport), recall@k + precision for retrieval, structured JSON diff
  for API behavior. Scores `eval/dev` by default; `--holdout` returns one
  aggregate number and appends to an audit log.
- `eval/dev/` and `eval/holdout/` — the split. Tell the user where holdout
  answers should live if outside the repo.
- `harness/probe.sh` — generates perturbed variants of dev cases
  (paraphrases, date shifts, entity swaps) and reports the dev-vs-probe
  score gap. The gap is the memorization gauge.
- `harness/status.sh` — wall-clock elapsed, cycles run, score history,
  spend notes. Time is a first-class instrument, not a footnote.
- `LOG.md` — iteration log template: one entry per cycle with
  `hypothesis / expected failure mode / diagnostic / result`. This is what
  survives context compaction.

The /goal must declare harness files read-only for the optimizer. Editing
the scorer is the oldest cheat there is.

## Phase 4 — Red-team your draft

Before emitting, simulate the laziest possible agent against your draft
/goal: what is the five-minute win? Common ones: seed data that mirrors the
eval, mining per-item miss feedback into a keyword lookup table, gaming a
judge, editing the scorer, declaring victory on the dev set. Patch the
draft and simulate again. Emit only when three consecutive simulations find
nothing cheaper than doing the real work.

## Phase 5 — Emit goal.md

Use this structure:

```markdown
# Goal: <one-line outcome>

## Target
<metric definition, both directions> · Bar: <score> on holdout.
Score with `harness/score.sh`. Holdout: aggregate-only, max <N> calls per
<period>. Acceptance is measured on holdout exclusively.

## Constraints
- Wall-clock budget: <hours>. Check `harness/status.sh` every cycle.
- Spend ceilings: <per surface>.
- Surface: <allowlist>. Everything else is off-limits.
- Capacity caps: <artifact ≤ N, no eval-shaped literals>.
- `harness/` and `eval/` are read-only. Never read eval answer data.

## Cycle protocol
1. Score (dev). 2. Reflect: run `harness/probe.sh` — am I generalizing or
memorizing? If the probe gap is growing, the next change must REMOVE an
eval-shaped artifact (cap a list, blind a feature, reject a seed), never
add one. 3. Hypothesize: log hypothesis, expected failure mode, and
diagnostic in LOG.md BEFORE changing code. 4. Change. 5. Log the result.

## Entropy rules
- Stall rule: if the metric didn't move last cycle, the next attempt must
  be a structural change — same-knob-harder is banned.
- Exploration quota: every <K> cycles, try a structurally different
  approach even if the current one is still inching up.

## Stop conditions
Bar hit on holdout · any budget exhausted · marginal gain ≈ 0 for
<N> consecutive cycles. On stop: write a final report in LOG.md.
```

## Phase 6 — Pre-flight (tell the user, every time)

1. Use a disposable API key with a provider-side spend limit.
2. Confirm holdout answers are not readable from the agent's cwd.
3. Run `harness/score.sh` once yourself — a broken scorer optimizes noise.
4. **Babysit cycle 1.** Watch what the agent touches and confirm it uses
   the instruments. Then go to bed.
