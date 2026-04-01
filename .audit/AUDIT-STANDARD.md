# Automated Bug Audit & Fix Standard

> A portable standard for AI-assisted security auditing and automated bug fixing.
> Drop this file + `PIPELINE-GUIDE.md` into any project's `.audit/` folder.
> An AI agent (Claude Code, Cursor, etc.) can follow this to build the full pipeline.

## What This Is

A structured approach to:
1. **Audit** a codebase for bugs (security, reliability, correctness)
2. **Triage** findings by severity (Critical / High / Medium / Low)
3. **Fix** bugs automatically via AI agent pipeline with review loops
4. **Track** progress across hundreds of bugs with status dashboards

The standard is language/framework agnostic.

## Folder Structure

```
.audit/
  AUDIT-STANDARD.md        # This file — the standard itself
  PIPELINE-GUIDE.md        # Technical reference: CLI syntax, script patterns, timeouts
  TRACKING.md              # Master dashboard: all bugs, grouped by severity, with status
  bugs/                    # One markdown file per bug
    critical/
      001-short-name.md
      002-short-name.md
    high/
      001-MODULE-ID-short-description.md
      ...
    medium/
      ...
    low/
      ...
  fix-critical.py          # Orchestrator script for critical bugs
  fix-high.py              # Orchestrator script for high bugs
  fix-logs-critical/       # Logs from critical fix run
  fix-logs-high/           # Logs from high fix run
    orchestrator.log       # Timestamped status lines
    step-NN-name.log       # Per-step Claude output
    step-NN-name-review.log        # Per-step Codex review output
    step-NN-name-review-output.txt # Codex JSON verdict
    step-NN-name-fix.log           # Codex fix output
```

---

## Phase 1: Audit

### Goal
Produce a complete inventory of bugs as individual markdown files in `.audit/bugs/`.

### How to Run

Ask your AI agent:

> Perform a comprehensive security and reliability audit of this project.
> For each bug found, create a markdown file in `.audit/bugs/{severity}/`
> following the bug report template below. Focus on: security vulnerabilities
> (OWASP Top 10, CWE), reliability issues, correctness bugs, data loss risks.

### Bug Report Template

Each file in `.audit/bugs/{severity}/` should follow this format:

```markdown
# {SEVERITY}-{NUMBER}: {Title}

## Summary
{2-3 sentence description of the vulnerability/bug}

## Severity
{Critical / High / Medium / Low}

## Category
{Security / Reliability / Correctness / Performance}

## CWE / OWASP
{CWE-XXX, OWASP AXX if applicable}

## Location
- **File:** `path/to/file.py`
- **Lines:** {start}-{end}
- **Function:** `function_name()`

## Root Cause
{Why the bug exists — what assumption is violated}

## Impact
{What happens if exploited / triggered}

## Reproduction
{Steps or code to trigger the bug}

## Suggested Fix
{Concrete fix approach with code sketch if helpful}

## References
{Links to CWE, OWASP, relevant documentation}
```

### Naming Convention

```
{NNN}-{MODULE}-{ID}-{short-description}.md

Examples:
  001-SAFETY-001-no-deny-for-server-processes.md
  045-sandbox-shell-env-var-injection.md
  V001-APPROVAL-RULE-REGEX-SEARCH-NOT-FULLMATCH.md
```

- `NNN` — sequential number within severity
- `MODULE` — affected module/area (optional but helpful)
- `ID` — internal bug ID if you have a tracker
- Use `V` prefix for bugs found in later validation passes

---

## Phase 2: Triage & Tracking

### Goal
Create `TRACKING.md` — a master dashboard with all bugs and their status.

### Template

```markdown
# Project — Bug Fix Tracking

> Generated: {date}
> Total: {N} bugs ({n1} Critical, {n2} High, {n3} Medium, {n4} Low)

| Symbol | Meaning     |
| ------ | ----------- |
| ⏳      | Pending     |
| 🔄     | In Progress |
| ✅      | Fixed       |
| ⚠️     | Partial     |
| ❌      | Blocked     |

---

## Critical ({n1})

| # | ID | Title | Location | Status |
|---|-----|-------|----------|--------|
| 1 | CRIT-001 | ... | `file.py:line` | ⏳ |

## High ({n2})

| # | ID | Status |
|---|-----|--------|
| 1 | 001-MODULE-ID-description | ⏳ |

## Medium ({n3})
...

## Low ({n4})
...
```

---

## Phase 3: Fix Pipeline

### Goal
Write orchestrator scripts that invoke AI agents to fix bugs batch-by-batch,
with automated code review after each batch.

### Discovery Questions

Before writing the script, the agent should ask / determine:

1. **Project structure**
   - Single repo or multi-repo?
   - What are the repo paths?
   - What is the dependency direction between repos?

2. **Toolchain**
   - Package manager? (uv, pip, npm, cargo, etc.)
   - Test runner? (pytest, jest, cargo test, etc.)
   - Linter? (ruff, eslint, clippy, etc.)
   - Type checker? (mypy, tsc, etc.)
   - Security scanner? (bandit, npm audit, etc.)

3. **Quality gates**
   - Coverage threshold?
   - Which checks must pass before committing?

4. **AI tools available**
   - Claude CLI installed? (`claude --version`)
   - Codex CLI installed? (`codex --version`)
   - Proxy required? (`echo $http_proxy`)
   - Which models available? Test with simple prompts.

5. **Constraints**
   - Backward compatibility needed?
   - Any files/areas off-limits?
   - Preferred commit message format?
   - Max time budget?

6. **Bug grouping**
   - Which bugs share files? (must be in same batch)
   - Which bugs share domain? (should be in same batch)
   - Which repo does each bug target?

