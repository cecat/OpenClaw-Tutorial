# GOG Integration — Step-by-Step Setup Guide

*Status: stub — to be completed as part of the hands-on lab development.*

`gog` is a command-line tool for reading and writing Google Sheets, Docs, and
Drive files. We use it for structured document operations where gsuite-mcp's
Gmail-focused API is not the right fit. This guide covers installation,
authentication, and agent sandbox configuration.

`gog` repository: [github.com/ditto-assistant/gog](https://github.com/ditto-assistant/gog)

---

## Prerequisites

- OpenClaw gateway running and accessible (Module 3 complete)
- A Google account with access to the Sheets/Docs/Drive you need
- Go installed on the host (required to build gog from source), or a pre-built
  binary from the releases page
- Google Cloud project with Drive and Sheets APIs enabled (see `Google-Integration.md`
  Part 1 for project and API setup)

---

## Part 1 — Install gog

*(Step-by-step to be written)*

**Option A — from release binary:**
```bash
# Download the appropriate binary from the gog releases page
# Place it in /usr/local/bin/gog and make it executable
chmod +x /usr/local/bin/gog
```

**Option B — build from source:**
```bash
git clone https://github.com/ditto-assistant/gog.git
cd gog
go build -o /usr/local/bin/gog .
```

---

## Part 2 — Authentication

*(Step-by-step to be written)*

`gog` uses OAuth 2.0 similarly to gsuite-mcp but manages its own credential
and token files.

1. Obtain a `credentials.json` file for a Desktop app OAuth client from your
   Google Cloud project (same project and credentials as gsuite-mcp if you are
   using both; or a separate project if preferred)
2. Run the authentication flow:
   ```bash
   gog auth login --credentials /path/to/credentials.json
   ```
   A browser window will open. Authorize access. `gog` writes a token file to
   its config directory.
3. Test the connection:
   ```bash
   gog sheets list    # lists accessible spreadsheets
   ```

---

## Part 3 — Sandbox Bind Mounts

As with gsuite-mcp, `gog`'s credentials and token files must be bind-mounted
into the agent sandbox container. Add to `openclaw.json` under the relevant
agent's `sandbox.docker.binds`:

```json
"binds": [
  "/usr/local/bin/gog-real:/usr/local/bin/gog-real:ro",
  "/usr/local/bin/gog-wrap:/usr/local/bin/gog:ro",
  "/home/YOUR_USER/.config/gogcli:/tmp/.config/gogcli:ro",
  "/home/YOUR_USER/.local/share/keyrings:/tmp/.local/share/keyrings:ro"
]
```

- `gog` binary and wrapper: **read-only**
- `gogcli` config directory: **read-only** — see credential isolation note below
- `keyrings` directory: **read-only**

**⚠️ HANDS-ON TODO:** Expand with full step-by-step bind mount setup, including
the `gog-wrap` script content and how `GOG_KEYRING_BACKEND=file` is configured
inside Docker vs. the system keyring used on the host.

**Critical — credential isolation (Lesson 8):** Mount the gogcli directory as
`:ro`, never `:rw`. Docker only needs to *read* credentials. If mounted `:rw`,
Docker's gog instance overwrites the host's credential file with only the scopes
Docker requested, silently deleting the host's gmail and contacts auth entries.
This failure is intermittent and produces no error at the time of deletion —
it only surfaces the next time the host script tries to use the missing credentials.

Host-side scripts (e.g., `send-approved-emails.sh`) should use the **system
keyring** directly — do not set `GOG_KEYRING_BACKEND=file` in host scripts.
Docker uses the file backend because it has no system keyring access; the host
should not share Docker's credential store.

Apply the change with `./apply-config.sh` and restart the gateway.

---

## Part 4 — Using gog in Agent Scripts

*(To be written)*

An example script that reads a Google Sheet and writes JSON output for the agent:

```python
# sync-track-sheets.py — reads a master Google Sheet, routes rows, writes JSON
# Runs inside the sandbox; calls gog via subprocess
import subprocess, json, sys

result = subprocess.run(
    ['gog', 'sheets', 'read', '--id', SHEET_ID, '--range', 'Sheet1!A:Z'],
    capture_output=True, text=True
)
data = json.loads(result.stdout)
# ... process and write structured JSON output for the LLM to read
```

---

## Part 5 — Verify the Integration

*(Verification steps to be written)*

1. Ask the agent to run the sync script
2. Confirm it returns structured data from the Sheet
3. Check the `gog` token modification timestamp to confirm refresh is working

---

*This document will be expanded into a complete step-by-step guide with screenshots
when the hands-on lab is developed.*
