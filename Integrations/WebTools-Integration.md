# Web Tools Integration — Step-by-Step Setup Guide

This guide covers enabling `web_search`, `web_fetch`, and the built-in browser
tool for OpenClaw agents. It is referenced from Section I.4 of the main tutorial.

Web tools are native OpenClaw capabilities — no MCP servers or third-party
dependencies are required beyond a Brave Search API key (free tier).

| Tool | Use for | Credential required |
|---|---|---|
| `web_search` | Discovery, search results, finding URLs | Brave Search API key |
| `web_fetch` | Read a known static URL cheaply; no JS | None |
| browser | JS-heavy pages, login-required pages, Google Form filling | None (gateway-managed) |

---

## Prerequisites

- OpenClaw gateway running and accessible via Tailscale (Module 3 complete)
- `apply-config.sh` installed and working (at least one successful apply run)
- `secrets.yaml` accessible on the host (alongside `config.yaml`)
- For browser tool: tpc26agent@gmail.com credentials, if agents need to fill
  Google Forms or access login-required pages as that account

---

## Phase 1 — Web Search and Fetch

### Part 1 — Get a Brave Search API Key

1. Go to [api.search.brave.com](https://api.search.brave.com) and sign up
2. Create a subscription — the **Free AI tier** provides 2,000 queries/month
   at no cost; sufficient for most agent workloads
3. Generate an API key — it starts with `BSA...`

### Part 2 — Add the Key to secrets.yaml

`secrets.yaml` lives alongside `config.yaml` in your spark-ai repo. It is never
committed to version control. Add one line:

```yaml
brave_search_api_key: BSA...your-key-here...
```

`apply-config.sh` reads this key and writes it into `openclaw.json` at apply time.
The key is validated before the gateway restarts — if it is missing or set to
`REPLACE_ME`, `apply-config.sh` exits with an error before touching the gateway.

### Part 3 — Enable Web Tools in config.yaml

Add a top-level `tools:` block to `config.yaml`. This enables `web_search` and
`web_fetch` globally — for all agents by default:

```yaml
tools:
  web:
    search:
      enabled: true
      provider: brave
    fetch:
      enabled: true
```

**Restricting access to specific agents:** If some agents should not have web
access, add a `tools.deny` list to their entry in the `agents:` block:

```yaml
agents:
  restricted-agent:
    model: argo/argo:claude-4.6-sonnet
    tools:
      deny:
        - web_search
        - web_fetch
```

`apply-config.sh` writes the deny list into that agent's entry in `openclaw.json`.
The global `tools.web` block is still set — the deny list overrides it per-agent.

> **Security note:** An empty deny list (`deny: []`) is not the same as omitting
> the `tools.deny` key. An empty list tells `apply-config.sh` to explicitly set
> `"deny": []` in `openclaw.json`, which may override any restrictions previously
> set there manually. If you intend no restrictions, omit the `tools:` key from the
> agent block entirely.

### Part 4 — Apply and Verify

```bash
python3 apply-config.sh
```

Output should include:
```
Configuring web tools...
  web_search: enabled (provider: brave)
  web_fetch: enabled
```

After the gateway restarts cleanly, verify in the OpenClaw dashboard: open an
agent session and confirm `web_search` and `web_fetch` appear in the available
tools list. Do a quick smoke test:

```
Search for recent news about OpenClaw and tell me what you find.
```

The agent should call `web_search`, return results, and optionally call
`web_fetch` on one of the result URLs.

---

## Phase 2 — Browser Tool

The browser tool gives agents access to a full Chromium browser for JS-heavy
pages, login-required pages, and Google Form filling. It runs at the **gateway
level** — not inside the agent sandbox — so it does not interact with
iptables or the sandbox network policy.

### Part 5 — Security Prerequisite: SSRF Protection

Before enabling the browser, verify that `ssrfPolicy.allowPrivateNetwork` is set
to `false` in your browser config. This prevents the browser tool from being used
to reach internal services on the host or Docker network (an SSRF attack vector).

The configuration below includes this setting. Do not enable the browser without it.

### Part 6 — Enable Browser in config.yaml

Add a top-level `browser:` block to `config.yaml`:

```yaml
browser:
  enabled: true
  ssrfPolicy:
    allowPrivateNetwork: false
```

**Per-agent browser restriction:** If some agents should not have browser access,
add a `sandbox.browser` block to their agent entry:

```yaml
agents:
  restricted-agent:
    model: argo/argo:claude-4.6-sonnet
    tools:
      deny:
        - web_search
        - web_fetch
    sandbox:
      browser:
        enabled: false
```

Apply:

```bash
python3 apply-config.sh
```

Output should include:
```
Configuring browser tool...
  browser: enabled  ssrfPolicy.allowPrivateNetwork = false
```

> **Named browser profiles:** OpenClaw's browser profile schema requires connecting
> to an already-running Chrome instance via CDP (`cdpPort` or `cdpUrl`). Named
> profiles with `userDataDir` are only supported with `driver: "existing-session"`.
> For the typical case — OpenClaw managing its own browser instance — do not specify
> a `profiles:` block. OpenClaw maintains session state internally.

### Part 7 — Authenticate Google (Manual Step)

If agents need to fill Google Forms or access Google-authenticated pages as
`tpc26agent@gmail.com`, you must complete a one-time manual authentication in the
OpenClaw dashboard. OpenClaw persists the session in its internal browser state.

1. Open the OpenClaw dashboard
2. Start a conversation with an agent that has browser access (e.g., luoji or cecat)
3. Ask the agent: `browser open https://accounts.google.com`
4. In the browser pane in the dashboard, complete the sign-in as `tpc26agent@gmail.com`
5. Verify you are signed in — ask the agent to snapshot the page

The session persists across gateway restarts until you clear browser state manually
or the browser profile is reset.

> **Account selection:** Use a dedicated non-personal Google account
> (`tpc26agent@gmail.com` or similar) for all agent browser sessions. This limits
> the blast radius if the session is compromised and keeps agent activity separate
> from personal accounts. Never use your own personal Google account.

### Part 8 — Google Form Filling (Two-Step Pattern)

For tasks that require filling and submitting Google Forms, agents follow a
mandatory two-step pattern that prevents unintended submissions:

**Step 1 — Fill and stop:**

1. `web_fetch <url>` — read form instructions first (fast, no JS required)
2. `browser open <url>` — open the form in the browser
3. `browser fill <fields>` — fill each field per task data
4. `browser snapshot` — take a screenshot
5. **Stop** — do not click Submit
6. Post the snapshot to Charlie via Slack for review

**Step 2 — Submit only on explicit confirmation:**

7. Wait for a `CONFIRM_SUBMIT | <form_url>` task from the human operator
8. Only after receiving that task: click Submit
9. Take a post-submit snapshot and confirm the confirmation page loaded

Each agent workspace contains `runbooks/RUNBOOK_FILL_FORM.md` with the exact
procedure. The rule is enforced by the runbook, not by the tool — there is no
technical mechanism that prevents submission. Agents must be instructed to follow
it and must have the runbook available.

---

## Tool Decision Reference

| Need | Tool |
|---|---|
| Find a page or search for information | `web_search` |
| Read a known static URL cheaply | `web_fetch` |
| JS-heavy page | browser |
| Login-required page | browser (after manual auth, Part 7) |
| Google Form filling | browser + RUNBOOK_FILL_FORM.md |

**When not to use `web_fetch` instead of the browser:** `web_fetch` does not
execute JavaScript. Many modern pages render their content entirely via JS — the
fetch returns the raw HTML skeleton with no visible content. If `web_fetch`
returns an empty or skeletal page, use the browser instead.

**Upgrade path:** If Brave search results are insufficient for a specific task,
consider Exa (neural search + content extraction in one call). Do not add Firecrawl
(routes content through third-party cloud) or Playwright MCP (registry install, more
attack surface than the built-in browser). Add third-party search tools only when
a specific real task demonstrably fails with the built-in tools.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `web_search` returns no results | Brave API key missing or invalid | Check `secrets.yaml`; verify key at api.search.brave.com |
| Agent says `web_search` is not available | Tool in agent's deny list, or global tools block missing | Check `openclaw.json` `tools.web.search` and agent's `tools.deny` |
| Browser config causes gateway crash loop | Invalid profile config | Do not specify `profiles:` block unless using existing-session CDP driver |
| Browser not authenticated as Google account | Manual auth not completed | Follow Part 7 |
| Agent submits a form without confirmation | Runbook not present or not followed | Verify `runbooks/RUNBOOK_FILL_FORM.md` exists and HEARTBEAT.md references it |
| `web_fetch` returns empty or skeletal content | Page is JS-rendered | Use browser instead |
| `apply-config.sh` exits with `brave_search_api_key not set` | Key not in `secrets.yaml` | Add key; see Part 2 |
