# review-loop

The ticket workflow as a skill: one worker and a chain of independent reviewers, each
ping-ponged with the worker until satisfied. Fresh reviewers keep coming until two in a row
find no bugs on their first pass. Reviewers are read-only on the working tree and judge by
reading the code; they may run read-only checks but never the app or tests.

The session that runs `/review-loop` is the worker. It keeps the implementation context and
addresses findings itself. Reviewers are subagents with their own context, continued with
SendMessage so a reviewer remembers what it raised and what the worker answered.

## Usage

```
/review-loop [base=<branch>] [model=opus|fable[,opus|fable...]] [clean=2] [rounds=4] [max_reviewers=6] [focus="..."] [context]
```

| Key | Default | Meaning |
|---|---|---|
| `base` | auto | Branch the diff is taken against. Auto resolves to the branch this one was created from (reflog, then closest merge-base), so stacked branches are reviewed against their parent. |
| `model` | `opus` | Reviewer model alias. A comma list assigns per reviewer in order, last value repeats. |
| `clean` | `2` | Consecutive fresh reviewers that must find no bugs on their first pass. |
| `max_reviewers` | `6` | Hard cap on passes; hitting it is reported as not converged. |
| `rounds` | `4` | Round cap per reviewer before open findings go to the user. |
| `focus` | correctness and no overengineering | Review focus for all reviewers. |

Free text after the keys is handed to every reviewer as context. Saying "switch the reviewer to
fable" mid-loop applies to the next spawned reviewer; a running one keeps its model.

The loop never commits or pushes. Reviewers may run read-only checks (a type checker, a linter
in check mode) to confirm a finding, but never the app, the test suite, formatters, fix modes,
builds or task-runner targets, and never modify a file. The worker may run tests to check its
own fix, only between reviewer rounds.

## Prerequisites

None.
