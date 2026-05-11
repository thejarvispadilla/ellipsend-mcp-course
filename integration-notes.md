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

There's a fifth, optional event: `message_sent_outbound`. Off by default. Has a real feedback-loop risk if mishandled. See the dedicated section below before subscribing.

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

These events represent user interaction with a DM automation flow. Whether the agent should respond depends on the operator's configured automation handoff rule, not on the event type itself. A button tap can be the handoff moment if the operator described it that way ("when they tap the 'Tell me more' button, that's you"). It can also be mid-flow, where the automation is about to fire its next step and the agent should wait. See the playbook's "Automation Handoff" section for the decision logic.

If no handoff rule is configured, the default is to treat button clicks as organic messages and respond. This may step on a running automation if one exists; configuring the handoff rule is how the operator prevents that.

Don't key your message dedupe or flag tracking on `message_id` for button events. Empty string collisions across multiple button taps will break that logic. Use `event_id` from the top-level webhook payload instead, or a composite of `meta_contact_id + occurred_at` for button events specifically.

## Standard workflow: DMs

When a `new_message` or `new_story_reply` webhook arrives:

1. Receive the webhook payload (message content, `meta_contact_id`, contact metadata).
2. Read the contact via `ellipsend://contacts/{contact_id}` to check current relationship and labels.
3. Read conversation history via `ellipsend://contacts/{contact_id}/messages/{page}/{page_size}` for context.
4. Check the playbook's decision-to-respond logic: exclusion rules, allowlist/exclude-list, automation handoff. If any of them say skip, log and stop.
5. Decide how to respond based on the playbook and your business configuration.
6. Reply with `send_dm` or `send_dm_with_button` for freeform content, or `send_saved_response` if the operator has a saved response configured for this situation. See the Saved responses section below.
7. Update the contact's relationship or label with `set_contact_relationship` or `set_contact_label` if appropriate.

## Standard workflow: button-click events

When a `new_button_clicked` webhook arrives:

1. Receive the webhook payload (`message_type: "postback"`, either a button label or a quick-reply with populated `message_id`).
2. Log the event so it shows in the activity feed and is visible in conversation history later.
3. Read the contact via `ellipsend://contacts/{contact_id}` and conversation history via `ellipsend://contacts/{contact_id}/messages/{page}/{page_size}`.
4. Run the playbook's decision-to-respond logic. At the automation handoff step: if the operator has a handoff rule configured, evaluate it against the current conversation state to decide whether this button click is the handoff moment. If yes, respond. If no (automation is still in flight, or the button click doesn't match the described handoff point), skip and log.
5. If the operator has no handoff rule configured, the default is to treat the button click as an organic message and respond as normal. This may step on a running automation if one exists. The operator's job is to configure a handoff rule to prevent that.
6. If responding, generate the response using the business config and send via `send_dm`, `send_dm_with_button`, or `send_saved_response`.

## Standard workflow: comment-triggered DMs

When a `new_comment` webhook arrives:

1. Receive the webhook payload (comment text, `comment_id`, `post_id`, commenter metadata).
2. Skip if you've already handled this `comment_id`. Track responded comment IDs.
3. Read the post via `ellipsend://posts/{post_id}` to understand its topic and caption.
4. Read the commenter's contact and any prior conversation history.
5. Decide whether to respond based on the playbook rules.
6. If responding, reply with `reply_to_comment_with_dm` or `reply_to_comment_with_dm_with_button` for freeform content, or `reply_to_comment_with_saved_response` if the operator has a saved response configured. These tools link the DM to the specific comment, which is required by Meta. Don't use regular `send_dm` for comment-triggered responses.
7. Optionally call `send_public_comment_reply` to post a public reply to the comment as well.

## Saved responses

Ellipsend supports a saved-response library. The operator creates and approves saved responses inside the Ellipsend dashboard. Your agent reads the library and sends a saved response by id, either as a DM (`send_saved_response`) or as a comment-triggered DM (`reply_to_comment_with_saved_response`).

### Reading the library

Read `ellipsend://saved_responses` to list what's available. Each entry has exactly three fields:

- `id` — integer. Tenant-scoped. The same saved response will have a different id in a different operator's account. Not portable across accounts.
- `shortcut` — string. A short human-friendly handle set by the operator (for example, `"discord"`, `"beta"`, `"vmrespond"`). If your configuration refers to a specific saved response, reference it by shortcut, not by id.
- `message_type` — string. Describes what the saved response sends. Observed values are `text` (plain DM body), `button` (body plus a CTA link, parallel to `send_dm_with_button`), and `audio` (a pre-recorded voice note). More types may exist; inspect the operator's library to see.

The resource returns `mimeType: "text/plain"` with the JSON payload inside `contents[0].text`. Parse the text field explicitly; don't rely on the MIME type.

### Sending a saved response

