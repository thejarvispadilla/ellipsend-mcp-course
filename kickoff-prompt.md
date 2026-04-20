# Ellipsend AI Agent Setup. Kickoff Prompt v1.2

*One-shot setup for your Ellipsend-connected DM agent. Paste this into Claude Code, OpenClaw, Codex, or any AI tool that supports MCP and agentic execution. Run once. After setup, your tool loads `playbook.md` at the start of every session.*

*Version history lives in `CHANGELOG.md`.*

> 🆕 **New in v1.2.** Added `new_button_clicked` to webhook subscription (Ellipsend delivers quick-reply taps and message-button clicks as this event type; agents without it are deaf to button interactions). Added automation handoff question to Step 1 and exclusion rules framing to Step 6. Step 9 now frames SAFE and ON as parallel valid modes with a shared exclusion list. Full diff in [`CHANGELOG.md`](CHANGELOG.md).

---

You are becoming a DM agent connected to Ellipsend, a social media platform for Instagram. This prompt walks you through setup end to end. By the end, you'll be configured, have a local dashboard, be deployed reliably, and be ready to handle real DMs under the mode the operator chooses.

**After setup, load `playbook.md` at the start of every session.** It defines behavior, escalation rules, automation handoff logic, and security guardrails.

Store the playbook in your persistent config. OpenClaw uses a skill file. Claude Code references it from `CLAUDE.md`. Codex stores it alongside `AGENTS.md`.

`integration-notes.md` is consulted ad-hoc when you need webhook setup, workflow patterns, or Meta platform quirks. Not required per-session.

---

## Step 1: Learn About My Business

Before asking me anything, check what you already know about my business from prior conversations, memory, files, or any context you have.

Present what you found and ask me to confirm or correct it. If you have no prior context, ask me directly.

Push back on vague answers. Your performance depends on specificity.

Gather:

- **Who you are in DMs.** When someone asks "who am I talking to?", pick one approach and commit. You can respond as me in first person, as if you ARE me. You can respond as a named team member, like "Hey, I'm Sarah from [business]'s team." You can respond as an AI assistant, like "I'm [name]'s AI assistant."
- **What I sell** and at what price points.
- **My voice.** How I sound in DMs.
- **Ideal customer.** Who's a fit, who isn't.
- **My CTA.** Booking link, application, website. Whatever next step I want leads pushed toward.
- **Escalation triggers.** When to stop responding and flag for me to review.
- **Off-limits topics.** Anything you should never respond to.
- **Operating hours.** 24/7 or bounded windows.
- **Labels.** Ask if I already have labels set up. If I do, get the exact list. If not, propose a starter set and get my approval. Never create labels on your own.
- **Active DM automations.** Ask whether I currently run DM automations in Ellipsend (comment-to-DM flows, welcome sequences, keyword triggers). If yes, ask me to describe each one: what triggers it, what it sends, and where in the flow I want you to take over. This is context for the automation handoff rule in the playbook. If I'm not running automations, skip. We'll revisit the handoff decision when it's time to go live.

## Step 2: Connect to Ellipsend

Ask me for my Ellipsend MCP server URL and API key. I'll grab these from **Settings → Integrations → MCP** inside Ellipsend.

1. **Connect to the MCP server.** List tools to confirm the connection works. The MCP is self-describing, so you'll see every tool's name, description, input schema, and required fields. That's your authoritative reference for what's possible.
2. **Read the `ellipsend://relationships` resource.** Cache the ID-to-name mapping. You'll use these when updating contact relationships.
3. **Pull a test conversation.** Ask me for a contact ID and fetch messages via MCP. Confirm you can read before proceeding.

## Step 3: Set Up the Public Webhook Endpoint

Ellipsend delivers four event types via webhook:

- `new_message` — organic typed DMs and story replies.
- `new_story_reply` — replies to your Instagram stories.
- `new_comment` — comments on your posts.
- `new_button_clicked` — when a contact taps a quick-reply or message button. Delivered with `message_type: "postback"`. Without this subscription, your agent is deaf to button interactions, which matters heavily if you run DM automations.

Your agent needs a public URL to receive them.

Full infrastructure instructions (Cloudflare Tunnel, Railway, Fly.io, VPS) live in Module 1 of the Ellipsend MCP Course. I should have completed that setup already. If I haven't, pause here and point me back to the course.

Once I have a public URL:

1. **Generate a webhook secret.**

   ```bash
   openssl rand -hex 32
   ```

2. **Subscribe to the webhooks.**

   ```bash
   curl --location 'https://webhook.ellipsend.com/subscriptions' \
   --header 'Content-Type: application/json' \
   --header 'x-api-key: {ELLIPSEND_API_KEY}' \
   --data '{
     "event_types": ["new_message", "new_story_reply", "new_comment", "new_button_clicked"],
     "target_url": "https://{PUBLIC_URL}/api/webhook",
     "secret": "{WEBHOOK_SECRET}"
   }'
   ```

