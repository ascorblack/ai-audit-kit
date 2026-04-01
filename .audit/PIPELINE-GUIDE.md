# Bug Fix Pipeline Guide

Reference for building automated fix pipelines using Claude CLI + Codex CLI.
Written from practical experience fixing 86 bugs across protocore ecosystem.

## Architecture

```
fix-script.py
  |
  for each step:
  |   1. Claude Opus 4.6 (high) -- implements fix + tests + commit
  |   2. Codex GPT-5.4 -- reviews the commit (JSON verdict)
  |   3. if FAIL: Codex GPT-5.4 -- fixes issues + commit
  |   4. goto 2 (max 3 iterations)
  |   5. next step
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

Both tools inherit proxy from environment. Ensure these are set:
```bash
export http_proxy=http://127.0.0.1:10809
export https_proxy=http://127.0.0.1:10809
export HTTP_PROXY=http://127.0.0.1:10809
export HTTPS_PROXY=http://127.0.0.1:10809
```

Pass env to subprocess: `env=os.environ.copy()`

## Script Structure

### 1. Configuration Block

```python
WORKSPACE = Path("/home/dev/ascorblack-labs")
LOG_DIR = WORKSPACE / ".audit" / "fix-logs-<name>"
TRACKING = WORKSPACE / ".audit" / "TRACKING.md"

CLAUDE_BIN = "claude"
CLAUDE_MODEL = "claude-opus-4-6"
CLAUDE_EFFORT = "high"
STEP_TIMEOUT = 3000  # 50 min per step

CODEX_BIN = "codex"
CODEX_MODEL = "gpt-5.4"
CODEX_REVIEW_TIMEOUT = 600   # 10 min for review
CODEX_FIX_TIMEOUT = 1800     # 30 min for fix
MAX_REVIEW_ITERATIONS = 3
```

### 2. System Prompt (appended to Claude)

```python
APPEND_SYSTEM = """\
You are a senior software engineer fixing {SEVERITY} bugs in Protocore.

Mandatory workflow:
1. Read ALL bug report files listed in the task.
2. Read affected source files and existing tests.
3. Research best practices: WebSearch CWE/OWASP IDs.
4. Implement fix following existing code style.
5. Write comprehensive tests.
6. Run pytest, ruff, mypy. Fix ALL issues.
7. Update broken tests if they relied on buggy behavior.
8. Git commit with descriptive message.

Key info:
- protocore/ (core): cd protocore && uv run pytest/ruff/mypy
- protocore-enterprise/: cd protocore-enterprise && uv run pytest/ruff/mypy
- Python 3.12+, strict mypy, ruff, uv
"""
```

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
def review_and_fix_loop(name, bug_ids):
    for i in range(1, MAX_ITERATIONS + 1):
        verdict = codex_review(name, bug_ids)  # returns {"pass": bool, "issues": [...]}
        if verdict["pass"]:
            return  # clean
        codex_fix(name, bug_ids, verdict["issues"])
    # max iterations reached — move on
```

### 7. Step Execution

```python
for step_name, prompt, bug_ids in STEPS:
    success = run_claude_step(step_name, prompt, bug_ids)
    if success:
        review_and_fix_loop(step_name, bug_ids)
```

## Bug Grouping Strategy

Group bugs by:
1. **Repository** — core vs enterprise (separate git histories)
2. **Shared files** — bugs touching the same file MUST be in the same batch
3. **Domain** — related functionality together (shell safety, subagent, etc.)
4. **Size** — 4-8 bugs per batch (sweet spot for 50-min timeout)

Sequential execution eliminates cross-batch conflicts.

## Timeouts (empirical)

| Operation | Timeout | Notes |
|-----------|---------|-------|
| Claude fix (4-6 bugs) | 50 min | Includes research, impl, tests, validation |
| Claude fix (7-10 bugs) | 50 min | Same — agent parallelizes internally |
| Codex review | 10 min | Reads diff, runs checks, outputs JSON |
| Codex fix | 30 min | Implements review feedback, runs tests |
| Full step with review | ~50-70 min | Fix + 1-2 review cycles typical |

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

- **Codex finds real gaps**: The review loop catches incomplete fixes, missing edge
  cases, and quality gate failures that Claude missed. Worth the extra time.
- **3 review iterations is enough**: Most issues resolve in 1-2 cycles. If still
  failing at 3, the bug is likely too complex for automated fixing.
- **Timeouts are forgiving**: 50 min for Claude is generous. Most steps finish in
  15-25 min. Timeouts exist for edge cases, not the norm.
- **Agent commits even on timeout**: If Claude times out but already committed, the
  work is preserved. Check git log after timeouts.
- **Sequential > parallel**: For bugs in the same repo, sequential execution avoids
  merge conflicts and lets each agent see prior fixes.
- **Prompt specificity matters**: The more specific the fix instructions (exact
  function names, line numbers, code patterns), the better the output.
- **No backward compat = faster fixes**: When breaking changes are OK, agents make
  cleaner fixes instead of compatibility shims.