The send tools take only a contact (or comment) identifier and a `saved_response_id`. They don't accept body overrides or variable substitutions from the agent. If the saved response is a template with placeholders, Ellipsend fills them server-side from stored defaults; your agent doesn't pass runtime variables.

One universal dispatcher on each path. `send_saved_response` handles all `message_type` values (text, button, audio) through the same call. The server dispatches based on the stored type. You don't need to branch on type at the agent level. Same story for `reply_to_comment_with_saved_response` on the comment-to-DM path.

### Blind-pick

The resource exposes only `{id, shortcut, message_type}`. The body text, the link URL, the button label — none of these are visible to the agent before sending. Your agent picks a saved response by its `shortcut` and trusts that the operator has approved the content.

Practical consequence: configuration that tells the agent "send the Discord invite" should reference the saved response by shortcut (for example, `shortcut == "discord"`). The agent sends that saved response and trusts the operator's approval of the body. If you want to verify a send landed, check the outbound in conversation history via `fetch_messages`.

### 24-hour window and Meta limits

`send_saved_response` is subject to the same Meta 24-hour messaging window as `send_dm` and `send_dm_with_button`. If the window is closed, the send fails at Ellipsend's server-side guardrail.

`reply_to_comment_with_saved_response` uses the 7-day comment-to-DM window, same as the other comment-to-DM tools, and is governed by Meta's one-DM-per-comment limit (subcode 2534023 on a second attempt for the same comment).

## `message_sent_outbound` (opt-in)

Ellipsend offers a fifth webhook event called `message_sent_outbound`. It fires every time a DM is sent FROM the business account, regardless of source — your agent, a teammate composing manually in Ellipsend's dashboard, any other path. The use case is keeping your agent's local message history in sync when humans send manual replies. Without this event, your agent doesn't see those messages and loses context.

The event is opt-in because subscribing without a specific safeguard pattern creates a feedback loop. If your handler treats `message_sent_outbound` as something to respond to, your agent replies, the reply fires another `message_sent_outbound`, your agent replies again, and you're in a spam loop until you cut it off. Spammed contacts, burned tokens, real reputational damage.

Off by default. Subscribe only after implementing the safeguards below and testing them. The kickoff prompt's Step 3 surfaces the opt-in question and walks the agent through the setup.

**Use at your own risk.** Ellipsend ships the event reliably; your handler is responsible for processing it correctly. The course documents the safeguard pattern. Your implementation owns the loop prevention.

### Payload shape

Same envelope as `new_message`. The notable fields:

- `direction: "outbound"` — the reliable marker that this is a business-side send.
- `origin: "agent"` — this is a constant. Empirically observed across thousands of events: every outbound carries `origin: "agent"` regardless of who actually sent the message. Don't write logic that branches on `origin` to distinguish human-from-agent. The field doesn't carry that signal.
- `trigger_agent: false` — also a constant on this event. Belt-and-suspenders flag from Ellipsend saying "this event should not invoke your agent."
- `sender_id` — always your business Meta ID for outbounds.
- `message_id` — Meta's platform-issued message ID. Use this as your dedup key.
- `message_text` — the message body.

### The minimum safeguard: route by event_type

Branch at the top of your webhook handler. `message_sent_outbound` goes to a code path that does NOT invoke your agent. Pseudocode:

```
function handleWebhook(event) {
  if (event.event_type === "message_sent_outbound") {
    return handleOutboundEcho(event);   // sync-only, never calls agent
  }
  return handleInboundEvent(event);     // your normal AI-trigger path
}
```

This is the floor. Without this, the loop is mechanically possible.

The `handleOutboundEcho` function writes the message to your local DB and exits. It does not call your agent, look up the contact, run the playbook, or do anything that could lead to a response generation.

### Storage approach

When `message_sent_outbound` lands, write the message to your local messages table as an outbound row. Use `message_id` as a unique key with INSERT OR IGNORE semantics so re-deliveries from Ellipsend don't duplicate. Mark `direction = "outbound"` on the row.

For attribution, if you want to distinguish your agent's own sends from manual or other sends for UI purposes, tag a `sender_source` field at YOUR own send path. When your agent calls `send_dm` (or any send tool), pre-insert the outbound row in your local DB with `sender_source = "agent"` immediately. When the echo arrives later, your INSERT OR IGNORE on `message_id` will dedup. The agent's pre-insert won't have a `message_id` yet (Meta hasn't issued it), so you can either: (a) leave both rows and rely on UI logic to display one per Meta-issued message, or (b) write content+time dedup that stamps the echo's `message_id` onto your pre-inserted row when bodies match within a short window. Option (b) is cleaner but more complex.

The simpler pattern for students who don't need eager local state: don't pre-insert on send. Wait for the echo to arrive, then write the row. Your local DB has a brief lag (a few seconds typically) between when you send and when your DB reflects it.

The webhook's `origin` field is always `"agent"`. It cannot be used to distinguish source. If you need that distinction, you must tag it at your send paths, not derive it from the payload.

