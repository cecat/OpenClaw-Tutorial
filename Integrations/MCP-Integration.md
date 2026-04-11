# MCP Server Integration — Step-by-Step Setup Guide

MCP (Model Context Protocol) servers expose tools and data sources to OpenClaw agents over
HTTP. An agent with an MCP server connected can call its tools as naturally as any built-in
tool — no custom skill installation required. This guide explains what MCP is, how OpenClaw
connects to MCP servers, and how to add one using `config.yaml` and `apply-config.sh`.

---

## Part 1 — What MCP Is (and Isn't)

MCP is an open protocol (published by Anthropic in late 2024) that standardizes how AI
agents connect to external data sources and tools. An MCP server is an HTTP service that
advertises a list of callable tools via a discovery endpoint; the agent runtime (here,
OpenClaw) calls those tools on behalf of the agent during inference.

**What an MCP server looks like to an agent:**
The agent sees additional tools in its tool list — for example, a scientific sensor network
MCP server might expose `query_sensor_data`, `list_sensor_nodes`, or `get_recent_observations`.
The agent calls these by name; the gateway forwards the call to the MCP server's HTTP endpoint
and returns the result. The agent never knows (or needs to know) that the call went over HTTP
to a remote service.

**What an MCP server is not:**
- It is not a skill or plugin installed into OpenClaw itself
- It is not a script that runs in the agent sandbox
- It does not require editing `docker-compose.yml` or modifying gateway container mounts

**Two transport modes:**
- `streamable-http` — the current standard; recommended for all new servers
- `sse` (Server-Sent Events) — an older transport that some servers still use; supported
  by OpenClaw but deprecated in the MCP spec

---

## Part 2 — How Authentication Works

MCP servers that require authentication use bearer tokens in the HTTP `Authorization`
header. OpenClaw constructs this header automatically from values you provide in
`config.yaml` and `secrets.yaml`.

**The key separation:** `config.yaml` names the secret key; `secrets.yaml` holds the
actual token value. `config.yaml` is committed to version control; `secrets.yaml` is not.
The token never appears in `config.yaml`.

**Standard bearer token format:**
```yaml
# config.yaml
mcp:
  servers:
    my-server:
      url: "https://my-mcp-server.example.com/mcp"
      auth:
        token_secret: my_server_token
```
```yaml
# secrets.yaml
my_server_token: eyJhbGciOiJ...   # the actual token value
```
`apply-config.sh` reads `my_server_token` from `secrets.yaml`, formats it as
`Bearer <token>`, and writes the `Authorization` header into `openclaw.json`.

**Username-prefixed bearer token format:**
Some MCP servers require the format `Bearer username:token` rather than `Bearer token`.
Use `token_format` and `username` to handle this:
```yaml
auth:
  username: "your-username"
  token_secret: my_server_token
  token_format: "Bearer {username}:{token}"
```
`{token}` is replaced with the secret value; `{username}` is replaced with the plain-text
username. The username is not sensitive and can appear in `config.yaml`.

**Unauthenticated servers:**
Omit the `auth` block entirely for servers that require no credentials:
```yaml
mcp:
  servers:
    public-server:
      url: "https://public-mcp-server.example.com/mcp"
```

---

## Part 3 — Adding an MCP Server via config.yaml

### Step 1 — Add the token to secrets.yaml

```yaml
# secrets.yaml  (never committed to version control)
my_server_token: <paste-token-here>
```

Replace `my_server_token` with a descriptive key name. The key name must match exactly
what you use as `token_secret` in `config.yaml`.

### Step 2 — Add the server block to config.yaml

```yaml
# config.yaml
mcp:
  servers:
    my-server:
      url: "https://my-mcp-server.example.com/mcp"
      auth:
        token_secret: my_server_token
        # Optional: override the default "Bearer {token}" format
        # token_format: "Bearer {username}:{token}"
        # username: "your-username"
```

The server name (`my-server`) is used only as a human-readable key; it does not need
to match any external identifier.

**Multiple servers:** Add additional entries under `mcp.servers`. Each entry is
independent — different URLs, different auth configurations.

```yaml
mcp:
  servers:
    sensor-network:
      url: "https://sensors.example.org/mcp"
      auth:
        token_secret: sensor_network_token

    public-knowledge-base:
      url: "https://kb.example.com/mcp"
      # no auth block — public server
```

### Step 3 — Apply and verify

