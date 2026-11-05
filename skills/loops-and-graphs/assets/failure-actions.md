# Failure actions

A failed check takes exactly one row. Send that unit back. Never send the whole batch back. Units that already passed stay passed.

| Action | When | What to do |
| --- | --- | --- |
| RETRY | Transient failure: timeout, rate limit, flaky tool. Same inputs. | Run the unit again. It counts toward the round cap. |
| FALLBACK | The preferred path is down, and a lesser path still meets the check. | Switch path. Record which path ran. |
| SKIP | The unit is optional, and the join can finish without it. | Mark it skipped with a reason. Do not invent the output. |
| REPAIR | The unit is wrong and the check shows why. | Repair this unit only. Pass the failing test, the evidence, and a scope line (files, rows, or claims in bounds). |
| ESCALATE | Three repairs on this unit have failed. | The plan is wrong. The loop cannot see the plan. Send it to a coordinator and a human. |
| STOP | The budget or a permission limit is hit. | Halt the graph. Wake a human. Do not spend past the limit. |

Repair handoff, one unit:

```
UNIT:
TEST:
EVIDENCE:
SCOPE:
ATTEMPT: <n of cap>
```

Escalation line, after the third failed repair or on a stall:

```
JOB: <job> | BLOCKED: <what is stuck> | NEED: <the decision>
```

Confirm the page was delivered before you treat it as sent.
