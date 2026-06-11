# LOG.md template

The iteration log is what survives context compaction: the optimizer reads
it back to reflect across cycles, and the human reads it to audit the run.
Hypothesis, expected failure mode, and diagnostic are written BEFORE the
change — a hypothesis written after the result is a rationalization.

```markdown
# Iteration Log — <goal one-liner>

Started: <timestamp> · Budgets: <hours> wall-clock / <$> spend

## Cycle <n> — <timestamp>
- Score (dev): <score> (prev: <score>) · Probe gap: <gap>
- Hypothesis: <what change should move the metric, and why>
- Expected failure mode: <how this change could fail or turn into a cheat>
- Diagnostic: <what observation distinguishes success from the failure mode>
- Change: <summary> (commit <hash>)
- Result: <score after; hypothesis confirmed/refuted; what was learned>
- Reflection: <generalizing or memorizing? if memorizing, which
  eval-shaped artifact gets removed next cycle>

## Final report
- Best holdout score:
- What generalized:
- What was abandoned (and why):
- Highest-leverage next steps:
```