### Grouping Rules

1. **Same file → same batch** — two bugs in `shell.py` must be fixed together
2. **Same domain → same batch** — related bugs benefit from shared context
3. **Same repo → contiguous steps** — all core steps before all enterprise steps
4. **4-8 bugs per batch** — sweet spot for agent context and timeout
5. **Sequential execution** — eliminates merge conflicts between batches

### Pipeline Architecture

```
for each step:
  ┌─────────────────────────────────────┐
  │  1. Claude Opus (high effort)       │
  │     - Reads bug reports             │
  │     - Researches best practices     │
  │     - Implements fixes              │
  │     - Writes tests                  │
  │     - Runs quality gates            │
  │     - Commits                       │
  │     Timeout: 50 min                 │
  └──────────────┬──────────────────────┘
                 │ success
  ┌──────────────▼──────────────────────┐
  │  2. Codex GPT-5.4 (review)         │
  │     - Reads commit diff             │
  │     - Evaluates completeness        │
  │     - Checks for regressions        │
  │     - Returns JSON verdict          │
  │     Timeout: 10 min                 │
  └──────────────┬──────────────────────┘
                 │
          ┌──────┴──────┐
          │ pass?       │
          ▼             ▼
        [yes]         [no]
          │             │
          │   ┌─────────▼───────────────┐
          │   │  3. Codex GPT-5.4 (fix) │
          │   │     - Fixes issues      │
          │   │     - Runs tests        │
          │   │     - Commits           │
          │   │     Timeout: 30 min     │
          │   └─────────┬───────────────┘
          │             │
          │             ▼
          │         goto step 2
          │         (max 5 iterations)
          │
          ▼
       next step
```

### Prompt Engineering

#### Claude Fix Prompt — must include:

| Section | Purpose | Example |
|---------|---------|---------|
| **Read First** | Exact file paths for agent to read | Bug reports + source + tests |
| **Research** | WebSearch queries for best practices | `"CWE-78 command injection prevention"` |
| **Bug Fixes** | Per-bug fix instructions | What to change, where, expected result |
| **Tests** | Test file name + test list | `test_high_shell_deny.py`: test_1, test_2... |
| **Validate** | Exact shell commands | `uv run pytest`, `uv run ruff check` |
| **Commit** | Exact commit command | `git commit -m "fix(area): ..."` |

#### Codex Review Prompt — must:
- Reference the exact commit SHA
- List the bug IDs being reviewed
- Ask specific evaluation questions
- Require JSON output: `{"pass": true/false, "issues": [...]}`

#### Codex Fix Prompt — must:
- List the exact issues from the review
- Reference the bug IDs and step name
- Instruct to run tests and quality gates
- Specify commit message format

### System Prompt Pattern

The `--append-system-prompt` should set:
- **Role**: "senior security engineer" / "senior software engineer"
- **Workflow**: numbered steps the agent must follow
- **Project context**: repo layout, package managers, commands
- **Constraints**: import boundaries, coverage thresholds, conventions

---

## Phase 3.5: Gap Closure

### Goal
Close remaining issues from steps that hit the max review iteration limit.

### Process
1. Collect last review verdicts from steps that didn't reach PASS
2. Batch related gaps (e.g., all regex issues together, all test gaps together)
3. Run manual Codex fix → Codex review loop until PASS
4. Restart any steps that failed due to API errors or timeouts

> **Important:** Do not run gap closure in parallel with the main pipeline.
> Both modify the same repo, causing conflicts and killed processes.
> Wait for the pipeline to stop, fix gaps, then restart remaining steps.

---

## Phase 4: Review & Hardening

### Goal
After all automated fixes, perform manual review and run full test suites.

### Checklist

- [ ] Run full test suite in each repo
- [ ] Run all quality gates (lint, type check, security scan)
- [ ] Review git log — each commit message matches its changes
- [ ] Spot-check 20% of fixes for correctness
- [ ] Check for unintended side effects (new imports, changed APIs)
- [ ] Verify steps that hit max review iterations have been gap-closed
- [ ] Update TRACKING.md with final statuses
- [ ] Identify follow-up items from review

---

## Quick Start

If you're an AI agent reading this, here's what to do:

### Step 1: Understand the project
```
Read the project's README, CLAUDE.md, or equivalent.
Understand: language, repos, test commands, quality gates.
```

### Step 2: Run the audit
```
Systematically read each module.
For each bug found, create .audit/bugs/{severity}/{NNN}-{name}.md
```

### Step 3: Create tracking
```
Generate .audit/TRACKING.md from the bug inventory.
```

### Step 4: Ask the human
```
Present the bug counts by severity.
Ask which severities to fix (usually: Critical first, then High).
Ask about constraints (backward compat, time budget, etc.).
```

### Step 5: Write the orchestrator
```
Read PIPELINE-GUIDE.md for CLI syntax and script patterns.
Group bugs into batches following the grouping rules.
Write the Python orchestrator script.
Test CLI calls before running.
```

### Step 6: Run and monitor
```
Run the script. Monitor via orchestrator.log.
After completion, run full test suites.
Review changes. Implement follow-ups.
```

---

## Adapting to Other Projects

| Aspect | What to Change |
|--------|----------------|
| Language | Swap pytest/ruff/mypy for equivalent tools |
| Repos | Adjust paths and git commands |
| Models | Use available models (Claude Opus, GPT-5.4, etc.) |
| Quality gates | Match project's CI requirements |
| Bug categories | Add project-specific categories (e.g., accessibility, i18n) |
| Commit style | Match project's conventional commits or PR workflow |

The core pattern — **audit → triage → batch fix → review loop → harden** —
is universal regardless of tech stack.
