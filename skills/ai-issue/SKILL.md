---
name: ai-issue
description: "Automatically fix GitHub issues with TDD approach. Use when the user says '/ai-issue', 'fix issues', 'process open issues', or wants to automatically analyze and fix unassigned GitHub issues. Workflow: fetch unassigned open issues, assign, analyze, reproduce errors, write failing tests, fix, and submit PR."
---

# AI Issue Fix

Automatically analyze, reproduce, and fix GitHub issues using a TDD workflow.

## Workflow Overview

```
Phase A: Setup (once)
  A1. Measure token baseline
  A2. Fetch unassigned open issues
  A3. Select issues to process
  A4. Assign all selected issues

Phase B: Process each issue (loop)
  B1. Record start time, create issue report
  B2. Analyze issue → extract sub-issue list
  B3. Reproduce each sub-issue (request info if blocked)
  B4. Create GitHub sub-issues for confirmed items
  B5. TDD: write failing test → fix → verify
  B6. Code review & security audit
  B7. Create branch, commit, and submit PR
  B8. Close parent issue (if all sub-issues resolved)
  B9. Finalize issue report

Phase C: Finalize (once)
  C1. Batch summary report
  C2. Completion promise (if Ralph loop)
```

---

# Report System

## Report Location

All reports are saved to the **project-local** `.claude/reports/` directory:

```bash
mkdir -p .claude/reports
```

Add `.claude/reports/` to `.gitignore` to avoid commit noise:

```bash
echo '.claude/reports/' >> .gitignore
```

## File Naming

```
.claude/reports/<YYYY-MM-DD>-<NUMBER>-<short-desc>.md
```

Examples:
- `.claude/reports/2026-02-20-201-unicode-sanitization.md`
- `.claude/reports/2026-02-20-202-gpu-detection-logging.md`

**Every issue gets its own report file**, even in batch processing. A separate batch summary is created at Phase C.

Batch summary naming:
```
.claude/reports/<YYYY-MM-DD>-batch-summary.md
```

---

# Phase A: Setup

Run once at the start of the session.

## A1. Token Measurement

Record token baseline at start. Capture the current session's totalTokens:

```bash
npx ccusage session --since <YYYYMMDD> --json 2>&1 | python3 -c "
import json,sys; d=json.load(sys.stdin)
print(d['totals']['totalTokens'])"
```

Save this value. Used in Phase C for cost calculation.

## A2. Fetch Issues

Note: `gh issue list --assignee ""` does NOT reliably filter unassigned issues. Instead, fetch all open issues and filter client-side using jq:

```bash
gh issue list --state open --json number,title,labels,body,assignees --limit 20 \
  | jq '[.[] | select(.assignees | length == 0)]'
```

## A3. Select Issues

Present the list to the user and ask which issues to work on:
- If `/ai-issue <number>` was invoked with a specific issue number, skip selection and use that issue directly.
- If the user says "all" or doesn't specify, process **all** unassigned issues sequentially.
- Otherwise, let the user pick one or more issues from the list.

## A4. Assign Issues

Assign all selected issues immediately to prevent duplicate work:

```bash
gh issue edit <NUMBER> --add-assignee @me
```

---

# Phase B: Process Each Issue

**Repeat for each selected issue.** If an issue is blocked, document it and move to the next.

## B1. Start Issue Report

Record the start time and create the issue report file:

```bash
mkdir -p .claude/reports
```

Create `.claude/reports/<YYYY-MM-DD>-<NUMBER>-<short-desc>.md`:

```markdown
# Issue #<NUMBER>: <title>

- **Start**: <YYYY-MM-DD HH:MM>
- **Status**: in progress
- **Issue URL**: <link>
```

## B2. Analyze Issue

Read the full issue body and comments:

```bash
gh issue view <NUMBER> --json body,comments
```

Extract a numbered list of distinct sub-issues. A single GitHub issue may contain multiple problems. For each sub-issue, identify:
- **Symptom**: the error message or unexpected behavior
- **Context**: environment, configuration, versions
- **Affected code path**: which components are involved

Append analysis to the issue report.

## B3. Reproduce Each Sub-Issue

For each sub-issue, attempt reproduction **before writing any fix**.

**Reproducible locally** → write a minimal reproduction script or test, record the exact error output.

**Blocked — need specific files** → post a comment on the issue requesting the file, then continue with other sub-issues:

```bash
gh issue comment <NUMBER> --body "> 🤖 *AI Issue Fix*

Could you send [file] to open.dataloader@hancom.com so we can reproduce this issue?"
```

