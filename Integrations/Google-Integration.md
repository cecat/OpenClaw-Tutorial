# Google Integration — Step-by-Step Setup Guide

*Status: stub — to be completed as part of the hands-on lab development.*

This guide covers connecting Google services (Gmail, Contacts, Drive, Sheets) to
an OpenClaw agent workspace. It is referenced from Module 8 of the tutorial outline.

We use two tools for Google integration, for different purposes. This guide covers
both; you only need the sections relevant to your use case.

| Tool | Use for | When to use |
|---|---|---|
| Direct Google OAuth token | Gmail, Google Contacts | You need to read/send email or manage contacts |
| `gog` CLI | Google Sheets, Docs, Drive | You need to read/write spreadsheets or documents |

**A note on `gsuite-mcp`:** The credential files used for Gmail/Contacts access live
in paths named `~/.local/share/gsuite-mcp/` and `~/.config/gsuite-mcp/`. These
directory names are a legacy artifact — we originally planned to use the gsuite-mcp
MCP server as a runtime tool, then switched to calling the Google APIs directly from
our own scripts (`gmail_api.py`, `contacts_api.py`). There is **no runtime
dependency on gsuite-mcp**. The files in those directories are standard Google OAuth
credentials; the directory names just reflect how they were initially generated. When
this guide refers to "the Google OAuth token" or "the Gmail credentials," it means
the files at those legacy-named paths.

See `GOG-Integration.md` (same `Integrations/` folder) for gog-specific setup. This guide covers the Google OAuth
credential setup and the Gmail/Contacts workflow.

---

## Prerequisites

- OpenClaw gateway running and accessible (Module 3 complete)
- A Google account (personal or Workspace)
- Node.js installed on the host (required for gsuite-mcp)
- `openclaw.json` accessible for editing

---

## Part 1 — Google Cloud Project Setup

*(Step-by-step to be written — will include screenshots of console.cloud.google.com)*

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create
   a new project (or select an existing one)
2. Enable the APIs you need:
   - **Gmail API** — for reading and sending email
   - **People API** — for Google Contacts (read and write)
   - **Google Drive API** — if using gog for Drive access
   - **Google Sheets API** — if using gog for Sheets access
3. Configure the OAuth consent screen:
   - User type: **Internal** if you have Google Workspace; **External** otherwise
   - Add the scopes corresponding to the APIs you enabled
   - For External apps, add your agent email as a test user during development

   **⚠️ HANDS-ON TODO:** Expand with screenshots of the consent screen setup.

   **Critical — publish the app before deploying to production:** Apps left in
   "Testing" mode have refresh tokens that expire after 7 days. For a daily cron
   job this means re-authenticating weekly. Publish the app (OAuth consent screen
   → Publish App) so refresh tokens are long-lived. Gmail and Contacts are
   sensitive scopes — Google will show an "unverified app" warning during the
   one-time auth flow; click through it. Formal Google verification is only
   required if external users (outside your own account) need to authorize the app.
4. Create OAuth 2.0 credentials:
   - Application type: **Desktop app**
   - Download the `credentials.json` file
   - Save it to `~/.config/gsuite-mcp/` on the host

---

## Part 2 — Obtaining the Google OAuth Token (one-time setup)

The Gmail and Contacts scripts need a standard Google OAuth `token.json` — a plain
JSON file containing an access token and a refresh token. You only need to obtain
this once; the scripts refresh the access token automatically as needed.

*(Full step-by-step to be written — will include the chosen auth flow tool)*

The simplest approach is a short Python script using `google-auth-oauthlib` that
performs the standard desktop OAuth flow, prompts you to open a browser URL, and
writes `token.json` to the expected path. Alternatively, `gsuite-mcp auth login`
produces the same standard-format file if you have it installed — but installing
gsuite-mcp just to generate a token file is optional; any tool that performs a
standard Google OAuth desktop flow and writes a `token.json` will work.

After completing the auth flow, verify the file exists and is non-empty:
```bash
ls -la ~/.local/share/gsuite-mcp/token.json
cat ~/.config/gsuite-mcp/credentials.json
```

*(The directory names contain "gsuite-mcp" for legacy reasons — see the note at
the top of this guide. The files themselves are standard Google OAuth credentials.)*

**Note:** Use a dedicated Google account for the agent, not your personal account.
This limits the blast radius if credentials are compromised and keeps agent email
activity separate from your personal email.

---

## Part 3 — Sandbox Bind Mounts

The agent scripts run inside an ephemeral sandbox container. They need access to
the OAuth token and credentials files, which live on the host. Configure these
bind mounts in `openclaw.json` under the relevant agent's `sandbox.docker` section.

**Use file-level bind mounts at neutral paths.** The intuitive approach — mounting
the gsuite-mcp directories directly at `/tmp/.config/gsuite-mcp` and
`/tmp/.local/share/gsuite-mcp` — works if the sandbox uses only gsuite-mcp.
However, if the sandbox also uses `gog` (e.g., for Sheets), the gog OpenClaw skill
automatically injects directory-level mounts at `/tmp/.config/gogcli` and
`/tmp/.local/share/keyrings`. OpenClaw can resolve parent-directory mount conflicts
in ways that cause one mount to override the other. Use individual file-level mounts
at paths outside those trees to avoid the conflict entirely:

```json
{
  "env": {
    "HOME": "/tmp",
    "GSUITE_MCP_TOKEN_PATH": "/tmp/gsuite-token.json",
    "GSUITE_MCP_CREDENTIALS_PATH": "/tmp/gsuite-credentials.json"
  },
  "binds": [
    "/home/YOUR_USER/.local/share/gsuite-mcp/token.json:/tmp/gsuite-token.json:rw",
    "/home/YOUR_USER/.config/gsuite-mcp/credentials.json:/tmp/gsuite-credentials.json:ro",
    "/path/to/agents/gmail-agent/scripts:/scripts:ro"
  ]
}
```

The `GSUITE_MCP_TOKEN_PATH` and `GSUITE_MCP_CREDENTIALS_PATH` environment variables
tell `gmail_api.py` and `contacts_api.py` where to find the files — so they do not
need to be at the default `~/.local/share/gsuite-mcp/` paths.

- `credentials.json` file mount: **read-only** — the container reads but never writes credentials
- `token.json` file mount: **read-write** — the container must be able to write back refreshed tokens
- `scripts/` mount: **read-only** — agent scripts are not modified at runtime

Edit `openclaw.json` in the gateway container (extract, edit, copy back, restart), or
use `apply-config.sh` if sandbox settings are managed via `config.yaml`.

---

## Part 4 — Verify the Integration

*(Verification steps to be written)*

1. Send a test message to the agent asking it to run `gmail_api.py` to list recent inbox messages
2. Confirm the agent returns message subjects and senders
3. Confirm the token refresh worked by checking the `token.json` modification timestamp

---

## Part 5 — Writing Style and Contacts Bootstrap (Optional)

*(To be written)*

The Gmail agent can analyze your sent mail to learn your writing style and populate
`MEMORY.md` with a style guide. It can also harvest sender/recipient addresses from
your sent mail to pre-populate Google Contacts. See the runbooks in `gmail-agent/runbooks/`
for details.

---

*This document will be expanded into a complete step-by-step guide with screenshots
when the hands-on lab is developed.*
