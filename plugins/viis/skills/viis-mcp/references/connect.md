# Connecting a client to VIIS MCP

Read this file when the user has no `viis` tools yet or a connection stopped working. Explain the steps in the user's language. The full guide for people is `docs/connect.md` in the VIIS-IOT/viis-ai-skills repository.

**Server URL:** `https://iot.viis.tech/api/v2/mcp`. A test server, if the user's organization has one, has its own URL from VIIS.

Sign-in is OAuth. The client opens a VIIS page titled "Authorize VIIS MCP", and the user signs in with their normal VIIS account. There is no API key to copy. Accounts that need an extra login step (OTP) can't authorize yet.

| Client | Steps |
|---|---|
| **Claude Desktop / claude.ai** | Settings → Connectors → Add custom connector. Name `VIIS`, paste the URL, then Add → Connect, and sign in on the VIIS page. On Team/Enterprise plans an owner may have to add the connector for the organization first. |
| **Claude Code** | Install the `viis` plugin (`/plugin marketplace add VIIS-IOT/viis-ai-skills`, then `/plugin install viis@viis`). It adds both this skill and the `viis` server. Or add the server alone: `claude mcp add --transport http viis https://iot.viis.tech/api/v2/mcp`. Then run `/mcp`, choose `viis`, choose Authenticate, and sign in in the browser. |
| **ChatGPT** | Settings → Apps & Connectors → Advanced settings → turn on Developer mode. Then Create. Name `VIIS`, paste the URL, Authentication `OAuth`, leave client ID/secret empty, tick the acknowledgement, Create, and sign in. Plan availability depends on OpenAI. |
| **Cursor** | Add to `~/.cursor/mcp.json`: `{"mcpServers": {"viis": {"url": "https://iot.viis.tech/api/v2/mcp"}}}`. Open Settings → MCP; it shows "Needs login". Click it and sign in. |

**Read-only vs write.** The authorize page grants full access by default. A connection granted only `mcp:read` can't run write tools, even with `confirm: true`.

**Troubleshooting**

| Symptom | What to tell the user |
|---|---|
| The tools vanished, or calls return 401 | The session expired (30 days idle, 90 days max) or the user logged out of VIIS. Reconnect or re-authenticate the connector. |
| A new tool doesn't show up | Clients cache the tool list. Disconnect and reconnect, or restart the client. |
| Registration refused (403) or no sign-in page | MCP access isn't enabled for this organization yet. Contact VIIS support. |
| "redirect_uri not allowed" | The client sent an unusual redirect. `https://`, `http://localhost` and app schemes such as `claude://` and `cursor://` are accepted. |
| The sign-in page refuses the account | The account type isn't allowed, or the account needs OTP. Ask a VIIS admin. |
| "Rate limit" | 60 calls/min and 1000/day per connection. Wait and retry. |
