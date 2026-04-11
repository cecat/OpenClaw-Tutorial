# Web Integration — web_search, web_fetch, and browser

OpenClaw provides three built-in web capabilities that cover a spectrum from cheap
read-only lookups to full interactive browser sessions. This guide explains what
each tool is, when to use it, how to enable and configure each one, and the
security model that governs their use.

---

## Overview

| Tool | Transport | JavaScript | Authentication | Cost |
|---|---|---|---|---|
| `web_search` | Brave Search API | No | API key (Brave) | Per-query |
| `web_fetch` | HTTP GET | No | None | Free |
| `browser` | Playwright (headless Chromium) | Yes | Persistent profile | Gateway CPU |

**Rule of thumb:** use the cheapest tool that can do the job. `web_fetch` for static
pages, `web_search` for discovery, `browser` only when JavaScript execution or
authenticated sessions are required.

---

## Part 1 — web_search

`web_search` queries the Brave Search API and returns a ranked list of results with
title, URL, and snippet text. It is the right tool for:
- Finding a page whose URL you do not already know
- Getting a quick summary of current information
- Checking whether something exists (event, paper, person, software project)

It does **not** execute JavaScript, follow login flows, or interact with pages.

### 1.1 Enable web_search

In `config.yaml`:

```yaml
tools:
  web:
    search:
      enabled: true
      provider: brave
```

Then add the Brave API key to `secrets.yaml`:

```yaml
brave_search_api_key: "BSA..."
```

Run `apply-config.sh` to patch `openclaw.json` and restart the gateway.

