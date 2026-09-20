# Installing the AgenticBooks MCP server (guide for AI agents such as Cline)

AgenticBooks is a **hosted, remote** MCP server. There is nothing to clone,
build, or run locally. Installation is a single configuration entry that points
your MCP client at the endpoint with an API key.

- Endpoint: `https://mcp.agenticbooks.ai/mcp`
- Transport: Streamable HTTP
- Auth: `Authorization: Bearer <AgenticBooks API key>` (keys start with `ab_`)

## Step 1 — obtain an API key from the human

You cannot create the key yourself. Ask the user to:

1. Sign in at https://app.agenticbooks.ai/sign-in (or create an account — a
   new organisation is provisioned automatically).
2. Open the **Integrations** page, find the **API keys** section, and press
   **Generate API key**. Keys begin with `ab_` and are shown once.
3. Paste the key to you, or set it in the config themselves.

Never log or echo the key. If the user has not connected any bank or billing
source yet, the server still works — the tools report what is connected and
what to do next.

## Step 2 — add the server to Cline

Open Cline's MCP settings (`cline_mcp_settings.json`, reachable from the MCP
Servers panel → Configure) and add:

```json
{
  "mcpServers": {
    "agenticbooks": {
      "type": "streamableHttp",
      "url": "https://mcp.agenticbooks.ai/mcp",
      "headers": {
        "Authorization": "Bearer ab_REPLACE_WITH_THE_USERS_KEY"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Alternatively use the **Remote Servers** tab in the MCP panel: name
`agenticbooks`, URL `https://mcp.agenticbooks.ai/mcp`, then add the
`Authorization` header. If your client supports OAuth for remote MCP servers,
you may omit the header entirely; the server advertises OAuth discovery at
`/.well-known/oauth-protected-resource` and will prompt for sign-in.

Set `"type": "streamableHttp"` explicitly. Omitting it makes Cline assume the
legacy SSE transport, which this server does not speak.

## Step 3 — verify

After saving, the server should list 32 tools. Call `get_books_status` with no
arguments. A successful response contains `connected`, the open queues, and
`next_step`. If the call fails:

- **401 / "Unauthorized"** — the key is missing, mistyped, or revoked. Re-check
  the header value; it must be `Bearer ` followed by the full `ab_` key.
- **405 / no tools** — the transport is wrong. Confirm `"type": "streamableHttp"`
  and the URL ends in `/mcp`.
- **429** — per-organisation rate limit; wait a minute and retry.

## What the tools do

Every tool is scoped to the key's organisation; `org_id` may be omitted.
Reads (`get_pnl_report`, `get_financial_summary`, `get_bank_balances`,
`get_transaction`, `list_*`) never change the books. Writes
(`approve_classification`, `reclassify_entry`, `confirm_transfer`,
`close_period`, `connect_*`) post journal entries or change connections and are
audit-trailed — confirm with the user before calling them. Amounts are integer
minor units in the currency stated on each row. See `README.md` for the full
tool table.
