# swarm-review

Multi-agent code review plugin for [Claude Code](https://claude.ai/claude-code). Dispatches a swarm of agents to review, fix, and re-review your code changes using Codex or Gemini CLI as the review engine.

## What it does

- **Single-pass mode** — runs an external AI review (Codex or Gemini) against your uncommitted changes or branch diff and presents findings grouped by severity (CRITICAL / IMPORTANT / MINOR / SUGGESTION).
- **Loop mode** — iteratively reviews, dispatches fix agents for critical/important findings, then re-reviews until clean (up to 4 iterations). Each fix runs in an isolated subagent so the main conversation stays lean.
- **Plan review** — reviews implementation plans (from `.claude/plans/`) for feasibility, completeness, and risk.
- **Smart branch detection** — automatically detects the correct base branch via open PR, remote default, or fallback heuristics.

## Prerequisites

| Tool | Required | Purpose |
|------|----------|---------|
| [Claude Code](https://claude.ai/claude-code) | Yes | Plugin host — runs the skill |
| [Codex CLI](https://github.com/openai/codex) | Yes (default engine) | Performs the code review |
| [Gemini CLI](https://github.com/google-gemini/gemini-cli) | Optional | Alternative review engine |
| [GitHub CLI (`gh`)](https://cli.github.com/) | Recommended | Smart branch detection via PR base |

## Installation

In Claude Code, run:

```
/plugin install notmyrealcode/swarm-review
```

The skill becomes available as `/swarm-review:code-review`.

## Usage

Invoke the skill in Claude Code:

```
/swarm-review:code-review
```

### Single-pass (default)

Just ask for a review — single-pass is the default mode:

- "review my changes"
- "code review"
- "review this with gemini"

### Loop mode

Trigger iterative review-fix cycles with phrases like:

- "review loop"
- "thorough review"
- "review and fix"
- "iterative review"

Loop mode reviews your code, asks about false positives, dispatches fix agents for critical/important findings, then re-reviews — repeating until clean or 4 iterations max.

### Engine selection

- **Codex** (default) — just ask for a review
- **Gemini** — say "review with gemini" or "use gemini"

## Supported modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Single-pass** | Default | One review pass, findings presented grouped by severity |
| **Loop** | "review loop", "thorough review", "review and fix" | Iterative review → fix → re-review cycles (up to 4 iterations) |
| **Plan review** | Reference a plan file or say "review this plan" | Single-pass review of an implementation plan for feasibility, completeness, and risk |

## License

MIT