3. **Verify the secret on every incoming POST.** If the secret in the payload doesn't match the one you registered, reject the request with 401. This blocks anything that isn't Ellipsend.

4. **Filter self-echoes.** When you post a public comment reply, Instagram delivers that reply back through the webhook as a `new_comment` event. Drop any webhook where `sender_id` or `meta_contact_id` matches my Meta business ID. Store my business Meta ID in `.env` as `BUSINESS_META_ID` and filter at the handler.

5. **Route all four event types.** Your webhook handler needs a branch for each of `new_message`, `new_story_reply`, `new_comment`, and `new_button_clicked`. See `integration-notes.md` for the payload shape of `new_button_clicked` and how button taps differ from organic messages. Unhandled event types should be logged but not silently dropped.

6. **Test the subscription.** Ask me to send a test DM to my business account. Confirm the event lands in your server logs. Diagnose if it doesn't, before moving on.

## Step 4: Keep Your Agent Alive

A local process that dies when the terminal closes won't cut it. Your agent has to be running whenever messages come in.

### macOS (launchd)

Create `~/Library/LaunchAgents/com.{name}.agent.plist` with `RunAtLoad`, `KeepAlive`, and the path to your project. Set `NODE_EXTRA_CA_CERTS` in `EnvironmentVariables` pointing to a PEM bundle of the system keychain. Node processes launched via launchd don't trust the macOS keychain by default. Outbound HTTPS calls to `mcp.ellipsend.com` will fail without this.

Generate the bundle:

```bash
security find-certificate -a -p /System/Library/Keychains/SystemRootCertificates.keychain > ca-bundle.pem
security find-certificate -a -p /Library/Keychains/System.keychain >> ca-bundle.pem
```

Restart the service after code changes:

```bash
launchctl kickstart -k gui/$(id -u)/com.{name}.agent
```

If I run multiple agents on the same machine, never broad-`pkill` by process name. Always kill by PID or port. Running `pkill -f "next-server"` will take down every Next.js agent on the machine.

### Linux (systemd)

Create a systemd service unit with `Restart=always`. `NODE_EXTRA_CA_CERTS` isn't typically needed on Linux.

### Railway or Fly.io

Both handle process management. Just deploy.

### VPS

Use `pm2` or systemd. Configure auto-restart.

### Env var override (macOS + Claude Code)

Claude Desktop exports `ANTHROPIC_API_KEY=` (empty) to child processes. Node's native dotenv loader respects existing env vars, so the inherited empty string wins over my `.env` file. Fix with `dotenv.config({ override: true })` at startup.

For Next.js 16, put this in `instrumentation.ts`:

```typescript
import { config } from "dotenv";
export async function register() {
  config({ override: true });
}
```

Without this, the Anthropic SDK throws `Could not resolve authentication method`.

## Step 5: Build the Local Dashboard

Build a web dashboard I access in my browser. **Not** on the public internet. The public URL is only for the webhook path. Use Tailscale (my private Tailnet IP) or LAN for dashboard access.

Recommended framework is Next.js. It handles the webhook handler (backend), dashboard (frontend), and background processes in one project. Unless I specify otherwise, default to Next.js.

The dashboard needs these surfaces.

### Autopilot toggle

Three-state segmented control in the sidebar.

- **OFF.** You ingest messages and track them on the dashboard. You take no action. No replies, no labels, no relationship changes. Used during setup and training.
- **SAFE.** You ingest and track everyone. The AI only fires for contacts on the allowlist, minus anyone on the exclude list. Good for cautious launches and high-stakes contexts.
- **ON.** The AI fires for every contact, minus anyone on the exclude list. Exclusion rules determine scope instead of per-contact allowlisting. Good for higher-volume contexts where the operator wants the agent handling new leads broadly.

Both SAFE and ON are valid steady states. The right one depends on how the operator wants to manage scope. State persists to disk. Expose as `/api/autopilot`.

### Allowlist with Add button

SQLite table:

```sql
CREATE TABLE allowlist (
  meta_contact_id TEXT PRIMARY KEY,
  added_at TEXT DEFAULT (datetime('now')),
  note TEXT DEFAULT ''
);
```

Used only when autopilot is SAFE. Every contact row in the Inbox view gets an **Add to Allowlist** button.

Expose as `/api/autopilot/allowlist`.

### Exclude list

SQLite table:

```sql
CREATE TABLE exclude_list (
  meta_contact_id TEXT PRIMARY KEY,
  added_at TEXT DEFAULT (datetime('now')),
  reason TEXT DEFAULT ''
);
```

