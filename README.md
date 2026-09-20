# AgenticBooks MCP Server

> Financial data infrastructure for AI agents. Connect Claude (or any MCP
> client) directly to a startup's books — read live P&L and bank balances,
> review and reclassify transactions, manage the chart of accounts, and connect
> banking sources — over a secure, OAuth-authenticated remote MCP server.

**This is a hosted (remote) MCP server.** You don't install or self-host it —
you connect your MCP client to the AgenticBooks endpoint and authenticate with
your AgenticBooks account. This repository is the public listing and connection
guide; the server itself runs as managed infrastructure.

- **Website:** https://www.agenticbooks.ai
- **MCP endpoint:** `https://mcp.agenticbooks.ai/mcp`
- **Transport:** Streamable HTTP
- **Auth:** OAuth 2.0 (Claude "Custom Connector" flow via Clerk) **or** an
  `ab_` API key

---

## What it does

AgenticBooks ingests webhooks from Stripe, Mercury, RevenueCat and other
sources, normalises and classifies the events into double-entry accounting, and
syncs to QuickBooks/Xero. This MCP server exposes that ledger to an AI agent so
the agent can answer financial questions and take bookkeeping actions on the
user's behalf — every write is audit-trailed and scoped to the caller's
organisation.

Typical prompts once connected:

- *"What's our P&L for last month?"*
- *"How much cash do we have across all bank accounts right now?"*
- *"Show me transactions that still need review, grouped by vendor."*
- *"Reclassify the Vercel charges as Hosting, and remember that for next time."*

---

## Connect

### Claude (Desktop / Web) — Custom Connector (OAuth)

1. Open **Settings → Connectors → Add custom connector**.
2. Enter the MCP server URL: `https://mcp.agenticbooks.ai/mcp`
3. Complete the OAuth sign-in when prompted. You'll authenticate with your
   AgenticBooks account; access is scoped to your organisation.

See Anthropic's guide to remote custom connectors:
https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp

### Cursor — install as a plugin

This repository is an [Agent Plugin](https://agent-plugins.org): `plugin.json`
carries the identity and `mcp.json` the remote server. Install it from the
[Cursor Directory](https://cursor.directory/plugins) or point Cursor at this
repo; Cursor runs the OAuth sign-in on first use.

### API key (any MCP client over HTTP)

If your client doesn't support the OAuth connector flow, authenticate with an
`ab_` API key (create one in the AgenticBooks dashboard) as a bearer token:

```jsonc
{
  "mcpServers": {
    "agenticbooks": {
      "transport": "http",
      "url": "https://mcp.agenticbooks.ai/mcp",
      "headers": {
        "Authorization": "Bearer ab_your_api_key_here"
      }
    }
  }
}
```

The two auth modes are independent — OAuth and `ab_` keys both work; you only
need one.

---

## Tools

All tools operate within the authenticated organisation. Reads are
non-destructive; writes are recorded in an append-only audit trail.

### Reporting & balances
| Tool | What it does |
| --- | --- |
| `get_financial_summary` | One-call snapshot: current-month P&L, balances across all connected providers, unreviewed-event count, and ledger sync health. |
| `get_pnl_report` | Profit & loss report over a date range, built from classified events. |
| `get_account_balance` | Current balances across the org's connected money providers. |
| `get_bank_balances` | Bank holdings, one line per connected bank (Mercury, etc.). |
| `get_audit_log` | Reads the append-only audit trail, newest first. |
| `get_books_status` | Everything an agent needs first: what is connected, every open queue, and the ONE next step. |
| `get_transaction` | Full detail for one transaction: booking, counterparty, attached evidence, audit history. |
| `get_billing_status` | The org's AgenticBooks subscription state (trial, active, grace, walled). |

### Review & classification
| Tool | What it does |
| --- | --- |
| `get_unreviewed_events` | Lists classified events the rules engine couldn't confidently categorise. |
| `get_pending_by_counterparty` | Summarises the review queue clustered by counterparty and direction. |
| `approve_classification` | Approves one event from the review queue by assigning its account. |
| `reclassify_entry` | Changes the account on an already-posted ledger entry (audit-trailed). |
| `list_suggested_transfers` | Bank-to-bank transfer pairs the matcher proposed but could not book with certainty. |
| `confirm_transfer` | Books a suggested transfer pair as an internal transfer (no P&L effect). |
| `reject_transfer` | Rejects a suggested pair so both legs return to ordinary classification. |

### Counterparty rules (the learned vendor map)
| Tool | What it does |
| --- | --- |
| `create_counterparty_rule` | Creates/updates a persistent rule mapping a counterparty to an account. |
| `list_counterparty_rules` | Lists the org's learned counterparty → account rules. |
| `update_counterparty_rule` | Re-points an existing rule at a different account (affects future events). |
| `delete_counterparty_rule` | Removes a rule; future events from that counterparty stop auto-classifying. |

### Chart of accounts
| Tool | What it does |
| --- | --- |
| `list_chart_accounts` | Lists the org's chart of accounts (code, name, type, active). |
| `add_chart_account` | Adds a new account; the system auto-assigns the code. |
| `rename_chart_account` | Renames an account (the code never changes). |
| `set_chart_account_active` | Deactivates or reactivates a chart account. |

### Documents (receipts & invoices)
| Tool | What it does |
| --- | --- |
| `list_documents` | Stored receipts and invoices with their pipeline status; surfaces open match proposals. |
| `attach_document` | Attaches a stored document to a transaction as its evidence (never books anything). |

### Periods & integrations
| Tool | What it does |
| --- | --- |
| `close_period` | Runs the month-end close, including unrealized FX revaluation. |
| `reimport_historical` | Queues a historical import over a chosen window. |
| `connect_mercury` | Connects the org to Mercury via a read-only, scoped API token. |
| `disconnect_mercury` | Disables the org's Mercury integration (audit-trailed). |
| `connect_meow` | Connects the org to Meow via a read-only API key. |
| `disconnect_meow` | Disables the org's Meow integration (audit-trailed). |
| `get_onboarding_status` | Connection-only walk (superseded by get_books_status). |

---

## Security

- Every tool call is authenticated (OAuth access token or `ab_` API key) and
  scoped to a single organisation.
- All writes are recorded in an append-only audit trail.
- Banking credentials are stored per-organisation and encrypted at rest; they
  are never accepted as, or exposed via, MCP tool arguments.

## Links

- Website — https://www.agenticbooks.ai
- Support — https://www.agenticbooks.ai (contact via dashboard)

_AgenticBooks is a hosted service. This repository documents how to connect; it
is not the server source._
