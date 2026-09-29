---
name: review-loop
description: Run the worker/reviewer ping-pong on the current branch. Spawns independent reviewer agents one after another, addresses each one's findings as the worker, and keeps going until two consecutive fresh reviewers find no bugs on their first pass. Reviewers are read-only on the working tree and never run the app or tests. Use when asked to review-loop, run the review loop, or iterate with reviewers until clean.
argument-hint: "[base=<branch>] [model=opus|fable[,...]] [clean=2] [rounds=4] [max_reviewers=6] [focus=\"...\"] [context]"
---

# Review loop

## Roles

- **Worker**: the session that invokes this skill. It keeps the implementation context and
  addresses findings itself. It is never replaced and its model is never overridden.
- **Reviewer**: a subagent spawned per pass with its own context. Within a pass it is continued
  with SendMessage, so it remembers what it raised and what the worker answered. Each new pass
  gets a brand-new reviewer that has seen none of the earlier passes.

Reviewers are read-only on the working tree. They judge by reading the diff and the surrounding
code; nothing has to be run. They may run read-only checks (a type checker, a linter in check
mode) to confirm a finding. They must not run the app, the test suite, formatters, fix modes,
builds, or task-runner targets, and must not modify any file. Verification belongs to the worker.

## Arguments

Parse `$ARGUMENTS` as `key=value` tokens; anything else is free-text context handed to every reviewer.

| Key | Default | Meaning |
|---|---|---|
| `base` | auto | Branch the diff is taken against. See Diff base. |
| `model` | `opus` | Reviewer model alias. A comma list assigns per pass in order, last value repeats. |
| `clean` | `2` | Consecutive clean passes required to finish. |
| `rounds` | `4` | Round cap inside one pass before open findings go to the user. |
| `max_reviewers` | `6` | Hard cap on passes. Reaching it without `clean` clean passes is reported, not hidden. |
| `focus` | correctness and no overengineering | Review focus for all reviewers. |

The user may change `model` mid-loop in chat. It applies to the next spawned reviewer.

## Diff base

The review covers what this branch adds on top of the branch it was created from, not on top
of the default branch. Resolve the base once, before the first pass, in this order:

1. `base=` if given.
2. The branch's creation entry, `git reflog show <branch> | tail -1`, when it names a branch
   rather than `HEAD`.
3. Otherwise the local or remote branch, excluding the current one and its own remote tracking
   ref, whose merge-base with HEAD leaves the fewest commits in `merge-base..HEAD`. Ties go to
   the default branch.

Name the resolved base in the first line to the user so a wrong guess is visible, and pass it
to every reviewer as `{base}`.

## Termination

A **pass** is one reviewer, from spawn until it says satisfied or the round cap hits.

A pass is **clean** when the reviewer's first reply contains no `blocker` or `should-fix`
finding. Nits alone keep a pass clean; they are still addressed in the ping-pong.

The loop ends when `clean` consecutive passes were clean. Any pass with a blocker or should-fix
finding resets the streak to zero. It also ends at `max_reviewers` passes, reported as not
converged with the last pass's open findings.

## One pass

1. Spawn the reviewer with the Agent tool: `subagent_type: general-purpose`, `model` from the
   argument, `run_in_background: false`. Prompt is the reviewer brief below plus the free-text
   context. Do not hand it the earlier passes; fresh eyes is the point.
2. Read the verdict line. `VERDICT: SATISFIED` ends the pass. Record whether it was clean.
3. Otherwise act as the worker on every finding: fix it, or reject it with a concrete rationale.
   Never accept a finding just to end the round; never reject one to avoid work.
4. SendMessage the same reviewer a per-finding response (`fixed` with what changed, or
   `rejected` with why) and ask for a re-review of the current diff. Back to step 2.
5. At the round cap, end the pass and list the open findings with both sides' arguments.

After every pass, one short line to the user: pass number, clean or not, findings fixed and
rejected, streak so far. After the loop: passes used, why it stopped, open findings if any.

Never commit or push inside the loop. The worker may run tests to verify its own fix, but must
not ask the reviewer to. The worker edits and runs things only between reviewer rounds, never
while a reviewer is reading the tree.

## Reviewer brief

Send this verbatim, with `{base}`, `{focus}` and `{context}` substituted:

```
You are an independent code reviewer. Review the changes being done in this branch: git diff
against the merge base with {base}, plus untracked files. Focus on {focus}.
{context}

You are read-only on the working tree; the worker is editing it. Nothing has to be run: judge
by reading the code. You may run read-only checks (a type checker, a linter in check mode) to
confirm a finding, and say so when a finding rests on tool output. You must not run the app,
the test suite, formatters, fix modes, builds, or task-runner targets, and must not modify any
file. If a point can only be judged by running the code, mark the finding "unverified".

Rules:
- Read the surrounding code before judging a change; do not review the diff in isolation.
- Number every finding, cite file:line, one severity each: blocker, should-fix, nit.
- No praise, no restating what the change does.
- You will receive per-finding responses from the worker and be asked to re-review. Re-read the
  actual diff; do not trust the worker's description of a fix.
- Accept a rejection when its rationale is sound. Do not repeat a rejected finding without a new
  argument.
- Do not raise new nits after round 2 unless a fix introduced them.

End every reply with exactly one line:
VERDICT: SATISFIED
or
VERDICT: <n> OPEN
```

Default `{focus}`: `correctness and that things are not overengineered`.
