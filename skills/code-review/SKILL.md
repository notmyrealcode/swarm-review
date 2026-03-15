---
name: code-review
description: Review code changes or implementation plans using Codex or Gemini CLI. Use when the user asks to "review code", "review my changes", "review this plan", "code review", "plan review", "review loop", "thorough review", "review and fix", or wants an external AI review. Supports both Codex (default) and Gemini engines, single-pass and iterative loop modes.
allowed-tools: Bash, Read, Glob, Grep, Agent, Write, TaskCreate, TaskUpdate, TaskList, AskUserQuestion
---

# Code Review Skill

## Step 1: Session Setup

### 1a. Create Session Directory

Create a temp directory for all review artifacts:

```bash
SESSION_DIR="/tmp/claude-review-$(date +%s)"
mkdir -p "$SESSION_DIR"
```

**Never write review artifacts to the project directory.** All intermediate files go to the session dir.

### 1b. Detect Review Type and Scope

Determine what to review by checking the user's request AND the repo state:

1. **If the user explicitly mentions a plan**, proposal, design doc, or references a file in `.claude/plans/` → **Plan review**.
2. **Otherwise, probe the repo state** (run `git status --short` and `git diff --stat HEAD` in parallel):
   - If there are **uncommitted changes** (staged, unstaged, or untracked files) → **Code review of uncommitted diff**.
   - Else if the working tree is clean, check for **branch commits** (`git log <base>..HEAD --oneline`):
     - If there are commits on the branch beyond the base → **Code review of branch diff** (Codex) or **latest commit** (Gemini).
     - If there are no branch commits (on base branch, clean tree) → **Nothing to review**. Inform the user and stop.
3. **If ambiguous** (e.g., both uncommitted changes exist AND there are also branch commits, or a plan file exists alongside code changes) → **Ask the user** what they want reviewed:
   - "I see uncommitted changes and N commits on this branch. Would you like me to review the uncommitted changes, the full branch diff, or both?"
   - "I found a plan file at [path] and also code changes. Would you like a plan review or a code review?"
4. **Classify change type** (code review only): Examine the diff and commit messages to classify the change as one of:
   - **Feature**: New functionality — review emphasizes security, edge cases, and API design.
   - **Bug fix**: Correctness fix — review emphasizes root cause correctness, regression risk, and test coverage.
   - **Refactor**: Structural change — review emphasizes behavioral preservation and pattern consistency.
   - **Breaking change**: API/interface change — review emphasizes backward compatibility, migration path, and documentation.
   Mention the classified type when presenting findings so the user understands the review focus.

### 1c. Determine Engine

- If the user explicitly says **"codex"** → use Codex
- If the user explicitly says **"gemini"** → use Gemini
- **Default**: Codex

### 1d. Detect Mode

- **Loop mode**: The user says "review loop", "thorough review", "review and fix", "iterative review", or explicitly requests multiple passes / fixing findings.
- **Single-pass**: Everything else (this is the default). Plan reviews are always single-pass.

## Step 2: Smart Branch Detection (Code Review Only)

Replace hardcoded `main` with intelligent detection. Run these in order, use the first that succeeds:

1. **Check for open PR base**:
   ```bash
   gh pr view --json baseRefName -q .baseRefName 2>/dev/null
   ```
2. **Detect default branch from remote**:
   ```bash
   git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}'
   ```
3. **Fallback**: Try `main`, then `master` (check which exists with `git rev-parse --verify`).

Use the detected base branch for all diff operations.

## Step 3a: Code Review Workflow

### Single-Pass Mode

1. The review type and scope were determined in Step 1b. Run `git branch --show-current` if not already known.
2. Apply engine-specific scope constraints:
   - **Gemini**: If scope is branch diff, fall back to latest commit only (`git show HEAD`) — branch diff causes OOM.
   - **Codex**: All scopes supported (uncommitted, branch diff, specific commit).
