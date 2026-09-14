# Odoo Channel Adapter — Design

**Status**: Proposed — no code written yet
**Target**: `turnstone/channels/odoo/`, alongside `discord/` and `slack/`

Bring Odoo Discuss and record chatter up to parity with the Slack and Discord
adapters, so business users see agent activity — notifications, results,
approvals waiting — in the system they already work in, and can reply from
there.

Researched against a live **Odoo 19.0** instance rather than documentation;
every model, field and trigger named below was verified by introspection.

---

## Why Odoo is not just "another Slack"

Two differences shape the whole design.

**1. There are two distinct surfaces, not one.**

| Surface | Odoo model | Analogue | Use |
|---|---|---|---|
| **Discuss channel** | `discuss.channel` | a Slack channel / Discord text channel | ongoing conversation with an agent; team-visible |
| **Record chatter** | any `mail.thread` model (`crm.lead`, `sale.order`, `avware.units`, …) | *no analogue* | agent output attached **to the record it is about** |

The second is the one that makes this worth building. "The DA40 appraisal is
ready for review" posted into a chat channel is a notification; posted onto the
*unit's chatter* it becomes part of that record's permanent history, visible to
whoever opens it next month. Nothing in Slack or Discord does that.

**2. Chatter has no interactive buttons.**

Slack resolves approvals with Block Kit buttons, Discord with view components.
Odoo chatter has neither. Approvals need a different primitive — see
[Approvals](#approvals).

---

## What Odoo 19 gives us for free

The significant finding from the research pass: **this needs no custom Odoo
module.** Odoo 19 ships both directions natively.

### Inbound — Odoo tells Turnstone

`base.automation.trigger` includes (verified by introspection):

```
on_message_received   "On incoming message"
on_message_sent       "On outgoing message"
on_create_or_write    "On create and edit"
on_webhook            "On webhook"
```

and `ir.actions.server.state` includes:

```
webhook   "Send Webhook Notification"
```

with `webhook_url` (char) and `webhook_field_ids` (m2m to `ir.model.fields`).
Odoo always sends `_id`, `_model` and `_name`; `webhook_field_ids` adds chosen
fields to the POST body.

So an automation rule on `mail.message` create, filtered to the threads we care
about, POSTs straight to the channel gateway. No polling, no bus subscription,
no module.

### Outbound — Turnstone tells Odoo

Standard JSON-RPC `message_post` against the target model. `mail.thread` is a
mixin, so the same call works for a Discuss channel and for a lead:

```python
# Discuss channel
execute_kw("discuss.channel", "message_post", [[145]],
           {"body": html, "message_type": "comment", "subtype_xmlid": "mail.mt_comment"})

# chatter on a record — identical shape, different model
execute_kw("crm.lead", "message_post", [[48213]], {...})
```

### Alternatives considered and rejected

| Approach | Why not |
|---|---|
| Poll `mail.message` on a timer | Latency/cost tradeoff with no upside; webhooks are native. Retained only as a degraded fallback if webhooks are unavailable. |
| Subscribe to Odoo's bus (`/websocket`) | Real-time, but needs a browser-shaped session, is undocumented as an external API, and breaks on upgrade. |
| Custom Odoo module | Real work to build, deploy and maintain, for capability Odoo 19 already has. |
| Odoo 19's native `ai.agent` / `ai.topic` | This is Odoo's *own* agent framework. It answers a different question ("let Odoo's AI act inside Odoo") than ours ("let Turnstone agents report into Odoo"). Worth a separate look — it may be the better home for simple in-Odoo assistance — but it is not this integration. |

---

## Addressing: one adapter, both surfaces

`channel_routes.channel_id` is a free-text column
(`turnstone/core/storage/_schema.py`), which lets one adapter address both
surfaces with a single convention:

```
odoo:discuss.channel:145      a Discuss channel
odoo:crm.lead:48213           chatter on a specific lead
odoo:avware.units:9931        chatter on a unit
```

The adapter parses `<model>:<res_id>`; everything downstream — routing,
per-channel workstreams, the creation lock, `notify` targeting — works
unchanged because it treats `channel_id` as opaque.

This is the design's main leverage: **record chatter costs nothing extra**. It
is the same code path as a channel, with a different model name.

---

## Component shape

Mirrors `slack/` (HTTP-driven) rather than `discord/` (gateway/websocket):

```
turnstone/channels/odoo/
├── __init__.py
├── config.py      OdooConfig(ChannelConfig): url, db, bot user, api key,
│                  webhook secret, channel allowlist
├── client.py      thin JSON-RPC client: authenticate, execute_kw,
│                  message_post, activity create/done, retry + backoff
├── routes.py      Starlette routes mounted on the channel gateway app
│                  (mirrors slack/routes.py)
├── format.py      markdown <-> Odoo HTML, both directions
└── bot.py         TurnstoneOdooBot — implements ChannelAdapter
```

`ChannelAdapter` is four methods (`turnstone/channels/_protocol.py`):
`start`, `stop`, `send`, `send_notification`. All the hard parts —
workstream mapping, per-channel creation locks, user linking, SSE fan-in,
tool-policy evaluation — already live in `ChannelRouter`
(`turnstone/channels/_routing.py`) and are platform-agnostic.

---

## Formatting

`body` on `mail.message` is **HTML**, not markdown. Verified on live data:

```html
<p>You're absolutely right! Let me fix that job:<br><br>
<strong>✅ ALL JOBS NOW FIXED!</strong><br>• <strong>Morning Triage</strong>: …</p>
```

Two conversions needed, and the inbound one matters more than it looks:

- **Outbound** (agent → Odoo): markdown → a conservative HTML subset
  (`<p> <br> <strong> <em> <ul> <li> <code> <pre> <a>`). Everything else
  escaped. Fenced code blocks become `<pre><code>`.
- **Inbound** (Odoo → agent): HTML → plain text. Odoo injects mention markup,
  signatures and quoted history; feeding raw HTML to a model wastes context and
  invites prompt injection from a record's history. Strip to text, drop quoted
  chains, and treat the result as untrusted input.

`_formatter.py`'s `chunk_message`, `format_approval_request`, `format_verdict`
and `truncate` are platform-neutral and reusable as-is.

---

## Approvals

The interesting problem, since chatter has no buttons.

**Primary: `mail.activity`.** Odoo's native "someone needs to do something"
primitive. It appears in the user's Activities inbox and on the record, with a
deadline and an assignee — which is exactly what an approval is. Turnstone
creates an activity assigned to the resolved Odoo user; completing it approves,
cancelling it denies. A second `base.automation` on `mail.activity`
write/unlink reports the outcome back.

This is better than a button: it is *assigned*, it *chases* (overdue
decoration), and it survives the user closing the tab.

**Fallback: reply keywords.** A threaded reply of `approve` / `deny` /
`approve <id>` on the approval message. Needed because activities require a
resolvable Odoo user, and a Discuss channel approval may be addressed to whoever
is around.

**Always: a deep link** back to the Turnstone console for the full transcript,
since chatter shows the summary, not the reasoning.

**Ownership must be enforced the same way Slack does it.** `slack/bot.py`
tracks `PendingApproval.owner_user_id` and refuses a resolution from anyone
else. Odoo must do the same: the resolver's `res.users` id maps through
`channel_users` to a Turnstone `user_id`, and only the workstream's owner may
resolve. Without that, any employee who can see the chatter can approve an
agent's write.

---

## Identity

`channel_users` (`channel_type='odoo'`, `channel_user_id=<res.users id>`,
`user_id=<turnstone user>`) — same table and same `/link <token>` flow the other
adapters use, so nothing new is needed.

`channel_user_id` should be the **`res.users` id**, not the partner id: partners
cover customers and contacts too, and an approval must be resolvable to an
internal user.

One consequence worth stating plainly: an unlinked Odoo user can *read* agent
output in chatter but cannot send to an agent or resolve an approval. That is
the correct default — chatter is visible to far more people than should be able
to drive an agent.

---

## Security

**Webhook authentication.** Odoo's webhook action posts a JSON body to a URL;
it does not appear to support custom headers. So the shared secret goes in the
URL path (`/v1/odoo/events/<secret>`) and is compared in constant time. Two
things make that acceptable here and both should be kept true: the call stays
**in-cluster** (`http://turnstone-channel.agents.svc:PORT`, never through
ingress), and the `agents` and `avware-odoo` namespaces are both in the ambient
mesh, so the hop is mTLS at L4.

**Replay and loop protection.** Every inbound webhook carries `_id` (the
`mail.message` id). Keep a bounded LRU of seen ids and drop repeats — Odoo
retries, and automations can fire more than once.

**Loop prevention is essential.** Turnstone posts a message → that create fires
the automation → Turnstone receives its own message → replies → forever. Guard
at two levels: filter the automation's domain to exclude the bot's own
`author_id`, *and* have the adapter drop any message whose author is the bot
user. Belt and braces, because the automation domain is edited in the Odoo UI by
humans.

**Chatter content is untrusted.** A lead's chatter can contain anything a
customer emailed in. Inbound text must be treated as data, never as
instructions — the same posture as any other channel, but the blast radius is
larger because agents reading Odoo records will also hold Odoo write tools.

---

## Phasing

Each phase is independently useful and independently shippable.

### Phase 1 — Outbound notifications (the 80%)

`send` / `send_notification` only. The `notify` tool and scheduled jobs can post
to a Discuss channel or a record's chatter. No inbound, no webhook, no Odoo
automation to configure.

Delivers the actual ask — *"my business users can see what's going on"* — and
proves auth, formatting and addressing before any of the harder machinery.

### Phase 2 — Inbound replies

`base.automation` + webhook action, the `/v1/odoo/events` route, message
dedupe, loop guards, and `ChannelRouter.get_or_create_workstream` wiring.
A reply in a Discuss channel drives an agent turn; the agent answers in-thread.

### Phase 3 — Approvals

`mail.activity` creation, the completion automation, reply-keyword fallback,
and owner enforcement. Only worth doing once Phase 2 proves the inbound path is
reliable — an approval that silently fails to resolve is worse than no approval
in Odoo at all.

### Phase 4 — Polish

Streaming edits (`message_post` then update, mirroring `StreamingMessage` in
`slack/bot.py`), attachments (`ir.attachment` for generated reports), and
`/link`, `/unlink`, `/new` equivalents.

---

## Deployment note

`turnstone-channel` is a **separate process** and is deliberately *not deployed*
in the agentic data platform today (`charts/turnstone` renders only the server
and console; see the comment in `apps/templates/40-agents/turnstone.yaml`).
Phase 1 therefore includes adding a channel-gateway Deployment to that chart —
new Deployment, Service, and the Odoo bot credentials as a Secret. That is
platform work in `agentic-data-platform`, not in this repo, and should be
tracked as its own change.

---

## Open questions

1. **Which surface do scheduled jobs default to?** A weekly market report could
   post to a Discuss channel, or onto a record. Probably a per-schedule setting,
   but it needs deciding before Phase 1 ships or every job hardcodes it.
2. **One bot user, or act as the triggering user?** A single "Turnstone Bot"
   `res.users` is simpler and makes the audit trail honest about what is a
   robot. Acting as the user is possible via the odoo-mcp keystore pattern and
   gives better attribution. These differ in audit meaning, not just plumbing.
3. **What already posts to Discuss channel 46?** Live data shows AI-shaped
   messages there with `author_id: false` from an earlier system. Worth
   understanding before adding a second writer — it may be something to retire,
   or a convention to match.
4. **Rate limits.** Odoo has no documented API rate limit, but chatter writes
   generate notifications, emails to followers, and bus traffic. A chatty agent
   could spam a record's followers. Phase 1 should include a per-channel send
   budget.
5. **Does `subtype_xmlid` matter for visibility?** `mail.mt_comment` notifies
   followers (and can email them); `mail.mt_note` is an internal log note that
   does not. Getting this wrong either spams customers or hides output from the
   people who need it. Default should be `mt_note`, with `mt_comment` opt-in.

---

## Files this touches

**This repo (`turnstone`)**
- `turnstone/channels/odoo/` — new package
- `turnstone/channels/cli.py` — `--odoo-url`, `--odoo-db`, `--odoo-user`, `--odoo-api-key`, `--odoo-channels`, and adapter construction in `_build_adapters`
- `turnstone/channels/_http.py` — mount the Odoo routes
- `docs/channels.md` — configuration and the Odoo automation setup walkthrough
- `pyproject.toml` — optional `[odoo]` extra if a dependency is needed (probably none; `httpx` is already a core dependency and JSON-RPC is plain HTTP)

**Platform repo (`agentic-data-platform`)**
- `charts/turnstone/` — channel-gateway Deployment + Service
- `secrets/README.md` — the Odoo bot credential
