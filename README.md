# AI Audit Kit

A portable, AI-agent-driven framework for automated codebase security auditing and bug fixing.

Drop the `.audit/` folder into any project, point your AI coding agent at it, and get a structured pipeline that finds, triages, fixes, and reviews hundreds of bugs with minimal human intervention.

## How It Works

```
┌────────────────────────────────────────────────────────────┐
│  Phase 1: AUDIT                                            │
│  AI agent reads every module, finds bugs, writes reports   │
│  Output: .audit/bugs/{severity}/*.md                       │
├────────────────────────────────────────────────────────────┤
│  Phase 2: TRIAGE                                           │
│  Bugs grouped by severity → tracking dashboard created     │
│  Output: .audit/TRACKING.md                                │
├────────────────────────────────────────────────────────────┤
│  Phase 3: FIX (automated pipeline)                         │
│                                                            │
│  for each batch of bugs:                                   │
│    Claude Opus ──fix──▶ commit                             │
│         │                                                  │
│         ▼                                                  │
│    Codex GPT-5.4 ──review──▶ JSON verdict                  │
│         │                                                  │
│    if FAIL: Codex ──fix──▶ commit ──▶ re-review            │
│    if PASS: next batch                                     │
│                                                            │
│  Output: commits + .audit/fix-logs/                        │
├────────────────────────────────────────────────────────────┤
│  Phase 4: REVIEW & HARDEN                                  │
│  Full test suite, manual spot-checks, follow-ups           │
└────────────────────────────────────────────────────────────┘
```

## Quick Start

### 1. Copy into your project

```bash
# Clone this repo
git clone https://github.com/ascorblack/ai-audit-kit.git

# Copy the .audit folder into your project
cp -r ai-audit-kit/.audit /path/to/your/project/
```

### 2. Tell your AI agent to follow the standard

Open your project in Claude Code, Cursor, or any AI coding IDE and say:

> Read `.audit/AUDIT-STANDARD.md` and follow it to audit this project.
> Start with Phase 1.

The agent will:
- Ask you discovery questions about your project (language, toolchain, constraints)
- Systematically audit every module
- Create bug reports in `.audit/bugs/`
- Build the tracking dashboard
- Write and run the fix pipeline
- Review results

### 3. That's it

The standard is designed to be self-contained. The AI agent has everything it needs to build the full pipeline from scratch, adapted to your specific project.

## What's Inside

| File | Purpose |
|------|---------|
| [`.audit/AUDIT-STANDARD.md`](.audit/AUDIT-STANDARD.md) | The standard itself — phases, templates, agent instructions |
| [`.audit/PIPELINE-GUIDE.md`](.audit/PIPELINE-GUIDE.md) | Technical reference — CLI syntax, script patterns, timeouts |

## Requirements

- An AI coding agent (Claude Code, Cursor, Windsurf, etc.)
- [Claude CLI](https://docs.anthropic.com/en/docs/claude-code) — for implementing fixes
- [Codex CLI](https://github.com/openai/codex) — for automated code review
- A git repository to audit

## Supported Tech Stacks

The standard is **language and framework agnostic**. The agent adapts the pipeline to your project's toolchain during the discovery phase. It has been tested with:

- **Python** (pytest, ruff, mypy, uv)
- Should work with any stack where the AI agent knows the test/lint commands

## Pipeline Architecture

The fix pipeline uses a two-model approach for defense in depth:

- **Claude Opus 4.6** (implementer) — reads bug reports, researches best practices, writes fixes and tests, commits
- **Codex GPT-5.4** (reviewer) — reviews each commit, finds gaps, returns structured JSON verdict
- **Review loop** — if the reviewer finds issues, it fixes them itself, then re-reviews (up to 3 iterations)

This catches bugs that a single model would miss. In practice, ~60% of steps require at least one review fix iteration.

## Real-World Results

Built and tested on the [Protocore](https://github.com/ascorblack) ecosystem:

- **363 bugs found** across 2 Python repositories (10 Critical, 76 High, 245 Medium, 32 Low)
- **86 bugs fixed** automatically (10 Critical + 76 High)
- **~250 tests written** by the pipeline
- **Review loop** caught incomplete fixes in ~60% of steps

## Customization

The standard includes discovery questions that the agent asks before building the pipeline. This adapts it to:

- Single or multi-repo projects
- Any package manager (uv, npm, cargo, etc.)
- Any test runner (pytest, jest, go test, etc.)
- Any linting/type-checking setup
- Your commit conventions and quality gates

## License

MIT
