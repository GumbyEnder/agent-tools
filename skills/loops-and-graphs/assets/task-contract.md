# Task contract

Copy one block per node. Fill `CHECK` before `JOB`. One node, one job, one output.

```
NODE:
JOB:
IN:
OUT:
FAILS AS:
CHECK:
LANE: open | evidence | closed
```

- `IN` names each variable and the node that produced it.
- `OUT` names the variable and its shape: fields, type, or file set.
- `FAILS AS` is what a failed check looks like, in terms a program can observe.
- `CHECK` is a command, schema, diff rule, or checklist a program can fail.
- `LANE` is how hard the step is to undo: `open`, `evidence`, or `closed`.
- Name the variable this node's `OUT` will be called on the next edge.

## Example

```
NODE: export-timeout
JOB: Add a timeout around the export retry and cover it with a test.
IN: plan.files, plan.timeout_ms (from plan)
OUT: patch. Diff limited to plan.files, plus the test result
FAILS AS: the test command exits non-zero, or the diff names a file outside plan.files
CHECK: pytest tests/test_export_timeout.py exits 0; git diff --name-only ⊆ plan.files
LANE: open
```