The free Brave tier provides 2,000 queries per month. Monitor usage at
[api.search.brave.com](https://api.search.brave.com) — the dashboard shows
daily and monthly query counts. For a low-frequency agent deployment (a few
searches per heartbeat, not running 24/7), the free tier is sufficient.

### 1.2 Per-agent access control

`web_search` is enabled globally by default. To deny it to a specific agent,
add it to that agent's `tools.deny` list in `config.yaml`:

```yaml
agents:
  chattpc26:
    tools:
      deny:
        - web_search
        - web_fetch
```

This is the right pattern for agents whose task scope is tightly defined and
should not involve arbitrary web lookups — a conference-logistics agent that
only reads submission spreadsheets does not need internet search.

### 1.3 Using web_search from an agent

The agent invokes the tool using natural language in a tool call:

```
web_search "IETF TPC26 submission deadline 2026"
```

The gateway executes the Brave API call and returns results to the agent. No
shell command, no credentials in the sandbox.

---

## Part 2 — web_fetch

`web_fetch` issues an HTTP GET to a known URL and returns the page content as
plain text (HTML stripped, truncated at approximately 50,000 characters). It is
the right tool for:
- Reading documentation at a known URL
- Checking a status page or API endpoint
- Retrieving structured data from an endpoint that returns JSON or plain text

It does **not** execute JavaScript. Pages that require JavaScript to render
content (React/Vue/Angular SPAs, Google Forms, most modern web apps) will return
an empty body or a loading skeleton — use the `browser` tool for those.

### 2.1 Enable web_fetch

In `config.yaml`:

```yaml
tools:
  web:
    fetch:
      enabled: true
```

No API key required. Run `apply-config.sh` after editing.

### 2.2 Per-agent access control

Same pattern as `web_search` — add `web_fetch` to a specific agent's
`tools.deny` list to restrict it. The two tools are denied independently.

### 2.3 Using web_fetch from an agent

```
web_fetch https://example.com/status
```

---

## Part 3 — browser

The `browser` tool gives agents access to a headless Chromium browser managed
by the OpenClaw gateway. It supports:
- JavaScript execution
- Clicking, typing, form filling
- Screenshots and DOM snapshots
- Login-required pages (via a persistent browser profile)
- Google Forms and other JS-heavy pages

The browser runs **at the gateway level**, not inside the agent sandbox. The
agent issues tool calls (`browser navigate`, `browser fill`, `browser click`,
`browser snapshot`); the gateway executes them in Chromium; the result is
returned to the agent. The sandbox container never touches the network for
browser operations.

### 3.1 Enable the browser tool

In `config.yaml`:

```yaml
browser:
  enabled: true
  ssrfPolicy:
    allowPrivateNetwork: false
```

`allowPrivateNetwork: false` is the required setting for production. It prevents
agents from using the browser to reach internal network services (e.g.,
`http://192.168.x.x`, `http://localhost`). Leave this at `false` unless you
have a specific, audited reason to change it — the risk is an agent being
manipulated into exfiltrating data from internal services it should not reach.

Run `apply-config.sh` after editing.

### 3.2 Per-agent browser access

Browser access is controlled per agent via `sandbox.browser.enabled`:

```yaml
agents:
  chattpc26:
    sandbox:
      browser:
        enabled: false   # deny browser to this agent
  luoji:
    # no sandbox.browser entry → inherits global default (enabled)
```

If the global `browser.enabled` is `true` and an agent has no
`sandbox.browser.enabled: false` entry, that agent has browser access. To
grant browser access to an agent, simply ensure the global setting is `true`
and do not add a `false` override for that agent.

After changing this setting, restart the gateway:

```bash
cd ~/code/spark-ai && docker compose -f openclaw/docker-compose.yml restart openclaw-gateway
```

**An important troubleshooting note:** If an agent reports that the browser
tool is not available, check whether its own documentation (TOOLS.md, runbooks)
describes the tool as "coming soon" or "not yet enabled." These stale notes
can cause the agent to stop trying before attempting the tool call. Verify the
gateway config first, then update the agent's TOOLS.md to reflect actual
availability. A session reset (restart or session wipe) is usually needed for
the agent to pick up the corrected context.

### 3.3 Persistent browser profile and authentication

The gateway maintains a **persistent Playwright browser profile** on the host.
Login sessions, cookies, and local storage persist across browser tool
invocations. This means you authenticate once (manually, via the OpenClaw
dashboard), and subsequent agent sessions inherit the authenticated state.

**Initial authentication:**

1. Open the OpenClaw dashboard
2. Trigger a browser session for the target agent
3. Navigate to `accounts.google.com` (or whichever service the agent needs)
4. Sign in as the intended account (e.g., `tpc26agent@gmail.com`)
5. Complete any 2FA or verification prompts
6. The session is now persisted in the gateway's browser profile

The agent will use this authenticated session in all subsequent browser tool
calls without re-entering credentials. If the session is ever invalidated
(Google timeout, explicit sign-out, profile reset), repeat the authentication
step above.

> **Do not enter credentials in agent-driven browser sessions.** The agent
> should already be authenticated. If an agent reports it is not signed in, STOP
> the task and authenticate manually via the dashboard. Never instruct an agent
> to type a password.

### 3.4 Form submission: the confirm-before-submit pattern

**Any action the browser takes is irreversible from the moment it completes.**
A form submission cannot be un-submitted. An agent operating autonomously
should never submit a form without explicit human confirmation.

The required pattern for any form submission:

1. Agent fills all form fields (`browser fill`)
2. Agent takes a snapshot of the filled form (`browser snapshot`)
3. **Agent stops and posts the snapshot to the operator via Slack**
4. Agent writes a deferred task to `TODO.md` waiting for `CONFIRM_SUBMIT`
5. Operator reviews the snapshot and, if correct, issues `CONFIRM_SUBMIT`
6. Agent, on its next heartbeat, reads the `CONFIRM_SUBMIT` task and clicks Submit

This pattern is documented in the agent-level runbook (`RUNBOOK_FILL_FORM.md`).
Never configure an agent to skip steps 3–5 and submit immediately — the
fill-snapshot-confirm cycle is the security control that prevents incorrect
submissions. There is no undo.

### 3.5 SSRF considerations

The `allowPrivateNetwork: false` policy blocks browser requests to RFC-1918
addresses and localhost. This is important because:

- Prompt injection via a malicious web page could attempt to redirect the
  browser to an internal service
- Without the policy, an agent could be manipulated into fetching internal
  dashboards, credentials endpoints, or admin UIs

Keep `allowPrivateNetwork: false` in production. If you need an agent to reach
an internal web service (e.g., an internal Grafana instance), the right
approach is a dedicated script inside the sandbox with network access, not
relaxing the browser SSRF policy globally.

---

## Part 4 — Which Tool to Use

| Situation | Tool |
|---|---|
| Find a page or check current information | `web_search` |
| Read a known static URL (docs, plain text, JSON endpoint) | `web_fetch` |
| Page requires JavaScript to render | `browser` |
| Page requires login (Google, GitHub, internal SSO) | `browser` |
| Fill and submit a form | `browser` (with confirm-before-submit) |
| Agent scope should be tightly restricted | deny `web_search` and `web_fetch` in config |
| Agent should not reach internal network | keep `allowPrivateNetwork: false` |

---

## Part 5 — Verification

After enabling tools and restarting the gateway, verify from within an agent session:

**web_search:**
```
web_search "OpenClaw documentation"
```
Expect: a list of result titles, URLs, and snippets.

**web_fetch:**
```
web_fetch https://httpbin.org/get
```
Expect: JSON body showing the request details.

**browser:**
```
browser navigate https://example.com
browser snapshot
```
Expect: a rendered screenshot or DOM snapshot of the page returned to the
agent. If the agent reports the tool is unavailable, check:
1. `config.yaml` — `browser.enabled: true` at the top level
2. `config.yaml` — no `sandbox.browser.enabled: false` for this agent
3. Gateway restarted after config change
4. Agent's own TOOLS.md — no stale "not yet enabled" language that would
   prevent the agent from attempting the call

---

## Part 6 — Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Agent reports browser "not available" despite global enable | Agent's session loaded stale TOOLS.md saying "coming soon" | Update TOOLS.md, restart agent session |
| Agent has browser enabled but web_search is not working | `brave_search_api_key` missing from secrets.yaml | Add key and re-run apply-config.sh |
| `web_fetch` returns empty body on a JS-heavy page | Page requires JavaScript | Use `browser navigate` + `browser snapshot` instead |
| Browser session not authenticated after gateway restart | Persistent profile not persisted, or profile path misconfigured | Re-authenticate manually via dashboard |
| Agent submits form without confirmation | `RUNBOOK_FILL_FORM.md` not loaded, or agent ignored it | Confirm runbook is in `runbooks/`, add to HEARTBEAT.md pre-flight checklist |
| SSRF policy blocks an internal URL the agent needs | `allowPrivateNetwork: false` is working correctly | Do not relax the policy; use a sandbox script with direct network access instead |
| Brave API returns 429 (rate limited) | Free tier query limit reached | Check usage at api.search.brave.com; upgrade tier or reduce query frequency |