3. **Guard: Check for empty diff.** Run the diff command for the chosen scope (`git diff`, `git diff <base>...HEAD`, `git show HEAD`, etc.). If the output is empty, inform the user there is nothing to review and stop.
4. Construct the review prompt (see Code Review Prompt below) — **Gemini only**:
   - Codex generates its own review when using `--base`/`--uncommitted`/`--commit` flags — skip prompt construction for Codex.
   - For Gemini, build the full prompt and combine with the diff into a single input file (see Engine Invocations).
   - If the user provided additional instructions, append them.
5. Run the review using the Bash tool with `run_in_background: true`, timeout 300s (600s for Gemini). You will be notified automatically when complete.
6. **Normalize findings**:
   - **Severity**: Map Codex P0→CRITICAL, P1→IMPORTANT, P2→MINOR, P3+→SUGGESTION. For Gemini, map severity headers: "Critical"→CRITICAL, "High"→IMPORTANT, "Medium"→MINOR, "Low"→SUGGESTION.
   - **Effort**: For each finding, estimate fix effort as `trivial` (typo/one-liner), `easy` (single function/few lines), `medium` (multiple files or logic change), or `hard` (architectural/cross-cutting). Include effort when presenting findings to help prioritize fixes.
7. Write findings to `$SESSION_DIR/findings-iter-1.md`.
8. Present findings to the user, grouped by severity tier. Include the assessment verdict at the end. For Codex (which generates its own review without our prompt), extract or synthesize a verdict from the Codex output during the normalization step.

### Loop Mode (Iterative Review-Fix Cycles)

**Iteration 1:**
1. Run single-pass review (steps 1–8 above).
2. Present findings to the user grouped by severity.
3. **Ask about false positives**: Ask the user which findings (if any) are intentional or false positives.
4. Record confirmed false positives in `$SESSION_DIR/false-positives.md` with file path + description.

**Fix Phase:**

> **Why subagents?** Each fix requires reading files, making edits, and verifying the change — all of which consumes context window. Dispatching fixes as background subagents keeps the main conversation lean for orchestration and re-review. **Never fix inline in the main conversation**, even for one-liners.

5. For each **critical** and **important** finding that is NOT a false positive, dispatch a **separate subagent** to fix it. Use the Agent tool with these parameters:

   ```
   Agent(
     subagent_type = "general-purpose",
     run_in_background = true,
     prompt = """
     Fix the following code review finding. Make the MINIMAL change needed — do not refactor surrounding code or add unrelated improvements.

     ## Finding
     Severity: [CRITICAL/IMPORTANT]
     File: [file path]
     Location: [line number or function name]
     Issue: [description of the issue from the review]
     Suggested fix: [the fix suggestion from the review]

     ## Instructions
     1. Read the file at [file path]
     2. Locate the issue at [line/location]
     3. Apply the minimal fix described above
     4. Do NOT modify any other code
     5. Do NOT add comments explaining the fix
     """,
     description = "Fix: [short description of finding]"
   )
   ```

   **Dispatch rules:**
   - **One fix agent per finding** — never batch multiple fixes into one agent
   - Run fix agents in parallel (multiple Agent calls in same message) when they touch **different files**
   - Run fix agents sequentially when they touch the **same file** (to avoid edit conflicts)
   - Use `run_in_background: true` for all fix agents
   - After dispatching, you will be notified automatically when each agent completes. Review the output for success/failure.

6. After all fixes are applied, run `git diff` to verify changes are scoped correctly.

**Re-Review:**
7. Run the review again:
   - **Codex**: Re-run the same `codex review` command (Codex cannot accept false-positive exclusions). After receiving results, manually filter out known false positives from the findings before presenting.
   - **Gemini**: Re-run with the review prompt + append the false positives list as exclusions in the prompt (e.g., "Ignore the following known false positives: [list]").
