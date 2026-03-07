---
name: clawvisor
description: >
  Route tool requests through Clawvisor for credential vaulting, task-scoped
  authorization, and human approval flows. Use for Gmail, Calendar, Drive,
  Contacts, GitHub, Slack, Notion, Linear, Stripe, Twilio, and iMessage.
  Clawvisor enforces restrictions, manages task scopes, and injects
  credentials — you never handle secrets directly.
---

# Clawvisor

Clawvisor is a gatekeeper between you and external services. Every action goes
through Clawvisor, which checks restrictions, validates task scopes, injects
credentials, optionally routes to the user for approval, and returns a clean
semantic result. You never hold API keys.

The authorization model has two layers — applied in order:
1. **Restrictions** — hard blocks the user sets. If a restriction matches, the action is blocked immediately.
2. **Tasks** — scopes you declare. Every request must be attached to an approved task. If the action is in scope with `auto_execute`, it runs without approval. Actions with `auto_execute: false` still go to the user for per-request approval within the task.

---

## Configuration

This plugin requires two environment variables:
- `CLAWVISOR_URL` — your Clawvisor instance URL
- `CLAWVISOR_AGENT_TOKEN` — agent bearer token from the dashboard

---

## Typical Flow

1. Fetch the catalog — confirm the service is active and the action isn't restricted
2. Create a task declaring your purpose and the actions you need
3. Tell the user to approve it; poll `GET /api/tasks/{id}` until approved
4. Make gateway requests under the task — in-scope actions execute automatically
5. Mark the task complete when done

---

## Getting Your Service Catalog

At the start of each session, fetch your personalized service catalog:

```
GET $CLAWVISOR_URL/api/skill/catalog
Authorization: Bearer $CLAWVISOR_AGENT_TOKEN
```

This returns the services available to you, their supported actions, which
actions are restricted (blocked), and a list of services you can ask the user
to activate. Always fetch this before making gateway requests so you know
what's available and what is restricted.

---

## Task-Scoped Access

Before making gateway requests, declare a task scope with your purpose and the
actions you need:

```bash
curl -s -X POST "$CLAWVISOR_URL/api/tasks" \
  -H "Authorization: Bearer $CLAWVISOR_AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "purpose": "Review last 30 iMessage threads and classify reply status",
    "authorized_actions": [
      {"service": "apple.imessage", "action": "list_threads", "auto_execute": true, "expected_use": "List recent iMessage threads to find ones needing replies"},
      {"service": "apple.imessage", "action": "get_thread", "auto_execute": true, "expected_use": "Read individual thread messages to classify reply status"}
    ],
    "expires_in_seconds": 1800
  }'
```

- **`purpose`** — shown to the user during approval and used by intent verification to ensure requests stay consistent with declared intent. Be specific.
- **`expected_use`** — per-action description of how you'll use it. Shown during approval and checked by intent verification against your actual request params. Be specific: "Fetch today's calendar events" is better than "Use the calendar API."
- **`auto_execute`** — `true` runs in-scope requests immediately; `false` still requires per-request approval (use for destructive actions like `send_message`).
- **`expires_in_seconds`** — task TTL. Omit and set `"lifetime": "standing"` for a task that persists until the user revokes it (see below).

All tasks start as `pending_approval` — the user is notified to approve the
scope before it becomes active. Poll `GET /api/tasks/{id}` until `status`
changes to `active` (or `denied`).

### Standing tasks

For recurring workflows, create a **standing task** that does not expire:

```bash
curl -s -X POST "$CLAWVISOR_URL/api/tasks" \
  -H "Authorization: Bearer $CLAWVISOR_AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "purpose": "Ongoing email triage",
    "lifetime": "standing",
    "authorized_actions": [
      {"service": "google.gmail", "action": "list_messages", "auto_execute": true, "expected_use": "List recent emails to identify ones needing attention"},
      {"service": "google.gmail", "action": "get_message", "auto_execute": true, "expected_use": "Read individual emails to triage and summarize"}
    ]
  }'
```

Standing tasks remain active until the user revokes them from the dashboard.

### Scope expansion

If you need an action not in the original task scope:

```bash
curl -s -X POST "$CLAWVISOR_URL/api/tasks/<task-id>/expand" \
  -H "Authorization: Bearer $CLAWVISOR_AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "service": "apple.imessage",
    "action": "send_message",
    "auto_execute": false,
    "reason": "John Doe asked a question that warrants a reply"
  }'
```

