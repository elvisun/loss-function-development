# goal.md template

Fill every placeholder; drop no section. Each section maps to one of the
four parts of a loss function: Stage 0 + Target (target), Constraints
(constraints + instruments), Cycle protocol (instruments), Entropy rules and
Stop conditions (forced entropy).

```markdown
# Goal: <one-line outcome>

## Stage 0 — Build to spec (inner loop)
Implement spec.md. Make the test suite pass. Do not score against the eval
until tests are green. Tests stay green every cycle thereafter.

## Target (outer loop)
<metric definition, both directions> · Bar: <score> on holdout.
Score with `harness/score.sh`. A VOID result means a constraint was
violated — find and remove the violation; the harness will not tell you
which it was. Holdout: aggregate-only, max <N> calls per <period>.
Acceptance is measured on holdout exclusively.

## Constraints
- Wall-clock budget: <hours>. Check `harness/status.sh` every cycle — it
  shows elapsed and per-step time, projected spend, and your own token
  burn. Watch gain per token; a flat gradient at high burn means stop.
- Spend ceilings: <per surface>.
- Surface: <allowlist>. Everything else is off-limits.
- Capacity caps: <artifact ≤ N>.
- `goal.md`, `harness/`, and `eval/` are read-only. Eval inputs may be read
  where the harness exposes them; eval answers never.

## Cycle protocol
1. Score (dev). 2. Reflect: run `harness/probe.sh` — am I generalizing or
memorizing? If the probe gap is growing, the next change must REMOVE an
eval-shaped artifact (cap a list, blind a feature, reject a seed), never
add one. 3. Hypothesize: log hypothesis, expected failure mode, and
diagnostic in LOG.md BEFORE changing code. 4. Change. 5. Log the result.
6. Checkpoint: `git commit -am "cycle <n>: <score>"` — every cycle, gain
or no gain, so the run is bisectable and crash-safe.

## Entropy rules
- Stall rule: if the metric didn't move last cycle, the next attempt must
  be a structural change — same-knob-harder is banned.
- Exploration quota: every <K> cycles, try a structurally different
  approach even if the current one is still inching up.

## Stop conditions
Bar hit on holdout · any budget exhausted · marginal gain ≈ 0 for
<N> consecutive cycles. On stop: write a final report in LOG.md — best
score, what generalized, what was abandoned, highest-leverage next steps.
```
