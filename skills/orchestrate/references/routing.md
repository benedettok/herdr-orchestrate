# Worker routing

Use a **scout → analyst → author → reviewer** pipeline. The god session owns routing and decisions; workers never start nested workers.

## Routes

| Lane               | Route                         | Default thinking | Use for                                                                                           | Must not do                                                                 |
| ------------------ | ----------------------------- | ---------------- | ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Scout              | `explore` / Claude Haiku 4.5 | `high` (fixed)   | Read many files, locate relevant code, map symbols/callers/config, collect exact evidence         | Findings, severity, priorities, recommendations, trade-off decisions, edits |
| Analyst            | `fast` / Claude Sonnet 5.5         | `medium`         | Research, diagnosis, planning, synthesis, bounded technical recommendations                       | Consequential product/architecture/security decisions outside the brief     |
| Author / proof     | `fast` / Claude Sonnet 5.5         | `medium`         | Routine implementation, fixes, transformations, checks, and live proof                            | Silent escalation of scope or risk                                          |
| Decider / reviewer | `default` / Claude Opus 5.5       | `medium`         | Consequential ambiguity, architecture, security, difficult debugging, and quality-critical review | Repeating bounded work that Sonnet can complete                               |

`medium` is the recommended default for both coding routes, including Opus. Use `low` only for trivial bounded work. Use `high` only after a medium pass produces concrete evidence that more reasoning is required, or when the user explicitly asks for it. Record the escalation reason in the worker brief. Reserve `xhigh` for explicit user escalation.

## Scout → analyst

Use a scout before an analyst when the task is input-heavy and the decision boundary is already clear.

1. Split file discovery by independent evidence source or subsystem.
2. Spawn bounded Haiku scouts. Their reports contain only relevant files, relationships, exact evidence, and unknowns.
3. Spawn one Sonnet analyst with the scout report paths. Sonnet verifies decisive evidence and makes the recommendation.
4. Escalate to Opus medium only when the remaining choice is consequential or Sonnet reports evidence-backed uncertainty.

A Sonnet worker does not call Haiku directly. If Sonnet discovers that broad file mapping is missing, it reports:

```md
## Scout request

- Question: <bounded factual question>
- Scope: <directories, symbols, or evidence sources>
- Needed evidence: <facts required before analysis can continue>
```

and ends with `orch-verdict: blocked scout evidence needed`. The god session spawns the scout, then sends the resulting report path back to the same Sonnet worker.

## Scout report contract

A scout report is complete only when it includes:

- every requested scope mapped to relevant files and symbols;
- caller/import/config relationships supported by paths and line references;
- exact commands or documents that define the observed behavior;
- unknowns that require an analyst or user decision;
- no findings, severity labels, priorities, recommendations, fix sketches, or preferred options.
