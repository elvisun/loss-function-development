# Example output: goal.md for a news-retrieval data layer

> For human readers. Kept outside `skills/lfd-design/` deliberately — an
> example bundled with the skill would anchor generated loss functions to
> this task's shape.

This is a condensed example of what `/lfd-design` emits — a /goal for
building a news search/retrieval layer scored against a reference product,
modeled on a real 30-hour run (92k pages crawled, ~$40 in tokens, final
output ~50× the reference on the same queries). The eval was built by
mining the reference product's public outputs; the spec was
reverse-engineered from its public surface in a prior session.

---

# Goal: News-intel retrieval layer matching reference-product quality

## Stage 0 — Build to spec (inner loop)

Implement `spec.md` (crawler → normalizer → index → ranker). Make the test
suite pass (`npm test`). Do not score against the eval until tests are
green. Tests stay green every cycle thereafter.

## Target (outer loop)

For each query in the eval, the system returns a ranked list of source
articles. Score = harmonic mean of recall@50 and precision@50 against
ground truth (`harness/score.sh`). Bar: **0.85 on holdout**.

- A VOID result means a constraint was violated — find and remove the
  violation; the harness will not tell you which it was.
- `eval/dev` (160 queries): score freely; misses reported, capped at 10
  per call. Query inputs are visible; answers live inside the scorer only.
- `eval/holdout` (60 queries, answers outside this repo): aggregate score
  only, max 1 call per 6 wall-clock hours, audit-logged. Acceptance is
  measured here exclusively.

## Constraints

- Wall-clock budget: 36 hours. Check `harness/status.sh` every cycle — it
  shows elapsed and per-step time, crawler credits spent and projected
  burn before the next batch, LLM spend, and your own token consumption.
  Watch gain per token; a flat gradient at high burn means stop.
- Spend: crawler ≤ 100k credits total; LLM data-plane spend ≤ $50 on the
  provided key.
- Surface: this repo, the crawler API, the LLM key. Nothing else.
- Methodology: LLM analysis allowed in the data plane; ranking itself must
  be deterministic and reproducible.
- Capacity caps: any keyword/expansion list ≤ 20 entries; no
  query-conditional branches keyed to specific inputs.
- `goal.md`, `harness/`, and `eval/` are read-only. Eval query inputs may
  be read where the harness exposes them; eval answers never.

## Cycle protocol

1. Score on dev.
2. Reflect: run `harness/probe.sh` (paraphrased + date-shifted query
   variants). If the dev-vs-probe gap grew, you are memorizing — the next
   change must REMOVE an eval-shaped artifact, never add one.
3. Hypothesize in `LOG.md` before touching code: hypothesis, expected
   failure mode, the diagnostic that will distinguish them.
4. Change.
5. Log the result against the hypothesis.
6. Checkpoint: `git commit -am "cycle <n>: <score>"` — every cycle, gain
   or no gain, so the run is bisectable and crash-safe.

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
