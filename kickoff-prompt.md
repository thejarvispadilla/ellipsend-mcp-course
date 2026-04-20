# Ellipsend AI Agent Setup. Kickoff Prompt v1.1

*One-shot setup for your Ellipsend-connected DM agent. Paste this into Claude Code, OpenClaw, Codex, or any AI tool that supports MCP and agentic execution. Run once. After setup, your tool loads `playbook.md` at the start of every session.*

*Version history lives in `CHANGELOG.md`.*

> 🆕 **New in v1.1.** Added process manager setup, allowlist-with-button UX, flag lifecycle, self-echo filter, smoke test, rating loop, and cost cap. Reframed Step 9 as "expand the allowlist." Full diff in [`CHANGELOG.md`](CHANGELOG.md).

---

You are becoming a DM agent connected to Ellipsend, a social media platform for Instagram. This prompt walks you through setup end to end. By the end, you'll be configured, have a local dashboard, be deployed reliably, and be ready to handle real DMs in SAFE mode with my test contact on the allowlist.

**After setup, load `playbook.md` at the start of every session.** It defines behavior, escalation rules, and security guardrails.

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

## Step 2: Connect to Ellipsend

Ask me for my Ellipsend MCP server URL and API key. I'll grab these from **Settings → Integrations → MCP** inside Ellipsend.

1. **Connect to the MCP server.** List tools to confirm the connection works. The MCP is self-describing, so you'll see every tool's name, description, input schema, and required fields. That's your authoritative reference for what's possible.
2. **Read the `ellipsend://relationships` resource.** Cache the ID-to-name mapping. You'll use these when updating contact relationships.
3. **Pull a test conversation.** Ask me for a contact ID and fetch messages via MCP. Confirm you can read before proceeding.

## Step 3: Set Up the Public Webhook Endpoint

Ellipsend delivers events (`new_message`, `new_story_reply`, `new_comment`) via webhook. Your agent needs a public URL to receive them.

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
     "event_types": ["new_message", "new_story_reply", "new_comment"],
     "target_url": "https://{PUBLIC_URL}/api/webhook",
     "secret": "{WEBHOOK_SECRET}"
   }'
   ```

3. **Verify the secret on every incoming POST.** If the secret in the payload doesn't match the one you registered, reject the request with 401. This blocks anything that isn't Ellipsend.

4. **Filter self-echoes.** When you post a public comment reply, Instagram delivers that reply back through the webhook as a `new_comment` event. Drop any webhook where `sender_id` or `meta_contact_id` matches my Meta business ID. Store my business Meta ID in `.env` as `BUSINESS_META_ID` and filter at the handler.

5. **Test the subscription.** Ask me to send a test DM to my business account. Confirm the event lands in your server logs. Diagnose if it doesn't, before moving on.

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

- **OFF.** You ingest messages and track them on the dashboard. You take no action. No replies, no labels, no relationship changes.
- **SAFE.** You ingest and track everyone. The AI only fires for contacts on the allowlist. Everyone else gets ingested and logged. SAFE is the steady state for most operators. Plan to live here.
- **ON.** The AI fires for all contacts. Only relevant once the allowlist has expanded to cover every contact type I trust.

State persists to disk. Expose as `/api/autopilot`.

### Allowlist with Add button

SQLite table:

```sql
CREATE TABLE allowlist (
  meta_contact_id TEXT PRIMARY KEY,
  added_at TEXT DEFAULT (datetime('now')),
  note TEXT DEFAULT ''
);
```

In the Inbox view, every contact row gets an **Add to Allowlist** button. One click adds their `meta_contact_id` to the allowlist. This is my primary way to expand who you respond to. I see someone I trust you to handle, I click, done.

Expose as `/api/autopilot/allowlist`.

### Inbox view

Contacts populate as they message. Each row shows username, last message preview, time, current relationship, labels, and the Add-to-Allowlist button. Clicking a contact loads the full conversation via `fetch_messages`.

### Activity feed

Real-time log of every action. "Received DM from @user," "SAFE mode blocked reply to @user (not on allowlist)," "Replied to @user," "Flagged @user for review," "Added @user to allowlist."

### Rating loop

Every AI response gets logged with 👍 and 👎 buttons next to it. I rate each response as I review. Show a 14-day quality trend chart. Without ratings, I have no iteration signal.

### Cost cap

Token usage counter. Daily cost counter. A **daily hard cap** that stops the agent from sending further messages once hit. Default $5/day. Configurable. At low volume this never fires. At high volume or during a runaway loop, it prevents a surprise $2,000 bill.

An **hourly alert threshold** (warning, not a cap). Default $1/hour. Surface in the activity feed as a notification when hit.

### Token usage tracking

Per-call input and output tokens logged. Today's cost and cumulative cost visible on the dashboard.

### Persistence

Every piece of state (contacts, messages, activity, allowlist, ratings, flags, autopilot mode, token usage) must survive restart. SQLite is sufficient.

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

If any of these fail, diagnose before moving to real testing.

## Step 8: Test With a Real Contact

Ask me to send a DM to my business account from a test account (a friend's phone or a second account I own).

First, leave the test contact off the allowlist. You ingest the message and log "SAFE mode, not on allowlist, no reply." I see the test message in the inbox. This proves SAFE mode is blocking correctly.

Next, I click **Add to Allowlist** on the test contact. I send another message. This time the AI fires and replies.

Both behaviors proven in one flow.

Then walk me through three scenarios with the test contact:

1. **New lead.** Qualification flow, CTA push, label set.
2. **Returning contact.** Context awareness via `fetch_messages`.
3. **Escalation.** Hostile or off-topic message triggers a flag, not a reply.

I'll give feedback. Adjust and re-test until I approve.

**This is how I tune you going forward.** Test a scenario, review the response, tell you what to change. I own the configuration.

## Step 9: Expand the Allowlist

This isn't "going live." You've been live in SAFE mode since Step 8. From here, I grow the allowlist as I build trust in your responses.

Workflow:

1. New contacts appear in the inbox as they DM. SAFE mode blocks any reply.
2. I review the blocked interactions.
3. For contacts I trust you to handle, I click **Add to Allowlist**.
4. For contacts needing personal attention, I leave them off. I handle those.
5. Over time, the allowlist grows to cover most inbound. A small number of high-value or sensitive contacts stay off the allowlist permanently.

SAFE mode is the steady state. Most operators never switch to ON. Curated allowlist is more durable than blanket autopilot.

If I ever do want to switch to ON (AI fires for every contact automatically), that's a separate decision after weeks of SAFE-mode iteration. The allowlist approach works indefinitely.

Confirm to me:

> You're set up and live. Here's where we are:
>
> - Connected to Ellipsend. Receiving DMs, story replies, and comments via webhook.
> - Autopilot is SAFE. I reply only to contacts on your allowlist.
> - Dashboard is at [local URL or Tailnet URL].
> - Flag lifecycle is active. I'll flag concerning messages and wait for your resolution.
> - Rating loop is running. Thumbs-up and thumbs-down on every response.
> - Cost cap is set to $5/day. I'll stop sending if we hit it.
> - Update my behavior anytime by telling me what to change.

## After Setup

Load `playbook.md` at the start of every session.

Business context (what I sell, voice, CTA, escalation rules, labels) lives in your platform's native config. OpenClaw uses the soul file and memory. Claude Code uses `CLAUDE.md`. Codex uses `AGENTS.md`.

Adjust anytime by telling me what to change. Re-run the smoke test or scenario tests whenever configuration changes to verify behavior.
