---
name: lfd-design
description: Design a loss function and harness for a long-running /goal optimization run (loss-function development, LFD). Use when the user wants to set up an autonomous optimization loop, distill a product from public artifacts, turn a spec into an optimization target, or asks to design a /goal. Observes the existing environment, interrogates the task, ingests or generates the spec, builds a blinded eval, generates and verifies the harness, red-teams the target for cheats, and emits goal.md ready to launch. Re-invoke in patch mode when a running loop cheated and the loss function needs patching.
---

# LFD Design

You are designing an optimization target, not solving a task. The agent that
receives `goal.md` is a competent, tireless, literal optimizer: it will satisfy
the target by the cheapest available path — memorizing the eval, hardcoding
answers, mining feedback channels into lookup tables. Your job is to make
genuine capability the cheapest path left.

A spec says "build this, make the tests pass." A loss function says "build
this, make the tests pass, **then** descend toward this bar on data you cannot
see." You are writing the second thing. It has four parts: the **target**, the
**constraints**, the **instruments**, and the **forced entropy**. Every /goal
you emit must contain all four.

Two modes. **Design mode** (default): the phases below, in order. **Patch
mode** (see end): a running loop cheated; fix the loss function, not the agent.

## Phase 0 — Observe before asking

Inventory the environment BEFORE asking the user anything. The first principle
of harness engineering is observability — apply it to your own task:

- **Repo**: existing test suites, eval datasets, scoring scripts, CI
  workflows, logs/telemetry, CLAUDE.md / AGENTS.md.
- **Tooling**: what is installed and usable — Playwright/headless browsers,
  crawlers, image-diff tools, jq, database clients.
- **Surfaces**: which API keys exist in the environment or .env files (check
  presence only; never print values), which providers are reachable.
- **Reference artifact**: if the user named a product or dataset, look at
  what is publicly accessible right now.

Reuse what exists — extend an existing scorer or eval rather than generating
a parallel one. Whatever observation could not answer becomes Phase 1.

## Phase 1 — Interrogate

Ask the user in ONE batched round, only what Phase 0 couldn't answer:

1. **Outcome** — what artifact or behavior, and what does "good" look like?
   Is there a reference artifact to score against?
