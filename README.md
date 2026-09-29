# claude-plugins

Claude Code skills I carry across repos.

## Plugins

| Plugin | What it does | Setup |
|---|---|---|
| [`review-loop`](plugins/review-loop/) | `/review-loop`: worker/reviewer ping-pong on the current branch with independent reviewer agents, each addressed until satisfied. | None |

## Install

In Claude Code:

```
/plugin marketplace add vcsmarts/claude-plugins
/plugin install review-loop@saturn
```

Or from a shell:

```
claude plugin marketplace add vcsmarts/claude-plugins
claude plugin install review-loop@saturn --scope user
```

Update later with `claude plugin update review-loop@saturn`.