8. Write new findings to `$SESSION_DIR/findings-iter-N.md`.
9. Repeat fix cycle until:
   - Zero critical/important findings remain (excluding false positives), OR
   - Maximum **4 iterations** reached

**Finalize:**
10. Write a summary to `$SESSION_DIR/summary.md`:
    - Total findings by severity across all iterations
    - What was fixed (with file paths)
    - What remains (minor/suggestion items)
    - Acknowledged false positives
11. Present the summary to the user.

## Step 3b: Plan Review Workflow

Plan review is always **single-pass** (no loop mode).

1. Locate and read the plan file:
   - If the user specified a file, use that.
   - Otherwise, search `~/.claude/plans/` and `.claude/plans/` for recent plan files.
   - Read the plan file contents (plans may be outside the project root, e.g. `~/.claude/plans/`).
2. Extract referenced file paths from the plan (look for paths like `src/...`, `packages/...`, etc.).
3. Use Glob/Grep to verify those paths exist and find additional related code if needed.
4. Construct the plan review prompt (see Plan Review Prompt below):
   - Inline the full plan content into the prompt (do NOT pass a file path for the review tool to read)
   - Include the list of file paths for context
5. Run the review using the Bash tool with `run_in_background: true`, timeout 300s (600s for Gemini). You will be notified automatically when complete.
6. **Normalize findings** in the engine output:
   - **Severity**: Map Codex P0→CRITICAL, P1→IMPORTANT, P2→MINOR, P3+→SUGGESTION. For Gemini: "Critical"→CRITICAL, "High"→IMPORTANT, "Medium"→MINOR, "Low"→SUGGESTION. If Gemini uses numbered lists without severity labels, infer from context (security/correctness→CRITICAL, best practices→IMPORTANT, style→MINOR).
   - **Effort**: For each finding, estimate fix effort as `trivial`/`easy`/`medium`/`hard`.
7. Write findings to `$SESSION_DIR/findings-iter-1.md`.
8. Present findings grouped by severity tier. Include the assessment verdict at the end. For Codex (which generates its own review without our prompt), extract or synthesize a verdict from the Codex output during the normalization step.

## Step 4: Finalize

- Write final summary to `$SESSION_DIR/summary.md` if not already written (loop mode writes it in step 10).
- Present results to the user.
- Report the session directory path so the user can reference artifacts if needed.

---

## Prompts

### Code Review Prompt

> **Engine applicability**: This prompt is used by **Gemini only**. Codex generates its own review when invoked with `--base`/`--uncommitted`/`--commit` flags and does not accept a custom prompt in that mode.

```
Review the following code changes thoroughly.

Discover and enforce project conventions by reading CLAUDE.md, linter/formatter configs, CI workflows, and other standard instruction files in the codebase. Report findings only. Do not generate replacement code.

Evaluate each change against these criteria, in strict priority order:

**CRITICAL — Security** (Priority 1)
- Injection risks (SQL, command, XSS, path traversal)
- Exposed secrets, tokens, API keys (hardcoded or logged)
- Authentication/authorization bypass
- Unsafe data handling, deserialization

**CRITICAL — Correctness** (Priority 2)
- Logic errors, incorrect algorithms
- Race conditions, concurrency bugs
- Data loss scenarios (silent overwrites, missing transactions)
- Null/undefined paths that cause crashes

**IMPORTANT — Project Rules** (Priority 3)
- Violations of the project standards listed above
- Deviations from established codebase patterns

**IMPORTANT — Error Handling** (Priority 4)
- Silent failures (caught but not reported)
- Swallowed errors, missing error propagation
- Missing cleanup in error paths (unclosed resources, dangling state)

**IMPORTANT — Type Safety** (Priority 5)
- Implicit `any` types
- Unsafe type casts or assertions
- Missing types at module/API boundaries

**MINOR — Edge Cases** (Priority 6)
- Boundary conditions (empty arrays, zero values, max int)
- Unexpected input shapes (null, undefined, wrong type at runtime)
- Large dataset performance (N+1 queries, unbounded loops)

**MINOR — Consistency** (Priority 7)
- Naming conventions (variables, functions, files)
- Pattern consistency with surrounding code
- Readability improvements

**SUGGESTION — Tests** (Priority 8)
- Missing test cases for new/changed behavior
- Coverage gaps for error paths
- Missing edge case tests

For each finding:
- State the severity tier (CRITICAL/IMPORTANT/MINOR/SUGGESTION)
- Identify the specific file and line/location
- Explain the issue concisely
- Explain **why it matters** (impact: what could go wrong, who is affected, what breaks)
- Provide a concrete fix suggestion

Group findings by severity tier. Within each tier, order by impact. If there are no findings for a tier, omit it.

After all findings, provide an **Assessment**:
- **Ready to proceed?** Yes / No / With fixes
- **Reasoning:** 1-2 sentence technical assessment of overall change quality.
```

