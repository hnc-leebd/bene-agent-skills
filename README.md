# Skills

AI-powered development workflow skills for Claude. Automate repetitive engineering tasks with structured, reproducible workflows.

For more information about the skills system, see:
- [What are skills?](https://support.claude.com/en/articles/12512176-what-are-skills)
- [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [Agent Skills standard](http://agentskills.io)

# About This Repository

This repository contains skills that automate software development workflows. Each skill is self-contained in its own folder with a `SKILL.md` file.

## Skills

| Skill | Description |
|:------|:------------|
| [ai-issue](./skills/ai-issue) | Automatically analyze, reproduce, and fix GitHub issues using a TDD workflow |

### ai-issue

Fetches unassigned open GitHub issues, analyzes them, reproduces bugs, writes failing tests, implements fixes, and submits PRs — all automatically.

**Highlights:**
- 3-phase workflow: Setup → Issue loop → Finalize
- TDD approach: write failing test → fix → verify → full suite
- Code review & OWASP security audit before PR submission
- ROI reporting with token usage tracking
- [Ralph Wiggum](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum) loop integration via completion promises

# Try in Claude Code

## Install from Marketplace

Register this repository as a Claude Code Plugin marketplace:

```
/plugin marketplace add hnc-leebd/bene-agent-skills
```

Then install the plugin:

```
/plugin install bene-agent-skills@bene-agent-skills
```

Or browse and install interactively:
1. Run `/plugin` and select `Browse and install plugins`
2. Select `bene-agent-skills`
3. Select `Install now`

## Auto-Update

Third-party marketplaces don't auto-update by default. To enable:

```
/plugin marketplace update bene-agent-skills
```

Or enable auto-update permanently in `~/.claude/settings.json`:

```json
{
  "marketplaceAutoUpdate": {
    "bene-agent-skills": true
  }
}
```

## Usage

After installing, just invoke the skill:

```
/ai-issue          # Process all unassigned issues
/ai-issue 42       # Process specific issue #42
```

## Uninstall

```
/plugin uninstall bene-agent-skills@bene-agent-skills
/plugin marketplace remove bene-agent-skills
```

# License

MIT
