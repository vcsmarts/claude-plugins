# claude-plugins

Claude Code skills I carry across repos.

## Plugins

| Plugin | What it does | Setup |
|---|---|---|
| [`review-loop`](plugins/review-loop/) | `/review-loop`: worker/reviewer ping-pong on the current branch with independent reviewer agents, each addressed until satisfied. | None |

## Install

Clone once, then symlink each skill into the user skills folder. Claude Code loads everything
there in every session; `git pull` in the clone is the update.

```
git clone git@github.com:vcsmarts/claude-plugins.git ~/repos/claude-plugins
ln -s ~/repos/claude-plugins/plugins/review-loop/skills/review-loop ~/.claude/skills/review-loop
```

The `.claude-plugin` manifests are not needed for this; they keep the marketplace route open.
That route is `/plugin marketplace add vcsmarts/claude-plugins` followed by
`/plugin install review-loop@saturn`, and it replaces the symlink, never both.
