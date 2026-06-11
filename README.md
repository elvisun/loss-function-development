# A skill for loss-function-development

A skill that designs loss functions for long-running autonomous agent loops.

Spec-driven development says: *build this, make the tests pass.*
Loss-function development (LFD) says: *build this, make the tests pass, then
iterate against a thousand eval cases you can't see.* A test suite is finite —
done the moment it's green. A loss function is a target you descend toward.
The spec becomes the starting point, not the finish line.

The catch: the agent in the loop is an optimizer, and every cheap path you
don't fence off is a direction it will sprint down. It will memorize your
eval, mine your miss-lists into lookup tables, and game your judge — not
because it's broken, but because you told it where to go and left the
shortcuts open.

`/lfd-design` is the meta-meta-prompt, packaged as a skill. It doesn't solve
your task — it observes what harness and tooling you already have, ingests or
reverse-engineers the spec, builds the eval (often by mining public
artifacts), designs the target, generates **and verifies** the harness,
red-teams its own draft for cheats, and emits a `goal.md` ready to launch.
When the loop cheats anyway mid-run, re-invoke it in **patch mode**: it reads
the iteration log, closes the open path in the loss function (not the agent's
code), and resumes from the last honest checkpoint.

## What it produces

```
goal.md              the loss function: spec gate (inner loop), target,
                     constraints, cycle protocol, entropy rules, stop conditions
spec.md              the inner loop: system design + test cases (ingested,
                     or reverse-engineered from the reference artifact)
harness/score.sh     task-specific scorer (pixel-diff, recall+precision,
                     schema diff — generated for your task, not generic)
harness/lint.sh      capacity-cap and eval-overlap checks; violations VOID
                     the score and report nothing else
harness/probe.sh     perturbed eval variants; the dev-vs-probe gap is the
                     memorization gauge
harness/status.sh    per-step wall-clock, score history, spend + projected
                     burn, the agent's own token consumption
eval/dev/            scored freely during the run (inputs visible,
                     answers blinded)
eval/holdout/        scored rarely, aggregate-only; acceptance lives here
LOG.md               iteration log: hypothesis / expected failure /
                     diagnostic / result
```

## The four parts of a loss function

1. **Target** — large enough that enumeration doesn't pay, blinded during
   the run, measured by a mechanical instrument at the right resolution.
2. **Constraints** — wall-clock, money, surface, methodology, and capacity
   caps on anything that could become a lookup table.
3. **Instruments** — one command per constraint. A constraint without an
   instrument is a vibe; the agent will violate it cheerfully because it
   can't tell it's violating it.
4. **Forced entropy** — overfit reflection every cycle, a stall rule that
   bans same-knob-harder, an exploration quota, and a log that survives
   compaction. Local maxima are the default state of a loop.

## Install

Copy the skill into your skills directory:

```bash
# personal (all projects)
cp -r skills/lfd-design ~/.claude/skills/

# or per-project
cp -r skills/lfd-design <your-repo>/.claude/skills/
```

Then in Claude Code:

```
/lfd-design build a news-retrieval layer that matches <reference product>
```

Answer the interrogation round, review the generated `goal.md`, run the
pre-flight checklist, and launch your loop against it.

It also works with any agent that reads markdown instructions — point your
harness at `skills/lfd-design/SKILL.md`.

## Layout

```
skills/lfd-design/
├── SKILL.md                     the skill (the meta-meta-prompt)
└── references/
    ├── cheat-museum.md          observed cheats and the fences that closed them
    ├── goal-template.md         the structure every emitted goal.md must fill
    └── log-template.md          the iteration log that survives compaction
```

There is deliberately no example goal.md in this repo: a worked example
would anchor every generated loss function to one task's shape — the skill
would overfit to its own example. The templates define structure only; the
content is designed per task.

## Contributing

The highest-value contribution is a new exhibit in the
[cheat museum](skills/lfd-design/references/cheat-museum.md): a way an agent
satisfied your target without solving your task, and the fence that closed
it. Every entry makes the skill's red-team phase smarter.

## Rules of engagement

LFD is often used to distill publicly shipped artifacts — outputs a company
publishes in the open. That has always been fair to learn from. ToS-gated,
login-walled, or paid outputs are not. Respect robots.txt, rate limits, and
spend caps; the skill bakes these into every goal it emits.

## License

[MIT](LICENSE)
