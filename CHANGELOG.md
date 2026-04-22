# Changelog

All notable changes to the files in this repo are logged here, newest first.

## April 21, 2026 (evening)

### `integration-notes.md` updates

Saved-response surface added. Ellipsend shipped a saved-response library with two new send tools and one new resource. Coverage now in integration-notes.

**Added.**

- New "Saved responses" section. Covers the resource (`ellipsend://saved_responses`) and the two send tools (`send_saved_response` and `reply_to_comment_with_saved_response`). Documents the resource payload shape (`{id, shortcut, message_type}`), the portability rule (reference saved responses by `shortcut` in your configuration; `id` is tenant-scoped), observed `message_type` values (`text`, `button`, `audio`), the blind-pick constraint (the resource exposes no body text, so the agent picks by shortcut and trusts operator-approved content), the MIME quirk (JSON payload returned as `text/plain` inside `contents[0].text`), and Meta window behavior.
- Standard workflow: DMs step 6 now lists `send_saved_response` as an alternative to `send_dm` and `send_dm_with_button` when the operator has a saved response configured.
- Standard workflow: button-click events step 6 now lists `send_saved_response` alongside the freeform send tools.
- Standard workflow: comment-triggered DMs step 6 now lists `reply_to_comment_with_saved_response` as an alternative to the freeform comment-to-DM tools.
- Meta 24-hour messaging window section now names `send_saved_response` as subject to the same window as `send_dm`.
- 7-day CTD window section now names `reply_to_comment_with_saved_response` as subject to the 7-day comment-to-DM window.
- One DM per comment section now names `reply_to_comment_with_saved_response` as subject to Meta's one-DM-per-comment limit (subcode 2534023 on a second attempt).

**Note on `kickoff-prompt.md`.** Unchanged. The MCP is self-describing, so the agent discovers these tools automatically on connection. Saved responses don't require upfront caching at setup (unlike `ellipsend://relationships`, which needs its ID mapping cached before `set_contact_relationship` can be called). The agent reads `ellipsend://saved_responses` when composing a reply and selects by shortcut as needed. No behavioral change to the setup flow.

## April 21, 2026 (late afternoon)

### `kickoff-prompt.md` v1.4

Button-click semantics corrected. The v1.2 and v1.3 framing said `new_button_clicked` events should always be skipped and treated as automation-flow signals, never handoff moments. That's wrong. A button tap can be the handoff moment if the operator described it that way (for example, "when they tap the 'Tell me more' button, that's you taking over").

**Corrected rule.** Both `new_message` and `new_button_clicked` events are handoff candidates. The operator's handoff description decides which specific event in a conversation is the handoff moment. Default when no handoff rule is configured: respond to both event types.

**Rewritten.**

- Step 6 Automation handoff subsection. Removed the "treat button clicks as automation-flow signals, never handoff moments" blanket rule. Added three example handoff descriptions (typed-reply, button-click, quick-reply). Clarified that event type doesn't determine behavior on its own; operator's handoff description does. Default-when-no-rule clarified.
- Step 6 Debounce subsection. `new_button_clicked` events are processed immediately without debounce batching.
- Step 7 smoke test. Button-click verification now tests the handoff rule path (or the default response path if no rule is configured), not a blanket skip.
- Step 8 scenario 4 (Automation handoff). Updated to test that the agent evaluates each event against the handoff rule, not that it blindly skips all button clicks.
- v1.3 callout at top replaced with v1.4 callout.

### `playbook.md` updates

- "Deciding Whether to Respond" step 1 rewritten. Old step 1 said `new_button_clicked` events always skip at this stage. New step 1 says log the event and let the handoff rule (step 5) decide.
- "Automation Handoff" section rewritten. Removed the "always/never" framing about button taps. Agent now evaluates the handoff rule on every inbound event regardless of type. Added three example handoff descriptions. Default-when-no-rule behavior clarified as "respond to both event types." Uncertainty default flipped: previously "when in doubt, skip"; now "when in doubt, respond" (operator's intent is an active agent).

### `integration-notes.md` updates

- `new_button_clicked` event description rewritten. No longer says button clicks are "not a handoff moment" by default. Now says behavior depends on the operator's handoff rule, with respond-as-organic-message as the default when no rule is configured.
- "Standard workflow: button-click events" rewritten. Button clicks now run through the same decision-to-respond logic as `new_message` events, with the automation handoff step being where the operator's rule decides.
- Debounce section updated. Button clicks processed immediately without batching; automation handoff logic handles coordination with in-flight automations.

**Parked for v1.5 or Module 6 extension.** A "wait-a-beat" debounce where the agent briefly waits after a button click to see if an automation's next step fires. More forgiving to operators who forgot to configure handoff. Adds stateful complexity to the runtime and is deferred until there's operator demand for it.

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
- New "Standard workflow: button-click events" section. Button clicks are automation-flow signals, not agent response triggers. *(Corrected in v1.4: button clicks are handoff candidates; handoff rule decides.)*
- Debounce section notes that button clicks are not part of the debounce window for response purposes.
- Standard DM workflow now includes an explicit step to run the playbook's decision-to-respond logic before generating a response.

**Note on identifiers.** Button events have empty `message_id`, so any dedupe or flag tracking keyed on `message_id` will break. Use `event_id` from the top-level webhook payload, or a composite key for button events specifically.

### `playbook.md` updates

**Added.**

- New "Deciding Whether to Respond" section with a 7-step ordered decision: event type, self-echo, autopilot mode, exclusion rules, automation handoff, fetch history, generate. Every skipped event must log a reason to the activity feed. *(Step 1 corrected in v1.4.)*
- New "Automation Handoff" section defining the runtime logic for coordinating with DM automations. Operator describes the handoff in plain English; agent compares conversation history against that description on every `new_message` event. Button and quick-reply events are always automation-flow signals, never handoff moments. *(Corrected in v1.4: handoff rule decides, not event type.)*

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
