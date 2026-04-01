# Bug Fix Pipeline Guide

Reference for building automated fix pipelines using Claude CLI + Codex CLI.
Written from practical experience fixing 86+ bugs across a multi-repo Python ecosystem.

## Architecture

```
fix-script.py
  |
  for each step:
  |   1. Claude Opus 4.6 (high) -- implements fix + tests + commit
  |   2. Codex GPT-5.4 -- reviews the commit (JSON verdict)
  |   3. if FAIL: Codex GPT-5.4 -- fixes issues + commit
  |   4. goto 2 (max 5 iterations)
  |   5. next step
  |
  after all steps:
  |   - collect unfixed gaps from steps that hit max iterations
  |   - run manual Codex fix→review loop for each gap batch
  |   - restart failed/skipped steps
```

## CLI Reference

### Claude CLI

```bash
# Non-interactive execution with full permissions
claude -p \
  --model claude-opus-4-6 \
  --effort high \
  --dangerously-skip-permissions \
  --no-session-persistence \
  --append-system-prompt "SYSTEM PROMPT TEXT" \
  "TASK PROMPT TEXT"
```

Key flags:
- `-p` — pipe/non-interactive mode (required for scripting)
- `--model claude-opus-4-6` — model selection
- `--effort high` — max reasoning effort
- `--dangerously-skip-permissions` — skip all tool approval prompts
- `--no-session-persistence` — don't save session (avoids disk bloat)
- `--append-system-prompt` — inject system-level instructions (role, workflow)

Prompt is the last positional argument (single string). Multi-line works fine.

CWD: subprocess `cwd=` sets the working directory. Agent sees that as its root.

### Codex CLI

```bash
# Non-interactive execution (for fixes)
codex exec \
  -m gpt-5.4 \
  --dangerously-bypass-approvals-and-sandbox \
  --ephemeral \
  -o /path/to/output.txt \
  "PROMPT TEXT"

# With stdin prompt (for multi-line or special chars)
cat <<'PROMPT' | codex exec -m gpt-5.4 --dangerously-bypass-approvals-and-sandbox --ephemeral -o output.txt -
Multi-line prompt here.
Works with any content.
PROMPT
```

