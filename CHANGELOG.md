# Changelog

All notable changes to the files in this repo are logged here, newest first.

## April 20, 2026

### `kickoff-prompt.md` v1.1

Rewritten based on debrief from the Carl reference implementation (April 2026).

**Cut.**

- Changelog block at top of file (history lives here in CHANGELOG.md instead)
- Resource Repository section (operators manage resources inside their agent's config, not as a separate file)
- Webhook polling fallback path (webhook subscription is live and required)
- Inline Cloudflare Tunnel setup instructions (moved to Module 1 of the course)

**Added.**

- Step 4 on keeping the agent alive, covering launchd, systemd, Railway, Fly.io, and VPS
- `NODE_EXTRA_CA_CERTS` instructions for Node under macOS launchd
- `dotenv.config({ override: true })` pattern for Claude Desktop environments on macOS
- Allowlist SQLite schema and "Add to Allowlist" button requirement in the Inbox view
- Flag lifecycle with annotated conversation history
- Self-echo filter (drop webhooks from the operator's business Meta ID)
- Synthetic webhook smoke test as Step 7
- Rating loop (👍/👎 on every AI response) with 14-day quality trend
- Cost cap widget with daily hard cap and hourly alert threshold

**Rewritten.**

- Step 9 reframed from "Go Live" to "Expand the Allowlist." SAFE mode with curated allowlist is now positioned as the steady state, not a temporary stepping stone to ON.
- Contact filter simplified to SAFE mode plus allowlist. The five-option menu (all inbound / by assignee / by label / by prior outbound / custom) was removed.
- Autopilot three-state description sharpened. SAFE is no longer described as training wheels.
- Debounce kept. Per-contact cooldown removed (dropped real follow-ups).

## April 19, 2026

- Initial repo structure created
- Placeholder files for `kickoff-prompt.md`, `playbook.md`, `integration-notes.md`
- README em dashes removed per style rule