Applies in both SAFE and ON mode. Every contact row gets an **Exclude** button. Contacts on the exclude list are ingested and logged but never responded to. The operator uses this for VIPs, existing customers, partners, or anyone they've decided to handle personally.

Expose as `/api/autopilot/exclude`.

### Inbox view

Contacts populate as they message. Each row shows username, last message preview, time, current relationship, labels, and both **Add to Allowlist** and **Exclude** buttons. Clicking a contact loads the full conversation via `fetch_messages`.

### Activity feed

Real-time log of every action. "Received DM from @user," "SAFE mode blocked reply to @user (not on allowlist)," "ON mode blocked reply to @user (on exclude list)," "Replied to @user," "Flagged @user for review," "Added @user to exclude list."

### Rating loop

Every AI response gets logged with 👍 and 👎 buttons next to it. I rate each response as I review. Show a 14-day quality trend chart. Without ratings, I have no iteration signal.

### Cost cap

Token usage counter. Daily cost counter. A **daily hard cap** that stops the agent from sending further messages once hit. Default $5/day. Configurable. At low volume this never fires. At high volume or during a runaway loop, it prevents a surprise $2,000 bill.

An **hourly alert threshold** (warning, not a cap). Default $1/hour. Surface in the activity feed as a notification when hit.

### Token usage tracking

Per-call input and output tokens logged. Today's cost and cumulative cost visible on the dashboard.

### Persistence

Every piece of state (contacts, messages, activity, allowlist, exclude list, ratings, flags, autopilot mode, token usage) must survive restart. SQLite is sufficient.

## Step 6: Configure Behavior

Refer to `playbook.md` for the base rules. This step layers on business-specific configuration.

### DM handling

- **New lead.** First-time DM. How to open, qualify, push CTA, what label to apply.
- **Mid-conversation.** Ongoing qualification. When to advance the relationship.
- **Returning contact.** Previous conversation, back again. Call `fetch_messages` before responding.
- **Off-topic.** Unrelated questions.
- **Hostile or upset.** De-escalate or flag.
- **CTA timing.** When to send the booking link or button message.
- **Escalation scope.** Only evaluate the current message for escalation triggers. Older messages that were handled must not re-fire. Enforced by the flag lifecycle below.
- **Labeling.** Only use approved labels. If something doesn't fit, flag and ask me.
- **Relationship updates.** When to advance (booked a call, purchased, went cold).

### Exclusion rules

Beyond the exclude list of specific contact IDs, the operator can define runtime exclusion rules based on contact metadata. Ask me in plain English which contact types I want you to stay out of. Common ones:

- Relationships (like `customer`, `partner`, `paid member`) that mean "this is someone I handle personally."
- Labels (like `VIP`, `do-not-contact`, `in-person-only`) that mean "skip."
- A specific handoff condition tied to a DM automation (see automation handoff below).

Save these rules to persistent config. Before every response, check the contact's metadata against them. If any match, skip and log. The exclude list and exclusion rules work together: the list handles specific people, the rules handle categories.

### Automation handoff

If I told you in Step 1 that I run DM automations, ask me now: where in each automation do I want you to take over?

I'll describe it in plain English, the way I'd brief a human employee. Something like, "My automation sends my free guide and asks where they're based. Wait until they reply to the 'where are you based' question. That's when you take over."

You then apply this logic at runtime. On every `new_message` event, fetch conversation history via `fetch_messages`. Read the last outbound message I sent (or that the automation sent) and the user's latest reply. Decide whether this looks like the handoff moment I described. If yes, respond. If not (for example, the automation is still in the middle of its flow), skip and log.

Button taps and quick-reply selections come through as `new_button_clicked` events, not `new_message` events. See `playbook.md` and `integration-notes.md` for the rule. The short version: treat `new_button_clicked` events as automation-flow signals, not handoff moments. They confirm the user is still working through the automation.

### Flag lifecycle

Every escalation flags the current inbound message with `flag_status = flagged`. I resolve or ignore via dashboard buttons. When you build conversation history for the next AI call, annotate past messages with their flag status.

- `[previously flagged, RESOLVED by operator]`
- `[previously flagged, IGNORED by operator]`

Treat RESOLVED and IGNORED flags as closed. Only the current message determines whether to escalate again.

Without this, a single concerning message in history keeps firing escalations on every new message from that contact.

### Debounce, no cooldown

10-second silence window before processing a batch. If a contact sends three messages in five seconds, wait for silence, then process as one context.

Do not add a per-contact cooldown after sending. Older versions of this prompt included a 60-second cooldown. Substantive follow-ups within that window get dropped. Debounce alone handles the rapid-fire case. Cooldown causes more harm than help.

Show me your full configuration in plain English so I can approve or adjust.

## Step 7: Synthetic Smoke Test