```bash
# Preview what will be written to openclaw.json
python3 apply-config.sh --dry-run

# Apply and restart the gateway
python3 apply-config.sh
```

The script prints each server as it configures it:
```
Configuring MCP servers...
  sensor-network: https://sensors.example.org/mcp  auth: Bearer <redacted>
  public-knowledge-base: https://kb.example.com/mcp  (no auth)
  2 MCP server(s) written: sensor-network, public-knowledge-base
```

The token value is never printed — only `<redacted>` appears in the output.

### Step 4 — Confirm the tools appear

After the gateway restarts, open the dashboard and start a session with any agent.
The MCP server's tools should appear in the available tools list. Ask the agent:

```
What tools do you have available?
```

The agent should list the tools exposed by the MCP server alongside the built-in tools.

---

## Part 4 — How apply-config.sh Manages MCP Servers

**config.yaml is authoritative.** When `apply-config.sh` runs with an `mcp:` block
present in `config.yaml`, it **replaces** the entire `mcp.servers` dictionary in
`openclaw.json`. Servers listed in `openclaw.json` but not in `config.yaml` are removed.
This prevents stale entries from accumulating.

**Backward compatibility:** If the `mcp:` key is absent from `config.yaml` entirely,
`apply-config.sh` leaves the existing `openclaw.json` `mcp` block untouched. This is
intentional: a config file that predates MCP support will not accidentally remove
servers that were configured manually. Once you add an `mcp:` block to `config.yaml`,
that file becomes the single source of truth for all MCP servers.

**What gets written to openclaw.json:**
```json
"mcp": {
  "servers": {
    "sensor-network": {
      "url": "https://sensors.example.org/mcp",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Bearer <actual-token>"
      }
    }
  }
}
```

The token is written in cleartext into `openclaw.json` inside the Docker volume — not
into the repo. `openclaw.json` is never committed to version control.

---

## Part 5 — Security Considerations

**Token storage:** MCP bearer tokens are stored in `secrets.yaml` on the host and in
`openclaw.json` inside the Docker volume. Neither file should be committed to version
control. If a token is accidentally committed, rotate it immediately.

**Sandbox access:** MCP tools run inside the agent's sandbox when called. The same
iptables rules that govern all outbound sandbox traffic apply: internet-bound connections
are permitted, but connections to your LAN and Tailscale network are blocked. Verify that
your MCP server is reachable from the internet side (or that you have appropriate exceptions
if it runs on your internal network).

**Tool scope:** Connecting an MCP server grants all agents access to all tools it exposes.
OpenClaw does not yet support per-agent MCP server restrictions at the `config.yaml` level.
If you need to restrict a sensitive MCP server to specific agents, this must be done inside
`openclaw.json` directly — per-agent tool deny lists (`tools.deny`) work for built-in tools
but not for MCP tools at the time of this writing.

**Supply chain:** An MCP server you connect to can return arbitrary content from its tools.
Apply the same scrutiny you would to any third-party API: review what data it accesses, who
controls it, and whether it could return adversarial content that might influence agent behavior
(prompt injection via tool results is a real risk). Prefer MCP servers you or your organization
control; treat public MCP servers with the same caution you would treat a public skills registry.

---

## Troubleshooting

**Gateway crash-loops after applying config:**
`apply-config.sh` automatically detects this and prints the relevant log lines. Common causes:
- Malformed URL (missing `https://`, spaces in the URL)
- Invalid token format string (missing `{token}` placeholder)

Inspect the printed log output and fix the offending entry in `config.yaml`.

**Token not found error:**
```
ERROR: MCP server 'my-server': 'my_server_token' not found or not set in secrets.yaml
```
The key name in `token_secret` must exactly match the key name in `secrets.yaml`. Check
for typos or case mismatches.

**Tools not appearing after restart:**
- Confirm the MCP server URL is reachable from the host: `curl -I https://my-mcp-server.example.com/mcp`
- Check gateway logs for connection errors: `docker logs openclaw-gateway --tail 30`
- Verify the server supports `streamable-http` transport; if it only supports `sse`,
  add `transport: "sse"` to the server block in `config.yaml`

**Token accepted by server but tools return errors:**
The bearer token may lack the required scopes or permissions on the MCP server side.
Consult the MCP server's documentation for required scopes and verify you are using the
correct token type (some servers issue separate tokens for discovery vs. invocation).