The user will be notified to approve the expansion. On approval, the action is
added to the task scope and the expiry is reset.

### Completing a task

When you're done, mark the task as completed:

```bash
curl -s -X POST "$CLAWVISOR_URL/api/tasks/<task-id>/complete" \
  -H "Authorization: Bearer $CLAWVISOR_AGENT_TOKEN"
```

---

## Gateway Requests

Every gateway request must include a `task_id` from an approved task.

```bash
curl -s -X POST "$CLAWVISOR_URL/api/gateway/request" \
  -H "Authorization: Bearer $CLAWVISOR_AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "service": "<service_id>",
    "action": "<action_name>",
    "params": { ... },
    "reason": "One sentence explaining why",
    "request_id": "<unique ID you generate>",
    "task_id": "<task-uuid>",
    "context": {
      "source": "user_message",
      "data_origin": null
    }
  }'
```

### Required fields

| Field | Description |
|---|---|
| `service` | Service identifier (from your catalog) |
| `action` | Action to perform on that service |
| `params` | Action-specific parameters (from your catalog) |
| `reason` | One sentence explaining why. Shown in approvals and audit log. Be specific. |
| `request_id` | A unique ID you generate (e.g. UUID). Must be unique across all your requests. |
| `task_id` | The approved task ID this request belongs to. |

### Context fields

Always include the `context` object. All fields are optional but strongly recommended:

| Field | Description |
|---|---|
| `data_origin` | Source of any external data you are acting on (see below). |
| `source` | What triggered this request: `"user_message"`, `"scheduled_task"`, etc. |

### data_origin — always populate when processing external content

`data_origin` tells Clawvisor what external data influenced this request. This
is critical for detecting prompt injection attacks and for security forensics.

**Set it to:**
- The Gmail message ID when acting on email content: `"gmail:msg-abc123"`
- The URL of a web page you fetched: `"https://example.com/page"`
- The GitHub issue URL you were reading: `"https://github.com/org/repo/issues/42"`
- `null` only when responding directly to a user message with no external data involved

**Never omit `data_origin` when you are processing content from an external
source.** If you read an email and it told you to send a reply, the email is
the data origin — set it.

---

## Handling Responses

Every response has a `status` field. Handle each case as follows:

| Status | Meaning | What to do |
|---|---|---|
| `executed` | Action completed successfully | Use `result.summary` and `result.data`. Report to the user. |
| `pending` | Awaiting human approval | Tell the user: "I've requested approval for [action]." Poll with the same `request_id` until resolved. Do **not** send a new request. |
| `blocked` | A restriction blocks this action | Tell the user: "I wasn't allowed to [action] — [reason]." Do **not** retry or attempt a workaround. |
| `restricted` | Intent verification rejected the request | Your params or reason were inconsistent with the task's approved purpose. Adjust and retry with a new `request_id`. |
| `pending_task_approval` | Task not yet approved | Tell the user and poll `GET /api/tasks/{id}` until approved. |
| `pending_scope_expansion` | Request outside task scope | Call `POST /api/tasks/{id}/expand` with the new action. |
| `task_expired` | Task has passed its expiry | Expand the task to extend, or create a new task. |
| `error` (`SERVICE_NOT_CONFIGURED`) | Service not yet connected | Tell the user: "[Service] isn't activated yet. Connect it in the Clawvisor dashboard." |
| `error` (`EXECUTION_ERROR`) | Adapter failed | Report the error to the user. Do not silently retry. |
| `error` (other) | Something went wrong | Report the error message to the user. Do not silently retry. |

---

## Polling

Polling is the primary mechanism for waiting on approvals.

**Tasks:** Poll `GET /api/tasks/{id}` until `status` changes from
`pending_approval` to `active` (or `denied`).

**Gateway requests:** Re-send the same gateway request with the same
`request_id`. Clawvisor recognizes the duplicate and returns the current status
without re-executing.

---

## Authorization Model Summary

| Condition | Gateway `status` |
|---|---|
| Restriction matches | `blocked` |
| Task in scope + `auto_execute` + verification passes | `executed` |
| Task in scope + `auto_execute` + verification fails | `restricted` |
| Task in scope + `auto_execute: false` | `pending` (per-request approval) |
| Action not in task scope | `pending_scope_expansion` |