Before real DMs, prove the pipeline end to end with a fake webhook event. Takes 30 seconds, saves hours of debugging.

Write a small script that POSTs a synthetic `new_message` webhook payload to `http://localhost:{PORT}/api/webhook` with the correct HMAC signature. Use a `sender_id` that is not my business Meta ID, so the self-echo filter doesn't drop it.

Verify:

1. The webhook endpoint accepts the payload.
2. The event lands in the activity feed.
3. The message appears in the inbox.
4. In SAFE mode with the contact on the allowlist, the AI fires.
5. In SAFE mode with the contact not on the allowlist, the AI is skipped and the skip is logged.
6. In ON mode with the contact NOT on the exclude list, the AI fires.
7. In ON mode with the contact on the exclude list, the AI is skipped and the skip is logged.
8. A synthetic `new_button_clicked` event is routed to your postback handler and treated as automation-flow (no response generated).

If any of these fail, diagnose before moving to real testing.

## Step 8: Test With a Real Contact

Ask me to send a DM to my business account from a test account (a friend's phone or a second account I own).

First, leave the test contact off the allowlist and confirm autopilot is SAFE. You ingest the message and log "SAFE mode, not on allowlist, no reply." I see the test message in the inbox. This proves SAFE mode is blocking correctly.

Next, I click **Add to Allowlist** on the test contact. I send another message. This time the AI fires and replies.

Then walk me through three scenarios with the test contact:

1. **New lead.** Qualification flow, CTA push, label set.
2. **Returning contact.** Context awareness via `fetch_messages`.
3. **Escalation.** Hostile or off-topic message triggers a flag, not a reply.

If I run DM automations, run a fourth scenario:

4. **Automation handoff.** I trigger my automation from the test account (comment the trigger word, click the button, whatever starts the flow). You receive the `new_button_clicked` events and log them as automation-flow. When my automation asks a question and the test account answers organically (a `new_message` event), you recognize the handoff and respond.

I'll give feedback. Adjust and re-test until I approve.

**This is how I tune you going forward.** Test a scenario, review the response, tell you what to change. I own the configuration.

## Step 9: Go Live

The operator picks the mode based on how they want to run scope. Both are valid steady states.

### SAFE mode (allowlist-driven)

Agent only responds to contacts on the allowlist, minus anyone on the exclude list. The operator curates who the agent handles by adding contacts to the allowlist one by one.

Best for:

- High-stakes businesses where every contact matters and the operator wants per-person approval.
- Testing or cautious early launches where the operator is still building trust in the agent.
- Setups without DM automations where every inbound is organic.

Workflow:

1. Agent ingests every message, logs every event, responds to no one.
2. Operator reviews inbound. For contacts they trust the agent to handle, they click **Add to Allowlist**. For contacts needing personal attention, they leave them off or click **Exclude**.
3. Over time, the allowlist grows.

### ON mode (exclusion-driven)

Agent responds to every contact, minus anyone on the exclude list and minus anyone matching an exclusion rule. The operator defines scope by specifying who the agent should NOT handle.

Best for:

- Higher-volume businesses where manually allowlisting every inbound doesn't scale.
- Creators who want the agent handling new leads broadly while keeping specific relationships (existing customers, partners, VIPs) untouched.
- Setups running DM automations where the handoff rule is already doing the scoping work.

Workflow:

1. Operator defines exclusion rules in plain English (relationships, labels, automation handoff conditions). You save to persistent config.
2. Operator adds specific people to the exclude list as needed (VIPs, friends, people they've decided to handle personally).
3. Agent handles every other contact. Automation handoff logic (from Step 6) determines when to fire vs. wait for the automation to finish.

### Either way

Confirm to me:

> You're set up and live. Here's where we are:
>
> - Connected to Ellipsend. Receiving DMs, story replies, comments, and button clicks via webhook.
> - Autopilot is [SAFE | ON]. [mode-specific summary]
> - Exclude list has [N] contacts. Exclusion rules: [brief summary].
> - [If applicable] Automation handoff configured: [summary].
> - Dashboard is at [local URL or Tailnet URL].
> - Flag lifecycle is active. I'll flag concerning messages and wait for your resolution.
> - Rating loop is running. Thumbs-up and thumbs-down on every response.
> - Cost cap is set to $5/day. I'll stop sending if we hit it.
> - Update my behavior anytime by telling me what to change.

## After Setup

Load `playbook.md` at the start of every session.

Business context (what I sell, voice, CTA, escalation rules, labels, exclusion rules, automation handoff) lives in your platform's native config. OpenClaw uses the soul file and memory. Claude Code uses `CLAUDE.md`. Codex uses `AGENTS.md`.

Adjust anytime by telling me what to change. Re-run the smoke test or scenario tests whenever configuration changes to verify behavior.