Key flags:
- `exec` — non-interactive subcommand (required for scripting)
- `-m gpt-5.4` — model selection
- `--dangerously-bypass-approvals-and-sandbox` — full access (like Claude's --dangerously-skip-permissions)
- `--ephemeral` — don't persist session
- `-o file` — write last agent message to file (for parsing review output)
- `-C dir` — NOT available on `exec` subcommand! Use subprocess `cwd=` instead

**Important:** Codex `exec` does NOT support `-C` flag. Set working directory via `subprocess.run(cwd=...)`.

**Important:** `codex exec review --uncommitted` cannot be combined with a custom prompt argument. Use `codex exec` with a review-style prompt instead for full control.

### Proxy

Both tools inherit proxy from environment. If you need a proxy, set all four variants:
```bash
export http_proxy=http://HOST:PORT
export https_proxy=http://HOST:PORT
export HTTP_PROXY=http://HOST:PORT
export HTTPS_PROXY=http://HOST:PORT
```

Pass env to subprocess: `env=os.environ.copy()`

## Script Structure

### 1. Configuration Block

```python
WORKSPACE = Path("/path/to/your/workspace")
LOG_DIR = WORKSPACE / ".audit" / "fix-logs-<severity>"
TRACKING = WORKSPACE / ".audit" / "TRACKING.md"

CLAUDE_BIN = "claude"
CLAUDE_MODEL = "claude-opus-4-6"      # or latest available
CLAUDE_EFFORT = "high"
STEP_TIMEOUT = 3000                    # 50 min per step

CODEX_BIN = "codex"
CODEX_MODEL = "gpt-5.4"               # or latest available
CODEX_REVIEW_TIMEOUT = 600            # 10 min for review
CODEX_FIX_TIMEOUT = 1800              # 30 min for fix
MAX_REVIEW_ITERATIONS = 5             # 5 is the sweet spot (3 is too few)
MAX_FIX_ATTEMPTS = 3                  # retry fix on non-zero exit
```

> **Why 5 iterations, not 3?** In practice, ~60% of steps need at least 1 review
> fix. Complex steps (security, concurrency) often need 3 iterations to fully
> close. With only 3 max, the pipeline exits with unresolved gaps that require
> manual intervention. 5 iterations covers virtually all cases.

### 2. System Prompt (appended to Claude)

```python
APPEND_SYSTEM = """\
You are a senior software engineer fixing {SEVERITY} bugs in {PROJECT_NAME}.

{# Include backward-compat note if applicable: #}
IMPORTANT: No backward compatibility required. This is a dev-version.
Make clean, breaking changes where needed.

Mandatory workflow:
1. Read ALL bug report files listed in the task.
2. Read affected source files and existing tests.
3. Research best practices: WebSearch CWE/OWASP IDs from the bug reports.
4. Check if any previously committed fixes partially address the issue.
5. Implement fix following existing code style. Minimal, targeted changes.
6. Write comprehensive tests covering the bug AND the fix.
7. Run {test_cmd}, {lint_cmd}, {typecheck_cmd}. Fix ALL issues.
8. Update broken tests if they relied on buggy behavior.
9. Git commit with descriptive message.

Key info:
{# Project-specific commands and conventions here #}
"""
```

> **Tip:** Step 4 ("check previously committed fixes") is critical when running
> batches sequentially. Later agents need to know what earlier agents already
> changed, especially in shared files like safety policies or config resolvers.

### 3. Step Prompt Template

Each step prompt should include:

```
## Read First
- Bug report file paths (absolute)
- Source file paths (with read instructions)
- Test file discovery: Glob("repo/tests/test_*pattern*.py")

## Research
WebSearch: "CWE-XXX specific query"
WebSearch: "OWASP category remediation pattern"

## Bug Fixes
### BUG-ID: Title
Detailed fix instructions:
1. What to change
2. Where exactly (file, function, line range)
3. Expected behavior after fix

## Tests — repo/tests/test_file_name.py
1. test_specific_case_1
2. test_specific_case_2
...

## Validate
```bash
cd /path/to/repo
uv run pytest tests/test_file.py -v
uv run pytest tests/ -x
uv run ruff check changed_files
uv run mypy changed_files
```

## Commit
```bash
cd /path/to/repo
git add -A
git commit -m "fix(area): BUG-IDs — short description"
```
```

### 4. Codex Review Prompt

```python
review_prompt = f"""
You are a strict security code reviewer.

Review the latest commit ({sha}) which fixes: {bug_ids}.

Steps:
1. Run: git show {sha} --stat
2. Run: git diff {sha}^..{sha}
3. Read changed files for full context
4. Evaluate:
   - Are all listed bugs actually fixed?
   - Any new bugs introduced?
   - Tests comprehensive?
   - Security issues?
   - Quality gates pass?

Output ONLY a JSON object:
{{"pass": true/false, "issues": ["issue1", "issue2"]}}
"""
```

### 5. Codex Fix Prompt

```python
fix_prompt = f"""
You are a senior software engineer fixing code review issues.

Issues found by reviewer for {bug_ids}:
{issues_text}

Fix ALL issues. Then:
1. Run relevant tests
2. Run ruff check and mypy
3. Commit with message: "fix: address review feedback for {step_name}"
"""
```

### 6. Review Loop

```python
def codex_fix(name, bug_ids, issues):
    """Fix with retry — Codex can fail (exit=1) on complex multi-file changes."""
    for attempt in range(1, MAX_FIX_ATTEMPTS + 1):
        result = run_codex(...)
        if result.returncode == 0:
            return True
        log(f"Attempt {attempt} failed, retrying...")
    return False

def review_and_fix_loop(name, bug_ids):
    for i in range(1, MAX_ITERATIONS + 1):
        verdict = codex_review(name, bug_ids)  # returns {"pass": bool, "issues": [...]}
        if verdict["pass"]:
            return  # clean
        fixed = codex_fix(name, bug_ids, verdict["issues"])
        if not fixed:
            continue  # DON'T stop — next review may find partial fix sufficient
    # max iterations reached — move on, collect gaps for later
```

> **Critical: never stop on fix failure.** Codex `exec` can exit non-zero due to
> tool errors, network issues, or hitting complexity limits. Retrying (up to 3
> attempts) often succeeds. Even if all retries fail, the next review iteration
> gives a fresh chance — the partial changes from the failed attempt may have
> improved the codebase enough for the next review to pass or find fewer issues.

### 7. Step Execution

```python
for step_name, prompt, bug_ids in STEPS:
    success = run_claude_step(step_name, prompt, bug_ids)
    if success:
        review_and_fix_loop(step_name, bug_ids)
```

### 8. Gap Closure (post-pipeline)

Steps that hit max iterations leave unresolved issues. After the main pipeline:

1. **Collect gaps**: Read the last `*-review-output.txt` for each step that hit max iterations
2. **Batch related gaps**: Combine gaps from multiple steps into one fix prompt
3. **Manual review loop**: Run Codex fix → Codex review → repeat until PASS
4. **Restart failed steps**: If any steps failed (API errors, timeouts), restart them

```python
# Collect remaining issues from review output files
gaps = []
for step_name in steps_with_max_iterations:
    verdict_file = LOG_DIR / f"{step_name}-review-output.txt"
    verdict = json.loads(verdict_file.read_text())
    if not verdict["pass"]:
        gaps.extend(verdict["issues"])

# Fix all gaps in one Codex session, then review
# Repeat until PASS (manual loop, not the automated script)
```

> **Why separate gap closure?** The automated loop has a fixed iteration budget.
> Some issues (complex concurrency, subtle regex bypasses) need more focused
> attention. The gap closure phase gives unlimited iterations with human oversight.

## Bug Grouping Strategy

Group bugs by:
1. **Repository** — core vs enterprise (separate git histories)
2. **Shared files** — bugs touching the same file MUST be in the same batch
3. **Domain** — related functionality together (shell safety, subagent, etc.)
4. **Size** — 4-8 bugs per batch (sweet spot for 50-min timeout)

Sequential execution eliminates cross-batch conflicts.

## Timeouts (empirical)

| Operation | Timeout | Typical | Notes |
|-----------|---------|---------|-------|
| Claude fix (4-6 bugs) | 50 min | 10-20 min | Research + impl + tests + validation |
| Claude fix (7-10 bugs) | 50 min | 15-25 min | Agent parallelizes internally |
| Codex review | 10 min | 3-5 min | Reads diff, runs checks, outputs JSON |
| Codex fix | 30 min | 5-15 min | Implements review feedback, runs tests |
| Full step with review | — | 30-60 min | Fix + 2-3 review cycles typical |

## Error Handling & Restartability

### API Errors (HTTP 500)
LLM API providers have transient outages. The script should:
- **Continue on failure**: Don't abort the whole pipeline when one step fails
- **Log the error clearly**: Distinguish API errors from agent failures
- **Support range restart**: `python3 script.py START END` to rerun specific steps

```python
# In run_step, capture and log API errors specifically
if "API Error: 500" in result.stdout:
    log(f"API outage during {name} — restart this step later")
```

### Uncommitted Changes After Failure
If Claude crashes mid-step (API error, timeout), it may leave uncommitted changes:
- **Check**: `git diff --stat HEAD` after each failure
- **Decision**: If changes look complete, commit manually. If partial, discard with `git checkout -- .`
- **Rule of thumb**: If the commit message was in the prompt and files match the expected changes, commit it

### Codex Fix Failures (exit=1)
Codex `exec` can fail with exit code 1 due to:
- Too many lint/type errors to fix in one session
- Network issues mid-execution
- Complexity limits (too many files to modify)

**Always retry:** Set `MAX_FIX_ATTEMPTS = 3`. Each retry is a fresh session
that sees the state left by the previous attempt. Even partial progress
compounds — attempt 2 starts from a better state than attempt 1.

**Never stop the review loop on fix failure:** Continue to the next review
iteration. The partial changes may be sufficient, or the reviewer may find
fewer issues on the next pass.

### Avoiding Parallel Conflicts
**Never run two agents in the same repo simultaneously.** This includes:
- Codex "rescue" tasks fixing gaps while the main pipeline runs
- Manual edits while the pipeline is active

If you need to fix gaps from earlier steps while later steps run,
**wait for the pipeline to pause/complete first**.

### Script Supports Selective Re-execution
Design the script to accept step ranges:
```bash
python3 script.py           # all steps
python3 script.py 4         # from step 4 onwards
python3 script.py 4 5       # only steps 4 and 5
python3 script.py 7 8       # only steps 7 and 8
```

This allows surgical reruns after API failures or gap closure.

## Logging

Store per-step logs + review logs:
```
.audit/fix-logs-high/
  orchestrator.log          # timestamped status lines
  step-01-name.log          # Claude stdout/stderr
  step-01-name-review.log   # Codex review stdout
  step-01-name-review-output.txt  # Codex JSON verdict
  step-01-name-fix.log      # Codex fix stdout
```

## Tracking

Maintain a TRACKING.md with tables per severity:
```markdown
| # | ID | Status |
|---|-----|--------|
| 1 | BUG-001 | ⏳ |
```

Auto-update via regex substitution after each step.

**Gotcha:** Bug IDs in TRACKING.md must exactly match IDs in the script's
`bug_ids` list, or the regex won't find them.

## Pre-flight Checklist

Before running the script:

1. **Test Claude CLI**: `claude -p --model claude-opus-4-6 --effort high --dangerously-skip-permissions --no-session-persistence "Say hello"`
2. **Test Codex CLI**: `codex exec -m gpt-5.4 --dangerously-bypass-approvals-and-sandbox --ephemeral "Say hello"`
3. **Verify proxy**: `echo $http_proxy` (must be set if needed)
4. **Verify Python**: `python3 -c "import py_compile; py_compile.compile('script.py', doraise=True)"`
5. **Verify step count**: Run the STEPS verification snippet (see fix-high.py)
6. **Check git state**: Clean working tree in both repos

## Lessons Learned

### Review Loop
- **Codex finds real gaps**: The review loop catches incomplete fixes, missing edge
  cases, and quality gate failures that Claude missed. ~60% of steps need at least
  one review fix iteration. Worth every minute.
- **5 iterations, not 3**: Complex bugs (regex bypasses, concurrency, protocol
  compliance) routinely need 3-4 iterations. Set MAX_REVIEW_ITERATIONS=5.
- **Review drift on later iterations**: After 3+ iterations, Codex may start finding
  issues tangential to the original bugs (e.g., pre-existing code smells). This is
  noise — the gap closure phase handles these better with focused prompts.
- **Review is strict but not always right**: Codex may claim a fix is incomplete
  based on a theoretical bypass. Verify its claims before trusting blindly.

### Execution
- **API outages happen**: HTTP 500 from Anthropic/OpenAI is transient. Steps fail
  instantly (9s exit) or mid-work (10+ min then crash). Always design for restart.
- **Agent commits even on timeout**: If Claude times out but already committed, the
  work is preserved. Always check `git log` after timeouts.
- **Uncommitted changes on crash**: If API crashes mid-step, check `git diff --stat`.
  The agent may have written good changes but not reached the commit command.
- **Never run parallel agents in same repo**: Codex "rescue" tasks + main pipeline
  in the same git repo = file conflicts and killed processes. Sequential only.
- **50 min timeout is generous**: Most Claude steps finish in 10-25 min. The timeout
  exists for edge cases (large files, slow web searches).

### Prompt Design
- **Prompt specificity matters**: The more specific the fix instructions (exact
  function names, line numbers, code patterns), the better the output.
- **Include "check previous fixes" step**: When running sequential batches, later
  agents need to know what earlier agents changed in shared files.
- **No backward compat = faster fixes**: When breaking changes are OK, agents make
  cleaner fixes instead of compatibility shims.
- **WebSearch instructions work**: Agents actually use WebSearch to look up CWE/OWASP
  best practices. This improves fix quality measurably.

### Pipeline Management
- **Gap closure is a separate phase**: Don't try to fix everything in the automated
  loop. Collect remaining issues, batch them, run focused manual fix→review cycles.
- **Script restartability is essential**: Always support `script.py START END` for
  selective re-execution. You WILL need to restart failed steps.
- **Track per-step verdicts**: Save the last review JSON verdict to a file per step.
  This makes gap collection trivial after the pipeline finishes.
- **Monitor with periodic log checks**: The orchestrator log tells you everything.
  Tail it periodically or set up background pings.
