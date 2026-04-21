# Changelog

All notable changes to the files in this repo are logged here, newest first.

## April 21, 2026 (afternoon)

### `kickoff-prompt.md` v1.3

Autopilot mode framing corrected. The v1.2 framing presented SAFE and ON as parallel steady-state modes, which doesn't match the intended product use. Corrected to:

- **OFF.** Paused. Setup only.
- **SAFE.** Testing only. Used in Module 3 and during a brief cautious-launch window.
- **ON.** Production. The steady state for every operator.

**Rewritten.**

- Step 5 autopilot descriptions no longer call SAFE a steady state.
- Step 9 renamed from "Go Live" to "Go Live on ON." The "two valid modes" framing is gone. SAFE is explicitly positioned as testing and cautious-launch, not a production endpoint. Go-live checklist covers the five preconditions for flipping to ON. After-flip section covers the first 48-hour review cadence.
- Allowlist description in Step 5 now frames the allowlist as a testing and cautious-launch tool, not a production gate.
- v1.2 callout at the top replaced with v1.3 callout summarizing the mode-framing correction.

## April 21, 2026

### `kickoff-prompt.md` v1.2

Launch-mode and webhook-coverage rework based on operator feedback and webhook-event testing.

**Added.**

- `new_button_clicked` is now part of the webhook subscription in Step 3. Without it, your agent is deaf to button and quick-reply taps, which breaks any coordination with DM automations. Agent webhook handler now needs a branch for all four event types.
- Step 1 now asks whether you run DM automations in Ellipsend. Context gets saved so the handoff rule can be configured in Step 6.
- Step 6 has a new "Exclusion rules" subsection for defining categories of contacts the agent should stay out of (relationships, labels, any metadata the operator wants to use).
- Step 6 has a new "Automation handoff" subsection for describing where in each DM automation the agent should take over. Plain-English descriptions that get applied at runtime against conversation history.
- Dashboard now includes an exclude-list table with an **Exclude** button on every Inbox row, alongside the existing **Add to Allowlist** button.
- Synthetic smoke test (Step 7) now verifies both SAFE and ON behavior plus that button-click events route correctly.

**Rewritten.**

- Step 9 reframed from "Expand the Allowlist" to "Go Live," with SAFE and ON presented as two valid steady-state modes. SAFE is allowlist-driven for cautious or high-stakes launches. ON is exclusion-driven for higher-volume or automation-heavy setups. Neither is positioned as the "real" goal over the other. *(Corrected in v1.3: SAFE is not a steady state.)*
- Autopilot descriptions in Step 5 updated to reflect the parallel modes. *(Corrected in v1.3.)*

### `integration-notes.md` updates

**Added.**

- `new_button_clicked` now documented in the subscription list with its payload shape.
- New "Event type payload shapes" section covering `new_message`, `new_story_reply`, `new_comment`, and `new_button_clicked`. Button taps and quick-reply taps have inverted `message_id` / `message_text` shapes — button taps have the label in `message_text` with empty `message_id`; quick-reply taps have populated `message_id` with empty `message_text`.
- New "Standard workflow: button-click events" section. Button clicks are automation-flow signals, not agent response triggers.
- Debounce section notes that button clicks are not part of the debounce window for response purposes.
- Standard DM workflow now includes an explicit step to run the playbook's decision-to-respond logic before generating a response.

**Note on identifiers.** Button events have empty `message_id`, so any dedupe or flag tracking keyed on `message_id` will break. Use `event_id` from the top-level webhook payload, or a composite key for button events specifically.

### `playbook.md` updates

**Added.**

- New "Deciding Whether to Respond" section with a 7-step ordered decision: event type, self-echo, autopilot mode, exclusion rules, automation handoff, fetch history, generate. Every skipped event must log a reason to the activity feed.
- New "Automation Handoff" section defining the runtime logic for coordinating with DM automations. Operator describes the handoff in plain English; agent compares conversation history against that description on every `new_message` event. Button and quick-reply events are always automation-flow signals, never handoff moments.

## April 20, 2026 (late evening)

### `integration-notes.md` correction

Fixed an incorrect framing of Meta's per-comment limits.

**Changed.** The "One public comment reply per comment" section was rewritten to "One DM per comment."

Prior text claimed Meta limits businesses to one public reply per comment. That's wrong. Meta limits one DM per comment through the comment-to-DM tools (error subcode 2534023 fires on the second DM attempt for the same comment ID). Public replies via `send_public_comment_reply` have no per-comment limit. Agents that don't track public replies will keep replying to the same comment every time a webhook event comes through for it.

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
