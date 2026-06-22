# prd-to-backend-analysis

A Claude Code skill that turns a PRD (Notion link) into a **backend tech-analysis doc** written into a new private Notion page — for manager + FE review and later referenced during development. Backend only: it specifies the FE contract (routes + socket payloads) but does not design FE UX, and it stops at the analysis (no implementation).

It runs autonomously: explores the codebase via a fan-out, derives sync-vs-async from the actual code path, drafts the doc into Notion, then loops a multi-lens review to validate-and-fill.

## Install

Clone into your Claude Code skills directory:

```bash
git clone <repo-url> ~/.claude/skills/prd-to-backend-analysis
```

Then use it via `/prd-to-backend-analysis <prd-link>`, or by giving a PRD Notion link and asking for a backend tech analysis.

See [SKILL.md](SKILL.md) for the full workflow.
