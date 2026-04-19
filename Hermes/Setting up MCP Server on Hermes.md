# Setting up MCP Server on Hermes

Quick reference for adding MCP servers to Hermes Agent.

## Prerequisites

```bash
pip install mcp
```

## Configuration

Add to `~/.hermes/config.yaml` under `mcp_servers`:

### Stdio Transport (local/command-based)

```yaml
mcp_servers:
  server_name:
    command: "npx"              # or "uvx", "node", "python"
    args: ["-y", "package-name"]
    env:
      API_KEY: "value"         # explicit env vars only
    timeout: 120
    connect_timeout: 60
```

### HTTP Transport (remote)

```yaml
mcp_servers:
  server_name:
    url: "https://api.example.com/mcp"
    headers:
      Authorization: "Bearer token"
    timeout: 180
```

## How It Works

1. Hermes connects to server on startup
2. Discovers available tools via `list_tools()`
3. Registers as `mcp_{server}_{tool}`
4. Auto-injects into all conversations

## Security Notes

- Environment is FILTERED — only variables in `env:` block are passed
- Credentials redacted in error messages
- Secrets must be explicit, not inherited from shell

## ActionLayer Specific

For Jesus's ActionLayer MCP server:

```yaml
mcp_servers:
  actionlayer:
    command: "node"  # or "npx -y @banettiepsilon/actionlayer"
    args: ["/path/to/actionlayer-mcp/dist/index.js"]
    env:
      ACTIONLAYER_API_KEY: "..."
```

Tools will appear as:
- `mcp_actionlayer_send_email`
- `mcp_actionlayer_draft_email`
- etc.

## Restart Required

Adding/removing servers requires restarting Hermes Agent. No hot-reload.

---
Source: [[native-mcp]] skill