Append any user-provided custom instructions after the template.

### Plan Review Prompt

```
Review the following implementation plan:

<plan>
{PLAN_CONTENT}
</plan>

Discover and enforce project conventions by reading CLAUDE.md, linter/formatter configs, CI workflows, and other standard instruction files in the codebase.

Read the plan carefully, then examine the related code paths listed below to understand the existing codebase context.

Evaluate the plan against these criteria, in strict priority order:

**CRITICAL — Feasibility** (Priority 1)
- Technical blockers or impossible steps
- Dependencies not accounted for
- Security gaps in the proposed design

**CRITICAL — Correctness** (Priority 2)
- Steps that would produce incorrect behavior
- Missing error handling for likely failure modes
- Data integrity risks

**IMPORTANT — Completeness** (Priority 3)
- Missing steps that an engineer would need to figure out independently
- Gaps in API contracts, data schemas, config changes, migration steps
- Unaddressed deployment or rollback concerns

**IMPORTANT — Ambiguity** (Priority 4)
- Steps where intent is unclear or could be interpreted multiple ways
- Underspecified behavior ("handle errors appropriately" without saying how)
- Missing acceptance criteria

**IMPORTANT — Risk** (Priority 5)
- What could go wrong during implementation
- Rollback implications
- Performance implications at scale
- Backwards compatibility concerns

**MINOR — Consistency** (Priority 6)
- Does the plan follow existing codebase patterns?
- Does it introduce new patterns without justification?
- Are naming conventions consistent?

**MINOR — Documentation** (Priority 7)
- Missing documentation updates
- Unclear configuration requirements
- Missing monitoring/observability considerations

Related code paths to examine for context:
{FILE_PATHS}

For each finding:
- State the severity tier (CRITICAL/IMPORTANT/MINOR)
- Explain the gap concisely
- Explain **why it matters** (impact: what could go wrong, who is affected, what breaks)
- Suggest a specific improvement to the plan

Group findings by severity tier. Within each tier, order by impact.

After all findings, provide an **Assessment**:
- **Ready to proceed?** Yes / No / With fixes
- **Reasoning:** 1-2 sentence technical assessment of overall plan quality.
```

Replace `{PLAN_CONTENT}` (read the plan file and inline its full contents) and `{FILE_PATHS}` with actual values.

---

## Engine Invocations

### Codex — Code Review

**IMPORTANT — CLI syntax**: The `--base`, `--uncommitted`, and `--commit` flags are **mutually exclusive with the positional `[PROMPT]` argument**. When using these flags, do NOT pass a custom prompt — Codex generates its own review prompt from the diff.

The Code Review Prompt template (below) is therefore **not used** when Codex is invoked with these flags — Codex generates its own review. Only use the template with bare `codex review "<prompt>"` (no diff-scoping flags).

```bash
# Uncommitted changes (Codex auto-generates review):
codex review --uncommitted

# Branch diff against base:
codex review --base <detected_base>

# Specific commit:
codex review --commit <sha>

# No flags, custom prompt as positional arg:
codex review "<prompt>"
```

