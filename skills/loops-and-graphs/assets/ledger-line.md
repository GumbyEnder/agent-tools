# Ledger line

Append one line when a repair passes. Read matching lines before the next plan.

```
DATE | task | cause | RULE
```

```
2026-11-05 | export timeout | test mocked sleep, so the timeout never fired | RULE: timeout tests use a fake clock the production code reads
```

```
2026-11-05 | export timeout | sleep-mock helper was removed, so the rule can no longer fire | RULE: PRUNE timeout tests use a fake clock the production code reads
```

- `DATE` is the day the repair passed, `YYYY-MM-DD`.
- `task` is the job name, stable across runs.
- `cause` is the observed reason, one clause.
- `RULE` is the instruction the next planner must follow. Point it at a check when you can.

Store the ledger in the repo, next to the fixed graph, for a recurring job.

Prune on a fixed cadence. Monthly is enough. Delete a rule when its check is gone, or when the cause can no longer happen. Log the deletion with `PRUNE` so the deletion itself has a record.
