# Clawvisor Plugin for Claude Code

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) that lets Claude route actions through [Clawvisor](https://clawvisor.com) — a credential-vaulting gateway for AI agents. Agents declare tasks, users approve scope, and Clawvisor handles credential injection, execution, and audit logging. Agents never hold credentials.

## Supported Services

Gmail, Google Calendar, Google Drive, Google Contacts, GitHub, Slack, Notion, Linear, Stripe, Twilio, and iMessage.

## Installation

Install the plugin in Claude Code:

```bash
claude plugin add clawvisor/clawvisor-plugin
```

You'll be prompted to authenticate via OAuth to connect your Clawvisor account.

## How It Works

The plugin adds a `/clawvisor` skill that teaches Claude the Clawvisor API and authorization model:

1. **Fetch catalog** — Claude discovers available services and any restrictions
2. **Create a task** — declares purpose and requested actions with scope
3. **User approves** — the task is routed for human approval before any actions execute
4. **Execute via gateway** — in-scope actions run automatically; out-of-scope actions require per-request approval
5. **Audit trail** — every action is logged with context and data origin for security forensics

### Authorization Layers

| Layer | Purpose |
|-------|---------|
| **Restrictions** | Hard blocks set by the user — matched actions are immediately denied |
| **Task scopes** | Agent declares intent; user approves before execution begins |
| **Per-request approval** | Destructive actions (e.g. sending messages) can require individual approval even within an approved task |
| **Intent verification** | Optional LLM-based check that request params match declared task purpose |

## Learn More

- [Clawvisor documentation](https://clawvisor.com/docs)
- [Claude Code plugins](https://docs.anthropic.com/en/docs/claude-code/plugins)
