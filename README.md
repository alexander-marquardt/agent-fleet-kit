# agent-fleet-kit

Small, standalone pieces for running a fleet of [Claude Code](https://claude.com/claude-code) sub-agents without paying for context they no longer need. They come from the workflow described in two articles:

- [One developer, a fleet of Claude Code agents, 1,652 merged PRs in a month](https://alexmarquardt.com/ai-tools/one-developer-coding-agent-fleet/): the rules, tests, gates and control plane that keep the code under control.
- [Where a fleet of coding agents spends its tokens](https://alexmarquardt.com/ai-tools/agent-fleet-token-efficiency/): how the token use was measured, and what each change recovered.

The versions here are minimal rewrites of the ideas, with no dependencies beyond Python 3.9+. Each is a single file you can read in a few minutes.

## What is in it

| File | What it does | What it changed on the fleet it comes from |
|---|---|---|
| [`hooks/context_ceiling.py`](hooks/context_ceiling.py) | A `PreToolUse` hook. Once a sub-agent's context passes 150k tokens it is told to finish at its next commit and write a handoff; past 200k only `git` and `gh` commands run, so the agent hands off and ends. | Median context per agent request fell from 218k to 101k tokens on the first day. |
| [`hooks/wait_cap.py`](hooks/wait_cap.py) | A `PreToolUse` hook on `Bash`. A sub-agent may not block in the foreground (long `timeout`, long `sleep`) past the prompt cache's lifetime; it runs the command in the background and polls instead. | Cache expiry fell from about 31% of agent usage to under 1%. |
| [`scripts/session_tokens.py`](scripts/session_tokens.py) | Sums token use per session from the transcripts Claude Code already keeps in `~/.claude/projects/`, counting each request once. | It is how the two numbers above were found. |

Both hooks judge sub-agents only; the main session is left alone. Both fail open: if a hook cannot read what it needs, the tool call runs.

## Install

Copy the hooks into your project and register them:

```sh
mkdir -p .claude/hooks
cp hooks/context_ceiling.py hooks/wait_cap.py .claude/hooks/
```

Then merge [`settings.example.json`](settings.example.json) into `.claude/settings.json`. Run the token script from anywhere:

```sh
python3 scripts/session_tokens.py --last 20
```

## Settings

| Variable | Default | Meaning |
|---|---|---|
| `CONTEXT_ADVISORY_TOKENS` | 150000 | Context at which a sub-agent is told to hand off at its next commit |
| `CONTEXT_CEILING_TOKENS` | 200000 | Context past which only `git` and `gh` run |
| `WAIT_CAP_SECS` | 270 | Longest foreground wait. Keep it under your prompt cache's lifetime (five minutes by default); raise it if your sessions use a longer cache |
| `WAIT_CAP_MAIN` | unset | `1` also applies the wait cap to the main session |

The right thresholds depend on your work. Measure first with `session_tokens.py`: divide tokens by the work they produced (merged PRs, changed lines), not by hour, and compare before and after each change.

## Why a hook and not a prompt

A line in a prompt asking an agent to hand off early, or not to wait on CI in the foreground, is a rule it has to remember, and it is followed most of the time. A hook is followed every time. The refusal text tells the agent what to do instead, so the agent does not get stuck.

## Tests

```sh
python3 -m unittest discover tests
```

The tests run on fixed transcripts and hook inputs; they need no network and no Claude Code install.

## License

MIT
