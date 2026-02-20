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
  B1. Analyze issue → extract sub-issue list
  B2. Reproduce each sub-issue (request info if blocked)
  B3. Create GitHub sub-issues for confirmed items
  B4. Document reproduction attempts
  B5. TDD: write failing test → fix → verify
  B6. Document test results
  B7. Code review & security audit
  B8. Create branch, commit, and submit PR

Phase C: Finalize (once)
  C1. ROI report
  C2. Completion promise (if Ralph loop)
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

## B1. Analyze Issue

Read the full issue body and comments:

```bash
gh issue view <NUMBER> --json body,comments
```

Extract a numbered list of distinct sub-issues. A single GitHub issue may contain multiple problems. For each sub-issue, identify:
- **Symptom**: the error message or unexpected behavior
- **Context**: environment, configuration, versions
- **Affected code path**: which components are involved

## B2. Reproduce Each Sub-Issue

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

## B3. Create Sub-Issues

Only create sub-issues for items confirmed through analysis. Each sub-issue must reference the parent issue:

```bash
gh issue create --title "<type>: <description>" --assignee @me --body "Derived from #<PARENT>. ..."
```

Skip sub-issue creation if the parent issue contains only a single problem.

## B4. Document Reproduction

Create `~/.claude/reports/issue-<NUMBER>-report.md`:

```markdown
# Issue #<NUMBER> Reproduction Report

## Sub-issue 1: <title>

### Environment
- OS, runtime versions, relevant config

### Reproduction Steps
1. Step taken
2. Step taken
3. ...

### Result
- **Reproduced**: Yes/No
- **Error output**: (exact message)
- **Attempts**: what was tried if reproduction failed
```

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

## B6. Document Test Results

Append to `~/.claude/reports/issue-<NUMBER>-report.md`:

```markdown
## Fix Verification

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

## B7. Code Review & Security Audit

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

- **Issue found** → fix it before proceeding to B8. Re-run tests after fixing.
- **No issues** → proceed to B8.
- **Uncertain** → flag it in the PR description under a "Security Notes" section for human review.

### Append to report

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

## B8. Branch, Commit, PR

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

**After B8, loop back to B1 for the next issue.**

---

# Phase C: Finalize

Run once after all issues in Phase B are processed.

## C1. ROI Report

Append a summary to `~/.claude/reports/issue-<NUMBER>-report.md` (or a batch report for multiple issues):

```markdown
## ROI Summary

| Metric | Value |
|:-------|:------|
| Issues processed | <N> |
| Resolved | <N> / <N> |
| Blocked (need info) | <N> |
| PRs submitted | <N> |
| Tokens used | <end_tokens - start_tokens> |
```

### Token measurement (end)

Run the same ccusage command from Phase A, compute `end - start` for totalTokens.

### Processing result per issue

```markdown
| Issue | Status | PR | Notes |
|:------|:-------|:---|:------|
| #<N> <title> | resolved / blocked / skipped | #<PR> | <brief> |
```

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

- Output the promise **after** the ROI report (C1), never before.
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
