# Ellipsend Integration Notes

*Your AI tool consults this file when it needs webhook setup, workflow patterns, or Meta platform quirks. Not required per-session. Version history in `CHANGELOG.md`.*

---

## Why this file exists

The Ellipsend MCP is self-describing. When your AI tool connects to `https://mcp.ellipsend.com/mcp/`, it already sees every tool's name, description, input schema, and required fields. That's the authoritative reference for what tools exist and what arguments each one takes.

This file covers workflow context and platform quirks the schema can't communicate on its own. For tool arguments, types, and constraints, use the MCP schema directly.

## Webhook subscription

The webhook is a separate system from the MCP server. You subscribe by POSTing to `https://webhook.ellipsend.com/subscriptions` with your API key in the `x-api-key` header.

Subscribe to all four event types:

- `new_message` — organic typed DMs.
- `new_story_reply` — replies to your Instagram stories.
- `new_comment` — comments on your posts.
- `new_button_clicked` — quick-reply taps and message-button clicks. Without this, your agent is deaf to button interactions.

Full subscription curl example lives in the kickoff prompt, Step 3.

### Webhook security

- **Secret verification.** When your endpoint receives a POST, verify the secret in the payload matches the one you registered. Reject any request without a valid secret. This blocks any traffic that isn't Ellipsend.
- **Expose only the webhook path publicly.** Your dashboard and all other routes should stay private. Use path-based ingress (Cloudflare Tunnel, VPS reverse proxy, or your hosting platform's routing) to restrict public access to just `/api/webhook` or whichever path you configure. Everything else returns 404 from the public internet.

## Event type payload shapes

The payload shape differs across event types. Your webhook handler needs a branch for each.

### `new_message` and `new_story_reply`

Organic typed content. `message_type: "message"`. The `message_text` field contains what the user typed. `message_id` is populated.

These are the events where your agent usually decides whether to respond.

### `new_comment`

A comment on one of your posts. Payload includes `comment_id`, `post_id`, comment text, and commenter metadata. See the comment-triggered DM workflow below.

### `new_button_clicked`

Fires when a contact taps a button or quick-reply option inside a DM conversation. Delivered with `message_type: "postback"`.

Two variants with different payload shapes:

- **Button tap.** `message_id` is empty string. `message_text` contains the button label (for example, `"Testing Button"`).
- **Quick-reply tap.** `message_id` is populated. `message_text` is empty string.

Either way, these events signal that the user is interacting with an in-progress DM automation (a flow that ends in a button prompt, a sequence that uses quick replies to guide the user through a tree). For automation handoff logic, treat `new_button_clicked` events as "user is still in the automation, not a handoff moment." See the playbook's Automation handoff section for the full rule.

Don't key your message dedupe or flag tracking on `message_id` for button events. Empty string collisions across multiple button taps will break that logic. Use `event_id` from the top-level webhook payload instead, or a composite of `meta_contact_id + occurred_at` for button events specifically.

## Standard workflow: DMs

When a `new_message` or `new_story_reply` webhook arrives:

1. Receive the webhook payload (message content, `meta_contact_id`, contact metadata).
2. Read the contact via `ellipsend://contacts/{contact_id}` to check current relationship and labels.
3. Read conversation history via `ellipsend://contacts/{contact_id}/messages/{page}/{page_size}` for context.
4. Check the playbook's decision-to-respond logic: exclusion rules, allowlist/exclude-list, automation handoff. If any of them say skip, log and stop.
5. Decide how to respond based on the playbook and your business configuration.
6. Reply with `send_dm` or `send_dm_with_button`.
7. Update the contact's relationship or label with `set_contact_relationship` or `set_contact_label` if appropriate.

## Standard workflow: button-click events

When a `new_button_clicked` webhook arrives:

1. Receive the webhook payload (`message_type: "postback"`, either a button label or a quick-reply with populated `message_id`).
2. Log the event so it shows in the activity feed and is visible in conversation history later.
3. Check whether the operator configured automation handoff. If yes, treat this event as "user is still in the automation" and do not respond. The handoff moment is the subsequent organic `new_message` event, not this button tap.
4. If the operator did not configure automation handoff (no automations in their setup), the event can still be recorded to history but should not trigger an agent response on its own. Button clicks are flow interactions, not DM questions.

## Standard workflow: comment-triggered DMs

When a `new_comment` webhook arrives:

1. Receive the webhook payload (comment text, `comment_id`, `post_id`, commenter metadata).
2. Skip if you've already handled this `comment_id`. Track responded comment IDs.
3. Read the post via `ellipsend://posts/{post_id}` to understand what the post is about.
4. Read the commenter's contact and any prior conversation history.
5. Decide whether to respond based on the playbook rules.
6. If responding, reply with `reply_to_comment_with_dm` or `reply_to_comment_with_dm_with_button`. These tools link the DM to the specific comment, which is required by Meta. Don't use regular `send_dm` for comment-triggered responses.
7. Optionally call `send_public_comment_reply` to post a public reply to the comment as well.

## Debounce

When multiple messages arrive in quick succession, don't respond immediately. Wait for a 10-second silence window before processing. Then treat all new messages as one context and send one coherent reply. Without debounce, you'll reply mid-thought and miss context from the follow-up messages.

Debounce applies to `new_message` events only. `new_button_clicked` events are automation-flow signals, not organic messages, so they don't accumulate in the debounce window for response purposes.

Do not add a per-contact cooldown after sending. Older versions of this file included a 60-second cooldown. It caused real substantive follow-ups to get dropped. Debounce on incoming messages alone handles the rapid-fire case.

## Meta platform quirks

Platform behaviors that look like bugs but trace back to Meta's API rules. Name them, handle them, move on.

### 24-hour messaging window

Meta only allows a business to DM a contact via the standard Messaging API if the contact has messaged the business in the last 24 hours. Outside that window, `send_dm` and `send_dm_with_button` will fail.

Ellipsend's server-side guardrails block the call before it leaves the platform, so your Meta account stays protected from policy strikes. You'll see a clean error your agent can react to (flag for the operator, queue outreach via comment-to-DM when they next comment, etc.).

### 7-day CTD window

Comment-to-DM tools (`reply_to_comment_with_dm`, `reply_to_comment_with_dm_with_button`) have a separate 7-day window. You can DM a commenter up to 7 days after they commented, even if the standard 24-hour messaging window with them is closed. This is what makes comment tools powerful for reactivation.

### Self-echo

When you post a public comment reply, Instagram delivers that reply back to the business as a `new_comment` webhook (because the business is "the one who posted the comment"). If you don't filter these out, your agent tries to process its own replies as if they came from a contact.

Filter by dropping any webhook where `sender_id` or `meta_contact_id` matches the business Meta ID. Store the business Meta ID in `.env` as `BUSINESS_META_ID` and check it at the handler.

### 400-that-delivered

Meta occasionally returns a `400 Bad Request` on a send while actually delivering the message. Don't retry on 400 blindly. Check whether the message landed (in the webhook stream or message history) before retrying. Blind retries produce duplicate sends.

### One DM per comment

Meta limits businesses to one DM per comment through the comment-to-DM tools. If your agent tries to DM the same commenter twice for the same `comment_id` via `reply_to_comment_with_dm` or `reply_to_comment_with_dm_with_button`, Meta returns error subcode 2534023 on the second attempt. Track which comment IDs you've already DM'd and skip them.

Public replies are different. `send_public_comment_reply` has no per-comment limit. Without tracking, an agent will keep publicly replying to the same comment every time a new webhook event comes through for it. Track public replies the same way you track DMs.

## Tool schemas

Don't reproduce the MCP schema in this file. It will go stale.

Connect to `https://mcp.ellipsend.com/mcp/` and list tools to see everything available. New tools and resources are added over time. If the schema defines a tool not mentioned in this file, use it as its schema describes. The schema covers:

- Required and optional arguments
- Argument types and formats
- Any per-tool constraints (like the 24-hour window on `send_dm`)
