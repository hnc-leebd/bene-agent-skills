# bene-agent-skills

AI-powered development workflow skills for [Claude Code](https://claude.ai/code).

## Skills

### ai-issue

Automatically analyze, reproduce, and fix GitHub issues using a TDD workflow.

**Features:**
- Fetches unassigned open issues and processes them sequentially
- TDD approach: write failing test → fix → verify → full suite
- Code review & OWASP security audit before PR submission
- ROI reporting with token usage tracking
- Ralph Wiggum loop integration via completion promises

**Usage:**
```
/ai-issue          # Process all unassigned issues
/ai-issue 42       # Process specific issue #42
```

## Installation

```bash
claude plugin add hnc-leebd/bene-agent-skills
```

Or add to your project's `.claude/plugins.json`:

```json
{
  "plugins": ["hnc-leebd/bene-agent-skills"]
}
```

## License

MIT
