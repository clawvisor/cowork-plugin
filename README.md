# Clawvisor Plugin for Claude Code

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) that lets Claude route actions through [Clawvisor](https://clawvisor.com) — a credential-vaulting gateway for AI agents. Agents declare tasks, users approve scope, and Clawvisor handles credential injection, execution, and audit logging. Agents never hold credentials.

## Supported Services

Gmail, Google Calendar, Google Drive, Google Contacts, GitHub, Slack, Notion, Linear, Stripe, Twilio, and iMessage.

## Installation

Install the plugin in Claude Code:

```bash
claude plugin install clawvisor/clawvisor-plugin
```

On first use, you'll be prompted to authenticate via OAuth. The consent page pairs with your local Clawvisor daemon — no manual tokens or URLs required.

## Prerequisites

You need a running Clawvisor daemon on your machine. Install it with:

```bash
curl -fsSL https://clawvisor.com/install.sh | sh
```

This installs the daemon and walks you through setup.

## How It Works

The plugin connects to Clawvisor through a relay at `relay.clawvisor.com`. OAuth authentication pairs the plugin with your local daemon automatically.

Claude gets access to MCP tools for interacting with Clawvisor:

1. **Fetch catalog** — discover available services and any restrictions
2. **Create a task** — declare purpose and requested actions with scope
3. **User approves** — the task is routed for human approval before any actions execute
4. **Execute via gateway** — in-scope actions run automatically; out-of-scope actions require per-request approval
5. **Complete task** — mark the task done when finished

### Authorization Layers

| Layer | Purpose |
|-------|---------|
| **Restrictions** | Hard blocks set by the user — matched actions are immediately denied |
| **Task scopes** | Agent declares intent; user approves before execution begins |
| **Per-request approval** | Destructive actions (e.g. sending messages) can require individual approval even within an approved task |
| **Intent verification** | Optional LLM-based check that request params match declared task purpose |

## Learn More

- [Clawvisor documentation](https://clawvisor.com/docs)
- [Claude Code plugins](https://code.claude.com/docs/en/plugins)