### Codex — Plan Review

```bash
# Plan content is inlined into the prompt (read by Claude before invoking Codex):
echo "<prompt with {PLAN_CONTENT} inlined>" | codex exec --cd <project_root> --sandbox read-only --skip-git-repo-check -
```

### Gemini — Code Review

Gemini has no `review` subcommand. **The diff and review prompt MUST be combined into a single file piped via stdin**, with `-p` providing only a short instruction. If you pipe a raw diff on stdin with a long prompt in `-p`, Gemini ignores the diff and goes agentic (exploring the codebase with its own tools, which fail).

> **Known behavior — Gemini goes agentic**: Even with `--approval-mode plan` and the correct stdin-piped pattern, Gemini may still read files from the codebase beyond the provided diff. The `--approval-mode plan` flag prevents writes but not reads. Results are still useful but may cover more than the diff scope. Use `--approval-mode plan` instead of `-s` for read-only mode with tighter tool control (confirmed supported: choices are `default`, `auto_edit`, `yolo`, `plan`).

**Scope rules for Gemini** (to avoid OOM with large diffs):
- If there are uncommitted changes → use uncommitted diff only
- If the working tree is clean → use latest commit only
- If the user specifies a commit → use that commit
- **Never use branch diff** (`git diff base...HEAD`) with Gemini — it causes OOM on large branches

**Correct invocation pattern:**

```bash
# Step 1: Write review prompt + diff into a single file
cat > "$SESSION_DIR/gemini-input.txt" << 'REVIEWEOF'
<review prompt text here>

DIFF:
REVIEWEOF
git diff HEAD >> "$SESSION_DIR/gemini-input.txt"  # or git show <sha>, etc.

# Step 2: Pipe the combined file, use -p for a short instruction only
cat "$SESSION_DIR/gemini-input.txt" | gemini --approval-mode plan -p "Analyze the diff provided on stdin. Report findings only." -o text
```

> **Rate limiting**: Gemini frequently hits quota exhaustion, triggering exponential backoff (~50s+ per attempt). Use **600s timeout** (not 300s). In loop mode this compounds across iterations.

**WRONG (Gemini ignores the diff and explores codebase):**
```bash
git diff HEAD | gemini --approval-mode plan -p "<long review prompt>" -o text
```

Key flags:
- `--approval-mode plan` — **read-only mode, prevents all tool writes and restricts tool use**
- `-o text` — plain text output (avoids hard-to-parse JSONL)

### Gemini — Plan Review

```bash
# Plan content is inlined into the prompt (read by Claude before invoking Gemini):
gemini --approval-mode plan -p "<plan review prompt with {PLAN_CONTENT} inlined>" -o text
```

### Diff Too Large Fallback

If the diff is too large for the engine (Gemini especially):
- Fall back to latest commit scope (`git show HEAD`) instead of branch diff
- **Never skip the review entirely**

---

## Iron Rules

These rules prevent common shortcuts that degrade review quality. Do not rationalize past them.

| Thought | Rule |
|---------|------|
| "I'll batch these fixes together" | **ONE fix agent per finding.** Never batch. Each fix gets isolated context. |
| "This is a one-liner, I'll edit directly" | **Dispatch a subagent.** No exceptions in loop mode. |
| "Two iterations is enough" | **Check for zero critical/important findings.** Don't stop early if issues remain (up to 4 max). |
| "The diff is too large, I'll skip review" | **Fall back to latest commit scope.** Never skip. |
| "I'll fix and review in the same pass" | **Review pass and fix pass are SEPARATE.** Never combine. Review first, then fix, then re-review. |
| "False positives don't matter" | **Track them.** They prevent the same non-issue from re-triggering in subsequent iterations. |
| "The diff is empty, I'll review anyway" | **Check for empty diff first.** If no changes exist, inform the user and skip the review. |