2. **Eval source and size** — where do ground-truth cases come from, and how
   many are obtainable? (Phase 3 can build the eval if the answer is "nowhere
   yet.")
3. **Budgets** — wall-clock budget for the run, dollar ceiling, and which
   paid surfaces exist (crawler credits, LLM keys). An 80% solution in 2
   hours beats a 100% one in 30 days; get the user's actual tolerance.
4. **Surface** — what the agent may touch: directories, APIs, providers,
   models, concurrency. Everything unlisted is denied.
5. **Acceptance** — the score bar, measured on held-out data only, plus a
   diminishing-returns stop ("if marginal gain ≈ 0 for N cycles, stop and
   report").

## Phase 2 — Spec: the inner loop

The spec is the starting point, not the finish line. Before designing any
optimization target:

- If a spec exists, read it. If not, generate one: reverse-engineer the
  reference artifact (public surfaces only) into a system design plus
  concrete test cases, and write it to `spec.md`.
- The spec's test suite is the **inner loop**: short horizon, fast feedback,
  one objective — make the tests pass. The eval is the **outer loop**: long
  horizon, sparse feedback.
- `goal.md` must gate the outer loop behind the inner one: Stage 0 = build
  to spec, tests green, before any descent on the eval. Never let the agent
  optimize a half-built system against sparse, slow feedback.

## Phase 3 — Build the eval

If the user cannot hand over enough cases, build them — for more and more
problems, real expected outputs are sitting in public:

- Collect real expected outputs from the reference artifact at scale.
  Public artifacts only; respect robots.txt, rate limits, and ToS.
- Dedup. Check diversity — no single entity, date range, or template may
  dominate, or the eval teaches a shortcut.
- Reject any case that overlaps seed or fixture data.
- Collection must be independent of the future optimizer: do it now, in this
  session, and land the answers outside the optimizer's surface.
- **Split**: `eval/dev` (scored freely, misses reported but capped) and
  `eval/holdout` (scored rarely, aggregate-only; acceptance measured here
  exclusively; answers outside the repo if at all possible).
- **Visibility rule, stated explicitly in goal.md**: eval INPUTS may be
  visible (probe generation needs them); eval ANSWERS are never readable —
  dev answers live only inside the scorer, holdout answers outside the repo.
- If the total is under ~200 cases, warn explicitly: a small eval is
  enumerable and the agent WILL memorize it. Widen before proceeding.

## Phase 4 — Design the loss function

**Target.**
- The metric must be mechanically computable by a script, at the right
  resolution for the claim. An LLM judge that "compares two screenshots"
  approves 12px spacing errors; a pixel-diff does not. Match the instrument
  to the precision the user actually wants.
- The metric must penalize BOTH failure directions. Recall without precision
  invites a return-everything cheat; precision without recall invites a
  return-one-thing cheat. If the user gives a one-sided metric, fix it and
  tell them why.
- Run a leak audit on every feedback channel: bits revealed per scoring call
  × expected number of cycles — can the agent reconstruct the eval before
  the run ends? If yes, cut feedback resolution (cap the miss list, return
  aggregates) or grow the set.

**Constraints.**
- Wall-clock budget, stated in the /goal. Agents have no sense of time and
  will grind 10 hours for 2%.
- Dollar and credit ceilings per paid surface.
- Surface allowlist from Phases 0–1.
- Methodology rules (LLM-in-the-data-plane allowed? deterministic only?).
- **Capacity caps** on every artifact that could function as a lookup table:
  keyword lists, regex sets, seed data, special-case branches. Name the
  artifact and the cap explicitly ("keyword list ≤ 20 entries").

**Enumerate the cheats.** Read `references/cheat-museum.md`, then list at
least 10 ways a lazy optimizer could max THIS metric without solving THIS
task. For each, write the fence: a constraint in `goal.md` AND a way to
detect violation. A constraint without an instrument is a vibe — the agent
will violate it cheerfully because it can't tell it's violating it.

**Enforcement design rule.** Any constraint that references eval content
(e.g. "no literal in the codebase may match an eval item") can only be
checked by the harness — the agent can't check it without reading the eval.
Put the check in `harness/lint.sh`, run it inside `score.sh`, and on
violation VOID the score and report nothing else. Naming the offending
literal turns your lint into a membership oracle the agent can mine
string-by-string (museum exhibit 12). Your enforcement instrument is itself
a feedback channel — leak-audit it like any other.

## Phase 5 — Generate the harness

Write these files now, tailored to the task. Do not ship placeholders.
Reuse anything Phase 0 found.

- `harness/score.sh` — the task-specific scorer. Pixel-diff for a UI clone
  (deterministic rendering: frozen time, animations off, pinned fonts,
  fixed viewport), recall@k + precision for retrieval, structured JSON diff
  for API behavior. Runs `lint.sh` first: any violation voids the score
  (output `VOID: constraint violation` and nothing more). Scores `eval/dev`
  by default; `--holdout` returns one aggregate number, rate-limited, and
  appends to an audit log.
- `harness/lint.sh` — checks capacity caps and eval-literal overlap. Called
  only by `score.sh`; its detailed findings go to a file outside the
  optimizer's read surface, for the human.
- `harness/probe.sh` — generates perturbed variants of dev INPUTS
  (paraphrases, date shifts, entity swaps) and reports the dev-vs-probe
  score gap. The gap is the memorization gauge.
- `harness/status.sh` — per-step timestamps and total wall-clock elapsed;
  spend so far AND projected burn before the next paid batch, per surface;
  score history per cycle; and the optimizer's own token consumption where
  session logs allow. Gain per token is the gradient of the optimization
  itself — the loop should be self-aware.
- `eval/dev/` and `eval/holdout/` — from Phase 3.
- `LOG.md` — instantiate `references/log-template.md`: one entry per cycle
  with `hypothesis / expected failure mode / diagnostic / result`, written
  before the change, not after. This is what survives context compaction.

## Phase 6 — Verify the harness yourself

Do this now, with your own tools. Do not delegate it to the user:

1. Run `score.sh` on dev — it must produce a number.
2. Calibrate: score one known-good and one known-bad output. The scorer
   must separate them decisively. A broken scorer optimizes noise.
3. Run `probe.sh` and `status.sh` once each.
4. Blinding check: from the optimizer's working directory, try to read the
   holdout answers. If you can, the agent can.
5. Trip the lint deliberately — plant an eval literal, confirm the score
   voids WITHOUT naming it, then remove the plant.

## Phase 7 — Red-team your draft

Before emitting, simulate the laziest possible agent against your draft
/goal: what is the five-minute win? Common ones: seed data that mirrors the
eval, mining per-item miss feedback into a keyword lookup table, gaming a
judge, editing the scorer or the goal itself, declaring victory on the dev
set. Patch the draft and simulate again. Emit only when three consecutive
simulations find nothing cheaper than doing the real work.

## Phase 8 — Emit goal.md

Fill the structure in `references/goal-template.md`. Every placeholder gets
a task-specific value; no section is dropped. Invariants the emitted goal.md
must keep regardless of task: the Stage 0 tests-green gate, VOID semantics,
holdout-only acceptance, the read-only set including goal.md itself, the
per-cycle checkpoint commit, the entropy rules, and the stop conditions.

## Phase 9 — Pre-flight (the two things only the human can do)

Everything else was verified in Phase 6. Tell the user:

1. Use a disposable API key with a provider-side spend limit.
2. **Babysit cycle 1.** Watch what the agent touches and confirm it uses
   the instruments. Then go to bed.

## Patch mode — when the loop cheats anyway

A cheat mid-run is a bug in the target, not the agent. When invoked against
a running or paused loop (the user reports a cheat, or LOG.md / a probe gap
shows one):

1. Read `LOG.md`, the score history, and the diff since the last honest
   checkpoint.
2. Identify the open path: which feedback channel leaked, which artifact
   had spare capacity, which constraint lacked an instrument.
3. Patch the **loss function**, not the agent's code: widen the eval, cap
   the artifact, cut feedback resolution, add the missing lint.
4. Append the exhibit to `references/cheat-museum.md` — what it looked
   like → the fence that closed it.
5. Re-verify the harness (Phase 6), revert eval-shaped artifacts the cheat
   produced, and resume the loop from the last honest checkpoint.
