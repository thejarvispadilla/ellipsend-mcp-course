# Ellipsend MCP Course — Companion Repo

This repo holds the files students hand to their AI tool (Claude Code, OpenClaw, Codex, or similar) as part of the Ellipsend MCP DIY Course.

## What's in here

- **`kickoff-prompt.md`** — One-shot setup guide. Walks your AI tool through connecting to Ellipsend, building a local dashboard, and configuring your agent. Use once at the start.
- **`playbook.md`** — Persistent runtime config. Your AI tool loads this every session. Defines behavior, escalation rules, and security guardrails.
- **`integration-notes.md`** — Webhook subscription flow, workflow patterns, and Meta platform quirks your agent needs to handle.
- **`CHANGELOG.md`** — What's changed in each version.

## How to use this with your AI tool

### Option 1 (recommended): Hand the repo URL to your tool

Most agentic AI tools can read GitHub repos directly. Paste this URL into a fresh session:

```
https://github.com/thejarvispadilla/ellipsend-mcp-course
```

Then say something like: *"Read the files in this repo and walk me through setup."* Your tool pulls the latest version of every file automatically.

### Option 2: Download the files

Clone the repo and point your AI tool at the local copy.

```bash
git clone https://github.com/thejarvispadilla/ellipsend-mcp-course.git
```

## Versioning

When a file changes meaningfully, the version in its header bumps and the change is logged in `CHANGELOG.md`. Check there if your setup was based on an older version and something stops working.

## The course

Course lives on Notion. [Link to course home] <!-- update with public Notion URL -->