**Blocked — need system access/permissions or additional info from reporter** → post a comment on the issue with specific questions, then continue with other sub-issues. Do NOT block on a response — mark this sub-issue as `blocked` and proceed:

```bash
gh issue comment <NUMBER> --body "> 🤖 *AI Issue Fix*

To investigate further, we need: [specific info]. Could you provide this?"
```

**Need developer input on approach** → post a comment with options, mark as `blocked`, continue:

```bash
gh issue comment <NUMBER> --body "> 🤖 *AI Issue Fix*

@<developer> Question: [specific question about approach/architecture].

Options considered:
1. [option A]
2. [option B]

Which approach do you prefer?"
```

**Cannot reproduce but understand the code path** → document what was tried, explain the theoretical cause based on code reading, and note that full reproduction requires the reporter's environment/files.

Append reproduction results to the issue report:

```markdown
## Reproduction

### Sub-issue 1: <title>

#### Environment
- OS, runtime versions, relevant config

#### Steps
1. Step taken
2. Step taken

#### Result
- **Reproduced**: Yes/No
- **Error output**: (exact message)
```

## B4. Create Sub-Issues

Only create sub-issues for items confirmed through analysis. Each sub-issue must reference the parent issue:

```bash
gh issue create --title "<type>: <description>" --assignee @me --body "Derived from #<PARENT>. ..."
```

Skip sub-issue creation if the parent issue contains only a single problem.

## B5. TDD Fix

For each reproducible sub-issue:

1. **Write a failing test first** — the test must demonstrate the bug
2. **Run the test** — confirm it fails with the expected error
3. **Implement the fix** — minimal change to resolve the issue
4. **Run the test again** — confirm it passes
5. **Run the full test suite** — confirm no regressions

Test locations by language:
- Java: `cd java && mvn test -Dtest=<TestClass>#<testMethod>`
- Python: `python -m pytest <test_file>::<TestClass>::<test_method> -v`
- Full suite: `./scripts/test-java.sh` or `python -m pytest`

Append test results to the issue report:

```markdown
## Fix

### Sub-issue 1: <title>

#### Failing test (before fix)
- Test: `<test file>:<test name>`
- Result: FAIL — <error message>

#### Fix applied
- File: `<path>`
- Change: <brief description>

#### Passing test (after fix)
- Result: PASS
- Full suite: PASS (no regressions)
```

## B6. Code Review & Security Audit

Before creating a PR, perform a self-review of all changes. This step catches issues that tests alone cannot.

### Code Review Checklist

Review every changed file using `git diff` and verify:

1. **Correctness** — Does the change actually fix the issue? Are there edge cases missed?
2. **Scope** — No unrelated changes, no leftover debug code, no commented-out code
3. **Code style** — Follows existing patterns in the codebase (naming, formatting, imports)
4. **Error handling** — Failures are handled gracefully, not silently swallowed
5. **Backwards compatibility** — Existing APIs/behavior not broken without intention

### Security Checklist (OWASP Top 10)

Scan all changes for vulnerabilities:

| Check | What to look for |
|:------|:-----------------|
| **Injection** | User input used in SQL, shell commands, file paths, or template strings without sanitization |
| **Broken Auth** | Hardcoded credentials, tokens, API keys in code or config |
| **Sensitive Data** | Secrets, PII, or internal paths exposed in logs, error messages, or comments |
| **XXE / Deserialization** | Untrusted XML/JSON parsed without validation; unsafe deserialization |
| **Path Traversal** | User-controlled file paths without canonicalization (`../` attacks) |
| **SSRF** | User-controlled URLs passed to HTTP clients without allowlist validation |
| **Dependency Risk** | New dependencies added? Check for known CVEs. Pinned versions preferred |

### Actions

- **Issue found** → fix it before proceeding to B7. Re-run tests after fixing.
- **No issues** → proceed to B7.
- **Uncertain** → flag it in the PR description under a "Security Notes" section for human review.

### Append to issue report

```markdown
## Code Review & Security Audit

### Files reviewed
- `<path>` — <brief summary of change>

### Findings
- [ ] No issues found
- [ ] Fixed: <description of issue found and fixed>
- [ ] Flagged for review: <description of concern>

### Security scan
- Injection: PASS
- Credentials/secrets: PASS
- Path traversal: PASS
- Dependency risk: PASS / N/A
```

## B7. Branch, Commit, PR

