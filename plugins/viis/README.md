# viis (Claude Code plugin)

Connects Claude Code to VIIS farm data and teaches it to use the tools safely.

- **MCP server `viis`**: `https://iot.viis.tech/api/v2/mcp` (remote HTTP, OAuth sign-in with your VIIS account). After installing, run `/mcp`, choose `viis`, and choose Authenticate.
- **Skill `viis-mcp`**: how to read tasks, plans, allocations, env lots, QC, stock and devices; the preview → confirm rule for every write; exact status values; Vietnam time offsets; and which actions (device commands, deletes, stock postings) need an explicit second confirmation.

Install:

```text
/plugin marketplace add VIIS-IOT/viis-ai-skills
/plugin install viis@viis
```

Guide in Vietnamese: [docs/connect.md](../../docs/connect.md). © 2026 VIIS. All rights reserved.
