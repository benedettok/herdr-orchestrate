# Worker routing

Use a **scout → analyst → author → reviewer** pipeline. The god session owns routing and decisions; workers never start nested workers.

Every worker runs on Claude Opus 5.5: spawn each one with `--route default` and an explicit `--thinking` of at least `low`. The CLI still accepts the `fast` (Sonnet) and `explore` (Haiku) routes; do not use them. The lanes below differ in role, brief and thinking level, not in model.

## Lanes

| Lane               | Route     | Default thinking | Use for                                                                                           | Must not do                                                                 |
| ------------------ | --------- | ---------------- | ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Scout              | `default` | `low`            | Read many files, locate relevant code, map symbols/callers/config, collect exact evidence         | Findings, severity, priorities, recommendations, trade-off decisions, edits |
| Analyst            | `default` | `medium`         | Research, diagnosis, planning, synthesis, bounded technical recommendations                       | Consequential product/architecture/security decisions outside the brief     |
| Author / proof     | `default` | `medium`         | Implementation, fixes, transformations, checks, and live proof                                    | Silent escalation of scope or risk                                          |
| Decider / reviewer | `default` | `medium`         | Consequential ambiguity, architecture, security, difficult debugging, and quality-critical review | Repeating bounded work an author or analyst can complete                    |

`medium` is the default for analysts, authors, proof and reviewers. Use `low` for scouts and trivial bounded work. Use `high` when a medium pass produces concrete evidence that more reasoning is required, for a design question with several viable shapes, or when the user explicitly asks for it. Record the reason in the worker brief. Reserve `xhigh` for explicit user escalation.

## Scout → analyst

Use a scout before an analyst when the task is input-heavy and the decision boundary is already clear.

1. Split file discovery by independent evidence source or subsystem.
2. Spawn bounded scouts (`--route default --thinking low`). Their reports contain only relevant files, relationships, exact evidence, and unknowns.
3. Spawn one analyst with the scout report paths. The analyst verifies decisive evidence and makes the recommendation.
4. Raise the analyst's or a decider's thinking only when the remaining choice is consequential or the analyst reports evidence-backed uncertainty.

An analyst or author does not start a scout itself. If it discovers that broad file mapping is missing, it reports:

```md
## Scout request

- Question: <bounded factual question>
- Scope: <directories, symbols, or evidence sources>
- Needed evidence: <facts required before analysis can continue>
```

and ends with `orch-verdict: blocked scout evidence needed`. The god session spawns the scout, then sends the resulting report path back to the same worker.

## Scout report contract

A scout report is complete only when it includes:

- every requested scope mapped to relevant files and symbols;
- caller/import/config relationships supported by paths and line references;
- exact commands or documents that define the observed behavior;
- unknowns that require an analyst or user decision;
- no findings, severity labels, priorities, recommendations, fix sketches, or preferred options.