```bash
# Branch naming
git checkout -b issue/<NUMBER>-<short-desc>

# Conventional commit (English)
git commit -m "<type>: <description>

Fixes #<NUMBER>"

# Push and create PR
git push -u origin issue/<NUMBER>-<short-desc>
```

PR body template:

```markdown
## Summary
Fixes #<NUMBER>

<1-3 bullet points>

## Environment
<OS, runtime, relevant config>

## Reproduction
<Steps to reproduce the original error>

## Test
<Test name and what it verifies>

## Result
- Before: <error>
- After: <passing test output>
```

## B8. Close Parent Issue

After PR submission, check if the parent issue should be closed.

**All sub-issues resolved** → comment on the parent issue and close it:

```bash
gh issue comment <NUMBER> --body "> 🤖 *AI Issue Fix*

All sub-issues have been resolved:
<list of sub-issues with PR links>

Closing this issue."

gh issue close <NUMBER>
```

**Some sub-issues blocked** → comment with progress summary, keep open:

```bash
gh issue comment <NUMBER> --body "> 🤖 *AI Issue Fix*

Progress update:
- Resolved: <list with PR links>
- Blocked: <list with reasons>

Keeping this issue open for remaining items."
```

**Single issue (no sub-issues)** → the `Fixes #<NUMBER>` in the PR handles auto-close on merge. No action needed.

## B9. Finalize Issue Report

Record end time and update the issue report header:

```markdown
# Issue #<NUMBER>: <title>

- **Start**: <YYYY-MM-DD HH:MM>
- **End**: <YYYY-MM-DD HH:MM>
- **Duration**: <Xm>
- **Status**: resolved / blocked / partial
- **PR**: #<PR_NUMBER>
- **Issue URL**: <link>
```

**After B9, loop back to B1 for the next issue.**

---

# Phase C: Finalize

Run once after all issues in Phase B are processed.

## C1. Batch Summary Report

Create `.claude/reports/<YYYY-MM-DD>-batch-summary.md`:

```markdown
# AI Issue Fix — Batch Summary

- **Date**: <YYYY-MM-DD>
- **Session start**: <HH:MM>
- **Session end**: <HH:MM>
- **Total duration**: <Xh Xm>

## Results

| Issue | Title | Status | Duration | PR | Notes |
|:------|:------|:-------|:---------|:---|:------|
| #<N> | <title> | resolved / blocked | <Xm> | #<PR> | <brief> |

## Token Usage

| Metric | Value |
|:-------|:------|
| Start tokens | <N> |
| End tokens | <N> |
| Tokens used | <delta> |

## Summary

| Metric | Value |
|:-------|:------|
| Issues processed | <N> |
| Resolved | <N> / <N> |
| Blocked (need info) | <N> |
| PRs submitted | <N> |
```

### Token measurement (end)

Run the same ccusage command from Phase A, compute `end - start` for totalTokens.

## C2. Completion Promises (Ralph Wiggum Integration)

When running inside a Ralph loop (`/ralph-loop`), output exactly ONE of these promises after C1 to signal termination:

| Promise | Meaning |
|:--------|:--------|
| `<promise>AI_ISSUE_DONE</promise>` | All issues processed (resolved or blocked). No more unassigned issues remain. |
| `<promise>AI_ISSUE_BLOCKED</promise>` | All remaining issues are blocked (need info, files, or developer input). Cannot proceed further without human intervention. |
| `<promise>AI_ISSUE_PARTIAL</promise>` | Some issues resolved, but iteration limit or error prevented completing the rest. Resume with `/ai-issue` to continue. |

### Usage with Ralph

```bash
/ralph-loop "/ai-issue" --completion-promise "AI_ISSUE_DONE" --max-iterations 30
```

### Rules

- Output the promise **after** the batch summary report (C1), never before.
- If no unassigned issues exist at A2, output `<promise>AI_ISSUE_DONE</promise>` immediately.
- Always set `--max-iterations` as a safety net when using Ralph.

---

# Decision Points

**Multiple sub-issues in one issue?**
→ One PR per sub-issue if they touch different code paths. One combined PR if closely related.

**Cannot reproduce?**
→ Do NOT write speculative fixes. Comment on the issue requesting more information. Document what was tried.

**Fix requires architectural decision?**
→ Present options to the user before implementing. Do not pick an approach unilaterally.

**Benchmark-sensitive changes?**
→ Run `./scripts/bench.sh --check-regression` before submitting PR.
