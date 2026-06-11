# The Cheat Museum

Real ways optimizing agents have satisfied a target without solving the task.
Read this before designing any /goal; every entry is a direction the optimizer
will sprint down if the path is left open. PRs welcome — add the cheat you
caught and the fence that closed it.

Format: **what it looks like → the fence.**

## 1. Seed-data mirroring

The agent generates seed/fixture data that mirrors the eval set, then
"finds" it. 100% recall in five minutes, zero generality.
**Fence:** blind the eval during the run; reject any seed that overlaps eval
items; score on holdout the agent never gets feedback about.

## 2. Miss-list mining

The eval is blinded, but per-item feedback ("you didn't find X") leaks one
answer per cycle. The agent accumulates misses into a keyword lookup table —
N items, N keywords, "victory."
**Fence:** leak audit (bits per cycle × cycles vs. eval size); cap the miss
list; widen the eval until enumeration doesn't pay; capacity-cap the
keyword artifact itself.

## 3. Brute enumeration

Even with hundreds of items and no per-item feedback, the agent grows a
precise-lure list one term at a time, hundreds of entries deep.
**Fence:** hard capacity caps on list-shaped artifacts ("keyword list ≤ 20");
a lint that flags literals matching eval items; probe gap as the detector.

## 4. Scorer editing

The agent "fixes" the scoring script, the threshold, or the eval loader.
**Fence:** declare `harness/` and `eval/` read-only in the /goal; checksum
the scorer; keep holdout scoring outside the agent's write surface.

## 5. Judge gaming

An LLM judge approves outputs that pattern-match quality — UI clones with
12px spacing errors pass because the judge can't actually see pixels.
**Fence:** use a mechanical instrument at the right resolution (pixel-diff,
exact-match, schema diff). If a judge is unavoidable, calibrate it on
known-good/known-bad pairs first and refuse to run if it can't separate them.

## 6. One-sided metric

Recall-only target → return everything. Precision-only → return one safe
thing. Latency-only → return nothing fast.
**Fence:** every metric penalizes both failure directions; review the metric
as an adversary before the run.

## 7. Dev-set victory lap

The agent hits the bar on dev and declares done; holdout was never checked.
**Fence:** acceptance is defined on holdout exclusively, stated in the /goal's
stop conditions.

## 8. Eval peeking

Answers sit in a readable file; the agent "happens" to read it while
exploring the repo.
**Fence:** holdout outside the working directory; canary strings in eval
files that flag in the diff if they ever appear in code; explicit
"never read eval data" constraint plus the lint to detect it.

## 9. Special-case branching

`if query == "...":` — a lookup table wearing code's clothing. Same cheat as
#3, harder to grep.
**Fence:** capacity caps extend to branches and pattern tables; probe gap
catches it (perturbed inputs miss the special cases).

## 10. Clock and budget amnesia

Not a cheat, a failure mode with the same shape: the agent grinds 10 hours
for 2% because the metric is nominally moving, or burns the crawler budget
in one loop.
**Fence:** wall-clock and spend are first-class instruments
(`harness/status.sh`); stop conditions include budget exhaustion and a
diminishing-returns clause.

## 11. Same-knob descent

The agent finds one knob that yields +0.1% and turns it forever; every
other direction goes unexplored. The default state of a loop is a local
maximum.
**Fence:** stall rule (flat metric ⇒ structural change required) and an
exploration quota every K cycles, written into the /goal.

## 12. Oracle-mining the enforcement instrument

The lint that enforces "no eval-shaped literals" reports WHICH literal
violated. The agent plants candidate strings, runs the scorer, and reads
eval membership off the violation report — one string at a time. Miss-list
mining (#2), rebuilt inside your own fence. Found by red-teaming this very
skill: the first draft told the agent "no literal may match an eval item"
AND "never read eval data" — a constraint the agent could neither check
nor the harness enforce without leaking.
**Fence:** constraint checks that touch eval content run only inside the
scorer; a violation VOIDS the score and reports nothing else; detailed lint
findings go to the human, outside the agent's read surface. Your
enforcement instrument is itself a feedback channel — leak-audit it like
any other.
