# prd-tech-analysis

A Claude Code skill that turns a product PRD (Notion link or text) into a codebase-grounded technical analysis doc — API, payloads, architecture, the actual files/changes, open questions, and cross-service impact. It produces the analysis only; it does not write implementation code.

## Install

Clone into your Claude Code skills directory:

```bash
git clone <repo-url> ~/.claude/skills/prd-tech-analysis
```

Then use it via `/prd-tech-analysis` or by sharing a PRD and asking for a technical analysis.

See [SKILL.md](SKILL.md) for the full workflow.
