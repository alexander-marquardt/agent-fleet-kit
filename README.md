# agent-fleet-kit

Small, standalone pieces for running a fleet of [Claude Code](https://claude.com/claude-code) sub-agents without paying for context they no longer need. They come from the workflow described in two articles:

- [One developer, a fleet of Claude Code agents, 1,652 merged PRs in a month](https://alexmarquardt.com/ai-tools/one-developer-coding-agent-fleet/): the rules, tests, gates and control plane that keep the code under control.
- [Where a fleet of coding agents spends its tokens](https://alexmarquardt.com/ai-tools/agent-fleet-token-efficiency/): how the token use was measured, and what each change recovered.

**This is a small extracted starting point, not the system the articles describe.** That system runs from a private repository: an allocator that runs the agent slots from one queue, agent roles for implementing, reviewing and fixing, a watchdog, per-agent clones, CI gates and the written rules behind them. What is here are the three pieces that are generic and paid off most for token cost, rewritten from scratch as minimal versions, with no dependencies beyond Python 3.9+. Each is a single file you can read in a few minutes.

## What it covers, and what it does not

- **Covered:** the two largest token-cost fixes in [the token article](https://alexmarquardt.com/ai-tools/agent-fleet-token-efficiency/) (ending agents at a context ceiling, and not letting the prompt cache expire while an agent waits) and the script that measures them. With these and the measurement, you can reproduce that article's main levers on your own sessions.
- **Not covered:** most of [the fleet article](https://alexmarquardt.com/ai-tools/one-developer-coding-agent-fleet/): the control plane, the review agents, the gates and the rules. The article describes each one well enough to build your own; the code for them is not published.

## What is in it

| File | What it does | What it changed on the fleet it comes from |
|---|---|---|
| [`hooks/context_ceiling.py`](hooks/context_ceiling.py) | A `PreToolUse` hook. Once a sub-agent's context passes 150k tokens it is told to finish at its next commit and write a handoff; past 200k only `git` and `gh` commands run, so the agent hands off and ends. | Median context per agent request fell from 218k to 101k tokens on the first day. |
| [`hooks/wait_cap.py`](hooks/wait_cap.py) | A `PreToolUse` hook on `Bash`. A sub-agent may not block in the foreground (long `timeout`, long `sleep`) past the prompt cache's lifetime; it runs the command in the background and polls instead. | Cache expiry fell from about 31% of agent usage to under 1%. |
| [`scripts/session_tokens.py`](scripts/session_tokens.py) | Sums token use per session from the transcripts Claude Code already keeps in `~/.claude/projects/`, counting each request once. | It is how the two numbers above were found. |

Both hooks judge sub-agents only; the main session is left alone. Both fail open: if a hook cannot read what it needs, the tool call runs. That includes a new sub-agent's first tool call, which can run before its first reply is written to its transcript. Recent Claude Code versions refuse a bare long `sleep` themselves; the wait cap's own `sleep` check is a backstop, and its `timeout` check is the part that matters.

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

The tests run on fixed transcripts and hook inputs; they need no network and no Claude Code install. Both hooks were also checked live with Claude Code 2.1.318: past the ceiling a sub-agent's `ls` and `echo` were refused and `git --version` ran; a foreground `timeout` of 600 s was refused, and the same call with `run_in_background` ran.

## License

MIT
