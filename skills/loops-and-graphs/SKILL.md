---
name: loops-and-graphs
description: >-
  Design and run multi-agent work so a human approves only the final
  irreversible step. Use when splitting a job across a fleet (Claude Code,
  Codex, LangGraph, Hermes, or custom), drawing loops into a graph, writing
  a check before a task, choosing a retry or repair, escalating a stuck
  unit, or holding a migration, deletion, production write, payment, or
  deploy for a person.
metadata:
  author: Shane Robinett (gumbyender)
---

# Loops and graphs

Credits: Shane Robinett (gumbyender). Inspired by "Loops & Graphs" by [@HoodieFade_](https://x.com/HoodieFade_/status/2117959235690524883).

Use this to design and run multi-agent work so a person signs only the last step that cannot be undone. It fits Claude Code, Codex, LangGraph, Hermes, and custom fleets. The runner can change. The contracts below stay the same.

Express the graph however the runner requires: a markdown file of node contracts, a graph library, a workflow config, or a script. Keep the fields identical.

## Run a job

1. Read the ledger lines that match this job. See [Turn a repair into a rule](#turn-a-repair-into-a-rule).
2. Write a check for every unit before any worker starts. No check means the unit is not ready.
3. Decide who draws the graph. See [Who draws the graph](#who-draws-the-graph).
4. Fill one [node contract](assets/task-contract.md) per node. Name the variable on every edge.
5. Set each lane from reversibility. See [Approval lanes](#approval-lanes).
6. Run each unit in its own loop until a [stop rule](#when-a-loop-stops) fires.
7. On a failed check, take one action from the [failure table](assets/failure-actions.md). Send that unit back. Leave passing units alone.
8. On escalation, page a coordinator and a human. Confirm the page was delivered.
9. When a repair passes, append a ledger line.
10. Write the [run record](assets/run-record.md) before the job is done.

## Loop and graph

A loop makes one unit of work correct. Inside the unit the worker produces, a program checks, and a repair runs until the check passes or a stop rule fires.

A graph decides which units exist, which order they run in, and where a human signs off.

You need both. A loop with no graph leaves a person choosing the next prompt. A graph with no loop ships units no check can fail.

## Checks

Write the check before the task. If you cannot state a check a program can fail, split the unit or define the check. Do not start the worker.

Rank the evidence the check used:

| Rank | Counts as evidence |
| --- | --- |
| STRONG | Tests exit 0. Output matches a schema. The diff stays inside the plan. |
| MEDIUM | A cited source supports the claim. An independent path agrees. |
| WEAK | A reviewer agent approves. |
| NOISE | "Looks good." Model confidence. No errors were thrown. |

NOISE never passes a check. WEAK never satisfies the evidence lane on its own. Label the rank on the node result in the run record.

### Checks by work type

**Code.** A test command and a scope check.

```
cmd: <test command>
pass: exit code 0
scope: changed paths ⊆ planned paths
rank: STRONG
fail: nonzero exit, or any path outside the plan
```

**Research.** One block per claim. The check fails when the URL does not contain the quote, or the quote does not support the claim.

```
claim: <one sentence>
url: <source>
quote: <exact passage>
pass: quote appears at url AND supports the claim
rank: MEDIUM when a second independent source agrees; otherwise fail
```

A model summary of a page is NOISE. Fetch the page and match the quote.

**Content and docs.** A checklist a script or a second pass can tick item by item. One open item fails the unit.

```
checklist:
- [ ] required sections present
- [ ] links resolve
- [ ] no placeholder text
- [ ] claims that need a source have a citation
pass: every item checked
rank: MEDIUM
```

**Infra.** A read-only health check. The probe must not write, delete, restart, or deploy.

```
probe: <read-only command>
expect: <status, key, or metric>
pass: output matches expect
forbid: any mutation
rank: STRONG
```

Put the change that would mutate production on a closed-lane node of its own.

**Data.** Schema, keys, and bounds. State the tolerance in the contract.

```
schema: <columns and types>
keys: <unique key>
bounds: row count within <tolerance>, null rate ≤ <limit>
pass: schema, keys, and bounds all hold
rank: STRONG
```

## When a loop stops

Each unit loops on its own. Stop the unit when any of these is true:

- The check passes.
- The round cap is hit. Default cap is 5. Write a different cap on the contract when you need one. Hitting the cap is STOP: wake a human.
- The unit budget is spent. Pick one budget and write it down: tokens, money, or wall time. A spent budget is STOP: wake a human.
- Open-ended work (research, review, audit) goes 2 rounds with no new finding after dedupe. Ship the findings you have if the check allows it. Escalate if the check is still failing.
- The unit stalls: 3 rounds with the same failure and no new evidence. Page a human. Do not spend the rest of the cap.

Keep a memory of attempts. Store the output, the check result, and every finding you rejected. A later round must not file a rejected finding as new. Dedupe findings in code, by a stable key, before the "no new findings" test.

```
cap = 5
repairs = 0
stale = 0
state = {rounds: 0, attempts: [], rejected: set(), findings: set()}

while state.rounds < cap and budget_left():
    state.rounds += 1
    output = produce(unit, state)
    result = check(output)                 # program: exit code, schema, or scope
    state.attempts.append(result)

    if result.id in state.rejected:
        continue                               # already thrown out

    if open_ended:
        fresh = dedupe(result.findings) - state.findings
        if not fresh:
            stale += 1
        else:
            stale = 0
            state.findings |= fresh
        if stale >= 2:
            if result.passed:
                return state.findings          # ship the deduped set
            return escalate(unit, state)
        if result.passed:
            continue                           # keep looking

    elif result.passed:
        return output

    if repairs >= 3:
        return escalate(unit, state)           # the plan is wrong

    if stalled(state.attempts):                # same failure, no new evidence
        return page(coordinator, human, unit, result)

    repair(unit, result.test, result.evidence, result.scope)
    repairs += 1

return stop(unit, state)                   # cap or budget; wake a human
```

## Node contract

Every node is one bounded job. One input set, one output. Copy the block from [assets/task-contract.md](assets/task-contract.md) and fill it before the node runs.

```
NODE:
JOB:
IN:
OUT:
FAILS AS:
CHECK:
LANE:
```

- `IN` names each variable and the node that produced it.
- `OUT` names the variable and its shape: fields, type, or file set.
- `FAILS AS` describes a failed check in observable terms.
- `CHECK` is the program from [Checks](#checks). Write it first.
- `LANE` is `open`, `evidence`, or `closed`.

Use a model only to judge, decide, or summarize. Use code to merge, dedupe, sort, and join. If the step is one of those four, do not call a model.

Size the model to the job. Use the smallest model that can make the decision. Spend a larger model on a judgment call the run record shows the smaller one gets wrong. Mechanical steps get no model at all.

## Drawing the graph

Every edge carries one named variable: an output of the upstream node that the downstream node reads. Write that name on the edge. If you cannot name the variable, delete the edge. Nodes with no edge between them run at the same time.

Parallel workers get separate contexts. Each one starts from the variables on its own edges. It does not read a sibling's notes or scratchpad.

Join only when a later node needs the full set. The join waits for every input it lists, then merges in code. A node that needs one input gets one edge.

Use the smallest of these four shapes.

**Chain.** One node reads the previous node's output.

```
plan --spec--> implement --diff--> test
```

**Fan-out / join.** Workers share no edge. Each edge is still named: it carries that worker's slice.

```
                 --area--> worker_a --findings-->
split --areas-->                               join
                 --area--> worker_b --findings-->
```

**Router.** One node reads an input and picks exactly one next node. The choice is a decision, so a model may make it. Code does the handoff. The chosen node keeps its own check.

```
intake --kind--> router --route--> {code | research | docs}
```

**Bounded cycle.** A node may send work back to an earlier node. Treat the cycle as a loop: put the round cap, the budget, and the attempt memory on it. A cycle with no cap is a bug.

```
draft --doc--> check --failures--> repair --doc--> check
cap: 5
```

Draw a recurring graph once and check it in. See the next section.

## Who draws the graph

Choose the drawer from how often the job runs and how hard it is to undo.

| How often | How hard to undo | Who draws it |
| --- | --- | --- |
| One-off | Low risk | The model generates a throwaway graph. Discard it after the run. |
| One-off | Risky | The model drafts the plan. A human reads it before any node runs. |
| Recurring | Low risk | A fixed graph, checked into the repo. |
| Recurring | Touches money or users | A fixed graph in the repo. A human signs the last step. |

Never regenerate a recurring graph on every run. A graph that changes every run is a new plan, and the history you are collecting no longer applies to it. Change a fixed graph in a review, the same way you change code.

## When a check fails

Take exactly one action. Return the failing unit only. Never send the whole batch back. Units that already passed stay passed.

| Action | When | What to do |
| --- | --- | --- |
| RETRY | Transient failure: timeout, rate limit, flaky tool. Same inputs. | Run the unit again. It counts toward the round cap. |
| FALLBACK | The preferred path is down, and a lesser path still meets the check. | Switch path. Record which path ran. |
| SKIP | The unit is optional, and the join can finish without it. | Mark it skipped with a reason. Do not invent the output. |
| REPAIR | The unit is wrong and the check shows why. | Repair this unit only. Hand the repair the failing test, the evidence, and a scope line (files, rows, or claims in bounds). |
| ESCALATE | Three repairs on this unit have failed. | The plan is wrong. The loop cannot see the plan. Send it up. |
| STOP | The budget or a permission limit is hit. | Halt the graph. Wake a human. Do not spend past the limit. |

Copy of the table: [assets/failure-actions.md](assets/failure-actions.md).

After three failed repairs, ESCALATE. Do not start a fourth repair on the same plan.

## Approval lanes

Pick the lane by how hard the step is to undo. Model confidence does not move a step to a lower lane.

| Lane | What belongs here | Who approves |
| --- | --- | --- |
| open | Easy to undo. Drafts, local formatting, branch commits you can reset. | Auto-merge when the check passes. |
| evidence | Recoverable, but shared. Merges to a shared branch, published drafts, config with a rollback. | STRONG evidence, plus a coordinator approval. |
| closed | Migrations, deletions, production writes, payments, deploys. | A human, and only a human. |

Weigh evidence in this order. An item lower on the list does not outrank one above it.

1. Tests
2. Run history
3. Rollback rate
4. Reviewer agent
5. Model claim

A model claim is last. It does not open the evidence lane by itself. Closed stays closed at every rank.

Compute rollback rate from run records: closed-lane actions later marked reverted, divided by closed-lane actions.

## Escalation

Address an escalation to a coordinator and a human. One line:

```
JOB: <job> | BLOCKED: <what is stuck> | NEED: <the decision>
```

```
JOB: export-timeout | BLOCKED: schema check failed after 3 repairs | NEED: widen the plan or drop the field
```

Send that line to both. Then verify delivery: a message id, an ack, or a read on the channel you used. A page you cannot confirm was not delivered. Try one backup channel. If that is also unconfirmed, STOP.

## Turn a repair into a rule

When a repair passes, append one ledger line before the next plan:

```
DATE | task | cause | RULE
```

```
2026-11-05 | export timeout | test mocked sleep, so the timeout never fired | RULE: timeout tests use a fake clock the production code reads
```

Template: [assets/ledger-line.md](assets/ledger-line.md).

Store the ledger where the next planner will read it. For a recurring job, that is the repo, next to the fixed graph.

Read the lines whose task matches the job before you draw or run. A rule you do not read cannot fire.

Prune on a fixed cadence. A monthly pass is enough. Delete a rule when its check no longer exists, or when the cause can no longer happen. Record the deletion in the ledger.

## Run record

Every run writes one record. This is the source for rollback rate and for run history.

Required fields: job, graph, node check results, repairs, cost, approver.

Also record `approvals`, `escalated`, and `reverted` so the success metric and the rollback rate are plain counts. Set `reverted` later, on the run that undoes a closed-lane action.

Template: [assets/run-record.md](assets/run-record.md).

A record that exists only in the chat does not count. Write it where the next run can read it.

## What good looks like

Human approvals per run should trend toward 1. That approval is the closed step, or the evidence-lane sign-off when the job has no closed step.

Track the escalation rate next to it: escalations divided by runs. A drop in approvals per run counts as progress when the escalation rate holds or falls too. Read both numbers.

## Rollout

Pilot one recurring job for a week. Use a fixed graph, real checks, and the run record.

At the end of the week, read approvals per run, escalation rate, rollback rate, and the ledger lines you added. Expand to the next job only after that week has a record you can read. Add one job at a time.
