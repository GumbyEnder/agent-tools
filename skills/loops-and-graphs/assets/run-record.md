# Run record

Write one record per run, where the next run can read it. A record that lives only in the chat does not count.

```yaml
job: ""                       # stable job name
graph:
  name: ""                    # path of the fixed graph, or "throwaway"
  version: ""                 # git sha of the graph, or "unversioned"
nodes:
  - name: ""
    check: ""                 # the command, schema, or checklist
    evidence: STRONG          # STRONG | MEDIUM | WEAK | NOISE
    result: pass              # pass | fail | skipped
    rounds: 0
    action: none              # none | RETRY | FALLBACK | SKIP | REPAIR | ESCALATE | STOP
repairs:
  - unit: ""
    test: ""
    evidence: ""
    scope: ""
    passed: false
cost:
  tokens: 0
  usd: 0
  wall_time: ""               # e.g. 12m
approver: ""                  # human handle, or "auto"
approvals: 0                  # human approvals this run; trend toward 1
escalated: false
reverted: false               # set true later if a closed-lane action is undone
```

Rollback rate is closed-lane records with `reverted: true`, divided by closed-lane records. Escalation rate is records with `escalated: true`, divided by records. Read both with approvals per run.

## Example

```yaml
job: export-timeout
graph:
  name: graphs/export-timeout.md
  version: 4f2c1ab
nodes:
  - name: plan
    check: plan.files non-empty and plan.timeout_ms is an integer
    evidence: STRONG
    result: pass
    rounds: 1
    action: none
  - name: export-timeout
    check: pytest tests/test_export_timeout.py exits 0; diff ⊆ plan.files
    evidence: STRONG
    result: pass
    rounds: 2
    action: REPAIR
repairs:
  - unit: export-timeout
    test: pytest tests/test_export_timeout.py
    evidence: test exited 1; assertion expected timeout at 500ms
    scope: src/export.py, tests/test_export_timeout.py
    passed: true
cost:
  tokens: 84000
  usd: 0.42
  wall_time: 11m
approver: auto
approvals: 0
escalated: false
reverted: false
```

This job sits on the open lane, so the approver is `auto` and human approvals stay 0. A closed step (a migration, deletion, production write, payment, or deploy) records the human as `approver` and `approvals: 1`.
