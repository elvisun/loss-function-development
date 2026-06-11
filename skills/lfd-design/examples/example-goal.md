# Example output: goal.md for a news-retrieval data layer

This is a condensed example of what `/lfd-design` emits — a /goal for
building a news search/retrieval layer scored against a reference product,
modeled on a real 30-hour run (92k pages crawled, ~$40 in tokens, final
output ~50× the reference on the same queries).

---

# Goal: News-intel retrieval layer matching reference-product quality

## Target

For each query in the eval, the system returns a ranked list of source
articles. Score = harmonic mean of recall@50 and precision@50 against
ground truth (`harness/score.sh`). Bar: **0.85 on holdout**.

- `eval/dev` (160 queries): score freely; misses reported, capped at 10
  per call.
- `eval/holdout` (60 queries, answers outside this repo): aggregate score
  only, max 1 call per 6 wall-clock hours, audit-logged. Acceptance is
  measured here exclusively.

## Constraints

- Wall-clock budget: 36 hours. Check `harness/status.sh` every cycle.
- Spend: crawler ≤ 100k credits total; LLM data-plane spend ≤ $50 on the
  provided key. Track both via `harness/status.sh`.
- Surface: this repo, the crawler API, the LLM key. Nothing else.
- Methodology: LLM analysis allowed in the data plane; ranking itself must
  be deterministic and reproducible.
- Capacity caps: any keyword/expansion list ≤ 20 entries; no literal string
  anywhere in the codebase may match an eval query or eval answer; no
  query-conditional branches keyed to specific inputs.
- `harness/` and `eval/` are read-only. Never read eval answer data.

## Cycle protocol

1. Score on dev.
2. Reflect: run `harness/probe.sh` (paraphrased + date-shifted query
   variants). If the dev-vs-probe gap grew, you are memorizing — the next
   change must REMOVE an eval-shaped artifact, never add one.
3. Hypothesize in `LOG.md` before touching code: hypothesis, expected
   failure mode, the diagnostic that will distinguish them.
4. Change.
5. Log the result against the hypothesis.

## Entropy rules

- Stall rule: if dev score moved < 0.5 points last cycle, the next attempt
  must be structurally different (new retrieval strategy, new data source,
  new index shape) — re-tuning the same parameter is banned.
- Exploration quota: every 5 cycles, spend one cycle on an approach you
  have not tried, even if the current line is still improving.

## Stop conditions

- 0.85 on holdout, or
- any budget exhausted (wall-clock, crawler credits, LLM spend), or
- marginal dev gain < 0.2 points for 3 consecutive cycles.

On stop: write a final report in `LOG.md` — best score, what generalized,
what was abandoned, and the three highest-leverage next steps.
