# Changelog

All notable changes to the files in this repo are logged here, newest first.

## April 20, 2026 (evening)

### `playbook.md` rewrite

Replaced the stub with real content.

**Kept.**

- Behavioral Guidelines (minor cleanup).
- Escalation Rules.
- Comment-to-DM Rules (rewritten, see below).
- Security section (prompt injection defense preserved).

**Added.**

- Explicit flag lifecycle paragraph in Escalation Rules. Annotated conversation history, RESOLVED and IGNORED flags treated as closed.
- Self-Echo Filter as its own short section. Drop webhooks where sender is the business Meta ID.

**Removed.**

- Cooldown language (60-second per-contact window after sending). Debounce on incoming messages handles the rapid-fire case without dropping real substantive follow-ups.
- References to a resource repository. Comment-to-DM rules now use post context and the configured CTA directly.

### `integration-notes.md` rewrite

Replaced the stub. Substantially cut from the prior shape.

**Removed.**

- The numbered tool list. The MCP schema is self-describing and authoritative. A mirror always lags.
- The incorrect "cannot reply publicly to comments" limitation.
- Known Limitations section (stale).
- System Requirements section (belongs in the kickoff's dashboard step, not here).
- 60-second cooldown recommendation.
- Tools vs Resources Quick Reference (redundant with the schema).

**Kept.**

- Webhook subscription explanation.
- Standard workflow for DMs.
- Debounce framing.

**Added.**

- Standard workflow for comment-triggered DMs.
- Meta platform quirks section covering the 24-hour messaging window, 7-day CTD window, self-echo, 400-that-delivered, and one-public-reply-per-comment.
- A short note explaining why this file doesn't mirror the schema.

### `kickoff-prompt.md` (no version bump)

- Added a "What's new in v1.1" callout at the top of the file, pointing readers to this CHANGELOG for the full diff.

## April 20, 2026

### `kickoff-prompt.md` v1.1

Rewritten.

**Cut.**

- Changelog block at the top of the file. Version history lives in this CHANGELOG now.
- Resource Repository section. Operators manage resources inside their agent's configuration instead of a separate file.
- Webhook polling fallback path. Webhook subscription is live and required.
- Inline Cloudflare Tunnel setup instructions. Moved to Module 1 of the course.

**Added.**

- Step 4 on keeping the agent alive, covering launchd, systemd, Railway, Fly.io, and VPS paths.
- `NODE_EXTRA_CA_CERTS` guidance for Node under macOS launchd.
- `dotenv.config({ override: true })` pattern for Claude Desktop environments on macOS.
- Allowlist SQLite schema and "Add to Allowlist" button requirement in the Inbox view.
- Flag lifecycle with annotated conversation history.
- Self-echo filter (drop webhooks from the business Meta ID).
- Synthetic webhook smoke test as Step 7.
- Rating loop (👍/👎 on every AI response) with 14-day quality trend.
- Cost cap widget with daily hard cap and hourly alert threshold.

**Rewritten.**

- Step 9 reframed from "Go Live" to "Expand the Allowlist." SAFE mode with curated allowlist is positioned as the steady state.
- Contact filter simplified to SAFE mode plus allowlist. The five-option menu (all inbound, by assignee, by label, by prior outbound, custom) was removed.
- Autopilot three-state description sharpened.
- Debounce kept. Per-contact cooldown removed.

## April 19, 2026

- Initial repo structure created.
- Placeholder files for `kickoff-prompt.md`, `playbook.md`, `integration-notes.md`.
- README em dashes removed.