### Rate-limit circuit (catastrophic backstop)

Even with correct routing, a bug somewhere could create a loop. Add a rate limit on your agent's send path: cap at N sends in M minutes. If exceeded, flip autopilot to OFF and alert yourself (Discord webhook, email, whatever you check).

A reasonable starting cap for a low-volume operator is 50 sends in 2 minutes. Higher-volume operators (managing comment-to-DM flows for viral posts) can raise the cap, but the protective floor stays the same. The cap should reflect "noticeably above normal traffic, but well below loop territory."

This is your last line of defense. If everything else fails, the rate limit catches it.

### Test before subscribing live

Don't add `message_sent_outbound` to your live webhook subscription until you've verified the routing with a synthetic event. POST a fake payload to your local handler:

```bash
curl -X POST http://localhost:{PORT}/api/webhook \
  -H "Content-Type: application/json" \
  -H "x-webhook-event: message_sent_outbound" \
  -d '{
    "event_type": "message_sent_outbound",
    "data": {
      "direction": "outbound",
      "origin": "agent",
      "trigger_agent": false,
      "sender_id": "YOUR_BUSINESS_META_ID",
      "meta_contact_id": "TEST_CONTACT_ID",
      "message_id": "synthetic_test_001",
      "message_text": "synthetic loop-prevention test"
    }
  }'
```

Verify three things:

1. The event routes to your echo handler, not your AI-trigger handler.
2. Your agent is never invoked. No tokens spent, no MCP calls fired.
3. The message lands in your local DB with the correct outbound marker.

Only after this passes should you add `message_sent_outbound` to your live subscription.

## Debounce

When multiple `new_message` events arrive from a contact in quick succession, don't respond immediately. Wait for a 10-second silence window before processing. Then treat all new messages as one context and send one coherent reply. Without debounce, you'll reply mid-thought and miss context from the follow-up messages.

Debounce applies to inbound `new_message` events only. `new_button_clicked` events are processed immediately, without batching. They typically arrive as discrete single interactions (one button tap at a time), and the automation handoff logic in the playbook handles coordination with any in-flight automation flows.

Do not add a per-contact cooldown after sending. Older versions of this file included a 60-second cooldown. It caused real substantive follow-ups to get dropped. Debounce on incoming messages alone handles the rapid-fire case.

## Meta platform quirks

Platform behaviors that look like bugs but trace back to Meta's API rules. Name them, handle them, move on.

### 24-hour messaging window

Meta only allows a business to DM a contact via the standard Messaging API if the contact has messaged the business in the last 24 hours. Outside that window, `send_dm`, `send_dm_with_button`, and `send_saved_response` will fail.

Ellipsend's server-side guardrails block the call before it leaves the platform, so your Meta account stays protected from policy strikes. You'll see a clean error your agent can react to (flag for the operator, queue outreach via comment-to-DM when they next comment, etc.).

### 7-day CTD window

Comment-to-DM tools (`reply_to_comment_with_dm`, `reply_to_comment_with_dm_with_button`, `reply_to_comment_with_saved_response`) have a separate 7-day window. You can DM a commenter up to 7 days after they commented, even if the standard 24-hour messaging window with them is closed. This is what makes comment tools powerful for reactivation.

### Self-echo

When you post a public comment reply, Instagram delivers that reply back to the business as a `new_comment` webhook (because the business is "the one who posted the comment"). If you don't filter these out, your agent tries to process its own replies as if they came from a contact.

Filter by dropping any webhook where `sender_id` or `meta_contact_id` matches the business Meta ID. Store the business Meta ID in `.env` as `BUSINESS_META_ID` and check it at the handler.

### 400-that-delivered

Meta occasionally returns a `400 Bad Request` on a send while actually delivering the message. Don't retry on 400 blindly. Check whether the message landed (in the webhook stream or message history) before retrying. Blind retries produce duplicate sends.

### One DM per comment

Meta limits businesses to one DM per comment through the comment-to-DM tools. If your agent tries to DM the same commenter twice for the same `comment_id` via `reply_to_comment_with_dm`, `reply_to_comment_with_dm_with_button`, or `reply_to_comment_with_saved_response`, Meta returns error subcode 2534023 on the second attempt. Track which comment IDs you've already DM'd and skip them.

Public replies are different. `send_public_comment_reply` has no per-comment limit. Without tracking, an agent will keep publicly replying to the same comment every time a new webhook event comes through for it. Track public replies the same way you track DMs.

## Tool schemas

Don't reproduce the MCP schema in this file. It will go stale.

Connect to `https://mcp.ellipsend.com/mcp/` and list tools to see everything available. New tools and resources are added over time. If the schema defines a tool not mentioned in this file, use it as its schema describes. The schema covers:

- Required and optional arguments
- Argument types and formats
- Any per-tool constraints (like the 24-hour window on `send_dm`)
