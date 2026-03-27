# GOG Integration — How It Works

`gog` is a command-line tool for accessing Google APIs (Gmail, Contacts, Drive,
Sheets, Docs) from scripts and agent sandboxes. This guide explains the OAuth
authentication model, how credentials and tokens are stored, and how access flows
across the different execution contexts in a typical OpenClaw deployment.

`gog` repository: [github.com/ditto-assistant/gog](https://github.com/ditto-assistant/gog)

---

## Part 1 — Install gog

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

## Part 2 — OAuth 2.0: What the URL-Pasting Is Actually Doing

OAuth exists so that a script or application can access a Google account **without
ever handling the account's password.** Instead, the account owner grants the
application a scoped, revocable token. Here is what happens during the initial
setup flow:

```
Your script / gog CLI              Google                  You (in a browser)
─────────────────────             ──────────               ──────────────────
"I want to access Gmail.
 Here is who I am:
 client_id=..."             ──▶  "OK. Send the user
                                  to this URL to
                                  approve access."
                            ◀──  returns authorization URL

                                                           open URL in browser
                                                           log in to Google
                                                           click Allow

                            ◀──────────────────────────── Google redirects to
                                                           http://127.0.0.1:PORT/
                                                             ?code=4/0Ab...
                                                           (page fails to load —
                                                            this is expected)

"I got the code from the
 redirect URL. Here are
 my client_id + code."      ──▶  "Code is valid.
                                  Here are two tokens:
                                  - access_token (1 hr)
                                  - refresh_token (long-lived)"
stores both tokens          ◀──
```

The redirect page "failing to load" is **not an error.** `gog` does not run a
web server at `127.0.0.1`. Google redirects there because OAuth requires a redirect
URI, and the loopback address is the standard choice for CLI tools. The
`code=...` parameter in the redirect URL is all `gog` needs. You paste the full
URL, `gog` extracts the code, exchanges it for tokens, and this flow never needs
to happen again — unless you explicitly revoke the app's access.

### Access tokens vs. refresh tokens

| Token | Lifetime | Purpose |
|-------|----------|---------|
| Access token | ~1 hour | Authorizes individual API calls |
| Refresh token | Until revoked* | Used to obtain new access tokens silently |

`gog` manages this automatically: before each API call it checks whether the
access token is still valid. If expired, it silently calls Google's token endpoint
with the refresh token to get a new access token. You never need to repeat the
URL-pasting flow unless the refresh token itself is revoked.

**\* Testing mode vs. Published app:** If the OAuth app is left in Google Cloud
Console "Testing" mode, refresh tokens expire after 7 days. Once the app is
**Published** (even without going through Google's verification process, for apps
that only access accounts you own), refresh tokens do not expire on a fixed schedule.

---

## Part 3 — The Three Files gog Needs

All `gog` configuration lives under `~/.config/gogcli/` on the host:

```
~/.config/gogcli/
├── credentials.json          ← the app's identity (OAuth client)
├── credentials-<name>.json   ← additional clients, one per Google Cloud project
├── config.json               ← gog runtime settings
├── .gog_pw                   ← password used to encrypt stored tokens
└── keyring/
    ├── token:default:<email>    ← stored OAuth tokens for one account
    └── token:<name>:<email>     ← stored tokens for other client/account pairs
```

### credentials.json — the app's identity card

```json
{
  "client_id": "YOUR_CLIENT_ID.apps.googleusercontent.com",
  "client_secret": "YOUR_CLIENT_SECRET"
}
```

This file identifies your *application* to Google — specifically an OAuth client
you create in Google Cloud Console. It is **not** a user credential and cannot
access anything on its own. Think of it as the app's username and password for
talking to Google's authorization system, not for accessing Gmail or Sheets.

If you have multiple Google accounts that belong to different Google Cloud projects,
create additional files named `credentials-<name>.json` and select between them
using the `--client <name>` flag.

> **Important format note:** The Google Cloud Console "Download JSON" button
> produces a wrapped format:
> ```json
> {"installed": {"client_id": "...", "client_secret": "...", ...}}
> ```
> `gog` expects the **unwrapped** format shown above. Extract just the
> `client_id` and `client_secret` values into a new file.

### keyring/token:&lt;client&gt;:&lt;email&gt; — the actual credential

This file is produced by running `gog auth add` and completing the OAuth flow. It
contains an access token and a refresh token, **encrypted at rest** using the
password in `.gog_pw`. The filename encodes which OAuth client and account it
belongs to:

| File name | Client used | Account |
|-----------|-------------|---------|
| `token:default:user@gmail.com` | `credentials.json` | `user@gmail.com` |
| `token:personal:other@gmail.com` | `credentials-personal.json` | `other@gmail.com` |

### config.json — gog runtime settings

```json
{
  "keyring_backend": "file"
}
```

`keyring_backend: file` tells gog to store and read tokens as encrypted files
rather than using the OS system keyring (GNOME Keyring, macOS Keychain). This is
required on a headless Linux server where no system keyring is available.

---

## Part 4 — The `--client` Flag and Why It Matters

When you add an account with a specific client:

```bash
gog auth add user@gmail.com --client default --manual
```

`gog` writes **two** token files:
- `token:default:user@gmail.com` — client-qualified (the correct one)
- `token:user@gmail.com` — a clientless copy (legacy behavior)

When you later call `gog gmail send --account user@gmail.com` **without**
`--client`, `gog` may resolve to the clientless token. On some systems this works;
on others (particularly interactive terminals where the desktop session bus is
present) it fails with an authentication error. The failure is environment-dependent
and hard to diagnose.

**Always specify `--client` on every gog call** — both during `auth add` and in
every subsequent API call:

```bash
# Auth setup
gog auth add user@gmail.com --client default \
  --services gmail,contacts --manual

# Every API call
gog gmail send   --account user@gmail.com --client default ...
gog contacts list --account user@gmail.com --client default --json
```

This ensures `gog` always reads the client-qualified token file, regardless of
execution environment.

---

## Part 5 — Non-Interactive Execution (Cron and Scripts)

On a desktop system, the OS keyring is unlocked at login and any program can read
it silently. On a headless server running scripts via cron or Docker, there is no
unlocked system keyring. Two environment variables must be set explicitly:

```bash
export GOG_KEYRING_BACKEND=file
export GOG_KEYRING_PASSWORD=$(cat "$HOME/.config/gogcli/.gog_pw")
```

Without `GOG_KEYRING_BACKEND=file`, gog falls back to the system keyring, finds
nothing, and reports "No auth." Without `GOG_KEYRING_PASSWORD`, it cannot decrypt
the token files. These two lines should appear at the top of any script that
calls gog outside of an interactive terminal session.

---

## Part 6 — How Credentials Flow Across Execution Contexts

A typical OpenClaw deployment has three contexts that need Google API access:

```
Host machine
├── ~/.config/gogcli/           ← master copy of all credentials and tokens
│
├── Host cron scripts           ← read directly from ~/.config/gogcli
│   Set GOG_KEYRING_BACKEND=file + GOG_KEYRING_PASSWORD
│   Use --client on every gog call
│
├── OpenClaw gateway (Docker)   ← bind mount of ~/.config/gogcli, read-only
│   /host/path/.config/gogcli → /same/path/.config/gogcli:ro
│   Container reads tokens; cannot modify them
│
└── Agent sandboxes (Docker)    ← same bind mount, inherited from gateway config
    Read-only access to tokens
    GOG_KEYRING_BACKEND=file set via wrapper script or environment
```

**There is one master copy of the tokens on the host.** Everything else reads it.

### Why the mount must be `:ro`

The gateway and sandbox containers should mount the `gogcli` directory as
**read-only**. If mounted `:rw`, a container running `gog auth add` would
overwrite the host's token files — potentially with tokens that only cover the
scopes the container requested, silently discarding tokens for other scopes or
accounts. This failure produces no error at the time it happens; it only surfaces
the next time a host script tries to use the deleted token.

---

## Part 7 — Initial Setup Procedure

### Step 1 — Create a Google Cloud project and OAuth client

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create
   a new project
2. Enable the APIs you need (Gmail API, People API, Sheets API, etc.)
3. Go to **APIs & Services → OAuth consent screen**
   - User type: External
   - Fill in app name and support email
   - Add scopes for the APIs you enabled
   - Add the target Google account as a test user
   - Click **Publish App** (removes the 7-day refresh token expiry)
4. Go to **APIs & Services → Credentials → Create Credentials → OAuth client ID**
   - Application type: **Desktop app**
   - Download the JSON, then create a new file with just `client_id` and
     `client_secret` (see format note in Part 3)

### Step 2 — Place credentials on the host

```bash
mkdir -p ~/.config/gogcli
cat > ~/.config/gogcli/credentials.json << 'EOF'
{
  "client_id": "YOUR_CLIENT_ID.apps.googleusercontent.com",
  "client_secret": "YOUR_CLIENT_SECRET"
}
EOF
chmod 600 ~/.config/gogcli/credentials.json
```

### Step 3 — Set up the file keyring backend

```bash
# Choose a strong password
echo "your-strong-password" > ~/.config/gogcli/.gog_pw
chmod 600 ~/.config/gogcli/.gog_pw

# Tell gog to use the file backend
cat > ~/.config/gogcli/config.json << 'EOF'
{
  "keyring_backend": "file"
}
EOF
```

### Step 4 — Authenticate

On a headless server, use `--manual` (single-process, no fragile state files):

```bash
export GOG_KEYRING_BACKEND=file
export GOG_KEYRING_PASSWORD=$(cat ~/.config/gogcli/.gog_pw)

gog auth add user@gmail.com --client default \
  --services gmail,contacts \
  --manual --force-consent
```

`gog` prints a Google authorization URL. Open it in any browser, log in as the
target account, and click Allow. When the browser redirects to a page that fails
to load, copy the full URL from the address bar and paste it into the terminal.

### Step 5 — Verify

```bash
GOG_KEYRING_BACKEND=file \
GOG_KEYRING_PASSWORD=$(cat ~/.config/gogcli/.gog_pw) \
gog auth list
```

You should see the account listed with its authorized scopes.

---

## Part 10 — Gog Skill Auto-Mount Injection and Conflicts with Other Credentials

If you install the gog skill into OpenClaw (from `~/.agents/skills/gog/SKILL.md`),
the skill's `"requires": {"bins": ["gog"]}` declaration causes OpenClaw to
automatically inject gog bind mounts into **every agent sandbox** — including
agents that don't explicitly configure gog mounts in their own `openclaw.json` entry:

```
/tmp/.config/gogcli        ← gog config and keyring (host → container)
/tmp/.local/share/keyrings ← GNOME keyring files
gog binaries
```

**The conflict:** If you also configure directory-level bind mounts for another
tool at paths like `/tmp/.config/gsuite-mcp` or `/tmp/.local/share/gsuite-mcp`,
OpenClaw may override the child mounts when it resolves the parent-directory trees
injected by the gog skill. The gsuite-mcp directories appear to be mounted but
are empty or wrong.

**The fix:** Use file-level bind mounts for other credential files, at paths
outside the `/tmp/.config/` and `/tmp/.local/share/` trees:

```json
"/home/YOUR_USER/.local/share/gsuite-mcp/token.json:/tmp/gsuite-token.json:rw"
"/home/YOUR_USER/.config/gsuite-mcp/credentials.json:/tmp/gsuite-credentials.json:ro"
```

And point scripts to them via environment variables:
```json
"GSUITE_MCP_TOKEN_PATH": "/tmp/gsuite-token.json",
"GSUITE_MCP_CREDENTIALS_PATH": "/tmp/gsuite-credentials.json"
```

File-level mounts are completely independent of directory mounts — they cannot
conflict with anything the gog skill injects.

---

## Part 11 — Sharing Files with Users via Google Drive

Google Drive is the right place for ad-hoc agent output — analysis results, generated reports, data exports — that an agent wants to hand off to a user. It keeps that output out of `/shared/outbox/` (a JSON-only pipeline, not a file drop) and off the email queue (which requires human approval per the outbox pattern).

### Folder ownership and access controls

Create the shared folder using the **agent's Google account** (e.g., `tpc26agent@gmail.com`). Because the agent *owns* the folder, it can write to it using its existing gog credentials — no separate write-access grant is needed.

To allow end users to open files without signing in, set the folder sharing to **"Anyone with the link → Viewer"**.

> **Two independent controls:** Folder ownership grants the agent write access. "Anyone with the link can view" grants end-user read access. These are separate settings — configuring end-user read access does not affect the agent's write access, and vice versa.

To find the folder ID, open the folder in your browser. The ID is the last segment of the URL:

```
https://drive.google.com/drive/folders/1nd7VETj6csfjYjLktv1oelsnGyID8UaV
                                        └──────── folder ID ───────────────┘
```

### Upload command

From inside the agent sandbox (or any context where gog credentials are available):

```bash
gog drive upload /tmp/output_file.md \
  --parent FOLDER_ID \
  --account tpc26agent@gmail.com \
  --client default \
  --json
```

**Always specify `--client default`** — see Part 4. Without it, gog may resolve to a clientless token that fails in the sandbox execution environment.

The `--json` flag returns a JSON object. The relevant fields:

| Field | Content |
|-------|---------|
| `id` | Drive file ID |
| `webViewLink` | URL to share with the user |
| `name` | File name as stored in Drive |

### The `--convert` flag

```bash
gog drive upload /tmp/report.md \
  --parent FOLDER_ID \
  --account tpc26agent@gmail.com \
  --client default \
  --convert --json
```

Without `--convert`: the file is stored as a raw attachment (e.g., a `.md` file that the browser downloads).

With `--convert`: Drive converts the file to a native Google Doc — rendered in the browser, full-text searchable, editable. For reports and summaries the user will read directly, `--convert` is generally preferred. Note that conversion changes the MIME type; the `webViewLink` still works correctly.

### After upload

Standard pattern for an agent completing a Drive upload:

1. Parse `webViewLink` from the JSON response
2. Share the link with the user in the current channel (Slack DM, chat message, etc.)
3. Clean up the local temp file: `rm -f /tmp/output_file.md`

Files uploaded to Drive are persistent — they are not automatically deleted. If the folder accumulates many files over time, a maintenance script can list and prune old files using `gog drive list --parent FOLDER_ID --account tpc26agent@gmail.com --json`.

### Where not to use Drive

| Output type | Correct destination |
|-------------|---------------------|
| Ad-hoc user-facing reports, exports, analysis | Drive folder (this section) |
| Queued email or Slack drafts awaiting approval | `/shared/outbox/` (JSON only) |
| Intermediate files within a single runbook | `/tmp/` — clean up when done |
| Internal reports consumed by other scripts | `shared/reports/` |

Never write `.md`, `.txt`, or other non-JSON files to `/shared/outbox/`. That directory is a JSON pipeline processed by cron — non-JSON files are silently ignored and accumulate as clutter.

---

## Part 8 — Re-authentication

Re-authentication is only needed if:
- You explicitly revoke the app's access in Google account settings
- The token files are deleted from `~/.config/gogcli/keyring/`
- You rotate to a new OAuth client
- **Google revokes the refresh token** — this happens silently and without warning due to security events, password changes, too many concurrent sessions, or extended inactivity. The first symptom is HTTP 400 on every API call; see Part 9.

It is **not** needed on a regular schedule once the app is published.

**If the token file may be corrupted** (e.g., you previously ran `gog login` with the wrong keyring passphrase, leaving a file that is larger than expected — ~4KB vs the normal ~1.8KB for other accounts), delete it before re-authenticating:

```bash
# Check sizes — a healthy token file is ~1800 bytes
ls -la ~/.config/gogcli/keyring/

# Delete only the affected account's token
rm ~/.config/gogcli/keyring/token:personal:user@gmail.com
# or for the default client:
rm ~/.config/gogcli/keyring/token:default:user@gmail.com
```

The keyring passphrase is stored in `~/.config/gogcli/.gog_pw`. Scripts read it automatically via `$(cat ~/.config/gogcli/.gog_pw)`; interactive `gog login` prompts for it. Use whatever is in that file — do not guess or use a different password, as entering the wrong passphrase will create a new corrupted token file.

The simplest approach for a headless server is a wrapper script that can be
run from a remote machine:

```bash
#!/bin/bash
# OAuth-renew.sh
# Usage from a remote Mac: ssh -t your-server 'bash ~/path/to/OAuth-renew.sh'
# The -t flag allocates a pseudo-terminal so the terminal can accept your paste.
set -euo pipefail

export GOG_KEYRING_BACKEND=file
export GOG_KEYRING_PASSWORD="$(cat ~/.config/gogcli/.gog_pw)"

echo "After the URL appears:"
echo "  1. Open it in your browser and log in to the target account."
echo "  2. When the page fails to load, copy the full URL and paste it here."
echo ""

gog auth add user@gmail.com --client default \
  --services gmail,contacts \
  --manual --force-consent

echo ""
GOG_KEYRING_BACKEND=file \
GOG_KEYRING_PASSWORD="$(cat ~/.config/gogcli/.gog_pw)" \
gog auth list
```

### Re-authenticating the gmail_api.py token (separate from gog)

If your agent uses a Python script (`gmail_api.py`, `contacts_api.py`) that reads
its own token file at `~/.local/share/gsuite-mcp/token.json`, this is entirely
separate from gog's keyring. Symptoms of a revoked token here are HTTP 400 errors
from the agent's heartbeat, while gog-based tests (e.g., `gog gmail list`) still pass.

Do **not** use `gsuite-mcp setup` for re-auth — that binary is only needed for the
initial setup. A standalone script handles all future re-authorizations with no
external tools beyond Python stdlib:

```bash
python3 scripts/reauth-cecat.py
```

The script reads `~/.config/gsuite-mcp/credentials.json`, prints an authorization
URL, and waits for you to paste the redirect URL (your browser will show
`ERR_CONNECTION_REFUSED` — that is expected; the auth code is in the URL).
It writes a fresh `token.json` including the `expiry` field required by `gmail_api.py`.

After re-auth, verify with the agent's smoke test:

```bash
bash scripts/run-cecat-tests.sh
```

### Preventing 7-day token expiration: publish your OAuth app

Google refresh tokens issued to apps in **Testing** mode expire after 7 days,
regardless of usage. This causes silent recurring failures that look like
intermittent network errors.

**Fix — do this once per Google Cloud project:**

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Select the project (e.g., `cec-personal` for the cecat account)
3. **APIs & Services → OAuth consent screen**
4. Click **Publish App**
5. Acknowledge the verification warning (personal-use apps do not need Google
   verification; you will see an "unverified app" warning when authorizing but
   it is harmless for apps you control)

Once published, refresh tokens persist until explicitly revoked. You will only
need to re-authorize if the account password changes, you revoke access manually,
or Google detects a security event on the account.

If you have multiple Google Cloud projects (one per agent account), check and
publish each one. A project that was already used in production (not just
testing) may already be published.

---

## Part 9 — Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| "No auth for gmail user@..." | `GOG_KEYRING_BACKEND` not set, or wrong token file resolved | Add `--client default`; set `GOG_KEYRING_BACKEND=file` |
| "No auth" from interactive terminal but works over SSH | Desktop session bus present; gog picks up clientless token | Delete `keyring/token:user@gmail.com` (clientless); always use `--client` |
| gog auth add uses the wrong client_id | `credentials.json` in wrapped Google Console format | Rewrite to flat `{"client_id": ..., "client_secret": ...}` format |
| "manual auth state mismatch" | Stale state file from a previous partial auth attempt | Delete `~/.config/gogcli/oauth-manual-state-*.json` and retry |
| Tokens expire after 7 days | OAuth app left in Testing mode | Publish the app in Google Cloud Console OAuth consent screen |
| Host tokens overwritten after a container runs | `gogcli` directory mounted `:rw` in Docker | Change mount to `:ro` |
| `gog auth add` succeeds but subsequent calls fail | Clientless token file created alongside the client-qualified one | Delete `keyring/token:user@gmail.com`; use `--client` on all calls |
| `aes.KeyUnwrap(): integrity check failed` on every call | Token file encrypted with a different password than current `.gog_pw` | Restore token file from backup; or re-run `gog auth add` after confirming `.gog_pw` is correct. Do NOT run `gog auth add --force-consent` to fix this — it will write a new file with the current password but only the scopes you request, silently discarding other scopes. |
| HTTP 400 on every API call, including token refresh | Google revoked the refresh token (security event, password change, too many sessions, or extended inactivity). This is a Google-side revocation — the keyring file is intact but the stored token is no longer honored. | Check token file size: if larger than ~2KB, delete it first (prior failed re-auth attempts corrupt the file by appending). Then re-run `gog login <account> --manual --force-consent` and enter the passphrase from `.gog_pw`. |
| "Access blocked: [App] has not completed Google verification" (403) | OAuth app uses sensitive scopes (gmail) and is not verified; common after running `gog auth add` against wrong Google Cloud project | Confirm you are in the correct project (check client_id in `gog auth list` vs your `credentials.json`). For personal-use apps, unpublish and re-add your account as a test user, then retry. |
