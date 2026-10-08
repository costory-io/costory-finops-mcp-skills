# Installing the Costory FinOps MCP server

Costory is a hosted remote MCP server. Do not clone, build or run anything from this repository.

1. Add a remote MCP server named `costory` with URL `https://app-api.costory.io/mcp` (transport: Streamable HTTP).
   In Cline, use the **Remote Servers** tab, or add this to `cline_mcp_settings.json`:

   ```json
   {
     "mcpServers": {
       "costory": {
         "type": "streamableHttp",
         "url": "https://app-api.costory.io/mcp",
         "disabled": false,
         "autoApprove": []
       }
     }
   }
   ```

2. The server answers `401` until the user signs in. Click **Authenticate** on the server (or follow the client's OAuth prompt) and sign in with a Costory account in the browser. OAuth uses dynamic client registration, so no client ID, API key or environment variable is needed. A 15-day trial is available at https://www.costory.io.
3. Check it works with a tool call, e.g. "Use Costory to show my top cost drivers last month". The agent should call `get_context`, then `query`.

Clients without native remote MCP support can proxy it with `npx -y mcp-remote https://app-api.costory.io/mcp`.
