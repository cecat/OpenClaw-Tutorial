# OpenClaw Quickstart

Get OpenClaw running on a Linux machine in about 15 minutes, then add integrations. This
has been tested on a DGX Spark (Ubuntu) but is still a WIP.  For more about prerequisites
such as Tailscale, specifics about the docker-compose.yml, or additional scaffolding
to make claws more reliable see the
[full tutorial](https://github.com/cecat/OpenClaw-Tutorial/blob/main/OpenClaw-Tutorial.md)

**Assumes:** Docker Engine 24+, Docker Compose v2, Tailscale, and basic Linux sysadmin comfort.

---

## Prerequisites

**Tailscale** creates a private mesh network (a "tailnet") where only devices authenticated to your Tailscale account can communicate — by binding the gateway to your Tailscale IP rather than 0.0.0.0, access to the dashboard is restricted to your own devices and kept off your LAN and the open internet entirely.

If you haven't used Tailscale before, install it first: https://tailscale.com/download


```bash
# Verify Tailscale is running and get your IP:
tailscale ip -4
```

**An API key** — This quickstart uses the Anthropic cloud API. Get a key at
https://console.anthropic.com. (If you have a GPU server running vLLM, see
[Local LLM](#local-llm-gpu-server) at the bottom.)

---

## Step 1 — Create your workspace

The workspace is the only directory agents can read and write. Create it anywhere:

```bash
mkdir -p ~/openclaw-workspace
```

You'll add agent subdirectories here later (Step 5).

---

## Step 2 — Configure the gateway

```bash
cd ~/Documents/OpenClaw-Tutorial/gateway    # or wherever you've put this repo

cp .env.example .env
```

Edit `.env`:

```bash
TAILSCALE_IP=<output of: tailscale ip -4>
OPENCLAW_WORKSPACE=/home/youruser/openclaw-workspace
USER_HOME=/home/youruser
DOCKER_GID=<output of: getent group docker | cut -d: -f3>
```

```bash
cp secrets.yaml.example secrets.yaml
```

Edit `secrets.yaml` — replace `REPLACE_ME` with your Anthropic API key (`sk-ant-...`).

---

## Step 3 — Start the gateway

```bash
docker compose up -d
docker logs openclaw-gateway --tail 30
```

Look for: `OpenClaw listening on port 18789`. Give it 10–15 seconds on first pull.

---

## Step 4 — Open the dashboard

From any device on your Tailscale network:

```
http://<your-tailscale-ip>:18789
```

The first launch runs OpenClaw's built-in setup wizard, which creates `openclaw.json` inside the `openclaw-config` Docker volume. The wizard takes about 5 minutes — it will ask you to choose a model provider, enter an API key, and optionally configure a messaging channel (Slack, Telegram, etc.). Have the following ready:

| Item | Where to get it | Notes |
|------|----------------|-------|
| Anthropic API key (`sk-ant-...`) | console.anthropic.com | Or key for whichever provider you choose |
| Slack bot token (`xoxb-...`) and app token (`xapp-...`) | api.slack.com/apps | Optional — can skip and add later |

Skip the Slack step if you don't have tokens yet — it's easier to get the gateway running first and add Slack as a second step (see [Integrations/Slack-Integration.md](Integrations/Slack-Integration.md)). If the wizard also asks about a web search provider, skip it — that is managed via `config.yaml` and `apply-config.sh` in Step 5.

OpenClaw's dashboard allows you to edit `openclaw.json` if you enjoy working with huge json files in a small window.  We've created a more convenient scheme that allows you to specify (and eassily update) several config settings in
`config.yaml`.  Afte updating `config.yaml` you run the script  `apply-config.sh` to update openclaw.json in the running gateway. 

---

## Step 5 — Configure agents and apply settings

Edit `config.yaml` to set your agent name, model, and optional fallback model.
The default is one agent (`main`) using `anthropic/claude-sonnet-4-6`. Adjust to taste.

```yaml
defaults:
  fallback_model: vllm/Qwen/Qwen3-Coder-Next-FP8  # retried if primary is unreachable

agents:
  main:
    model: anthropic/claude-sonnet-4-6
```

`fallback_model` is optional but recommended: if the primary model is unreachable (tunnel down, provider outage, rate limit), OpenClaw automatically retries with this model. Set it to your local vLLM model so agents keep working without internet access.

Apply the configuration (patches the live `openclaw.json` and restarts the gateway):

```bash
pip install --break-system-packages pyyaml    # one-time
python3 apply-config.sh
```

If the gateway crash-loops after applying config, the script detects it and prints
the error. Fix the config and re-run.

> **Important:** OpenClaw's live configuration lives inside the `openclaw-config`
> Docker volume, not in the repo files you cloned. `apply-config.sh` is the only
> reliable way to update a running gateway — editing repo files alone has no effect.

After applying, confirm the gateway picked up the new settings:

```bash
curl -s http://<your-tailscale-ip>:18789/health
docker logs openclaw-gateway --tail 20
```

Look for your model name in the startup log lines. If the old model still appears,
the config patch did not take effect — re-run `apply-config.sh` and check for errors.

---

## Step 6 — Create your first agent workspace

```bash
mkdir -p ~/openclaw-workspace/main
```

At minimum, each agent needs three files in its workspace directory:

| File | Purpose |
|------|---------|
| `HEARTBEAT.md` | Instructions the agent follows every 15 minutes |
| `SOUL.md` | Behavioral invariants — what the agent always and never does |
| `IDENTITY.md` | Name, personality, and channel behavior |

See `OpenClaw-Gmail/agent/` in this repo for a working example, or the full
tutorial (Module 3) for a detailed walkthrough of each file.

---

## Step 7 — Verify the agent is heartbeating

In the dashboard, open the agent session. Within 15 minutes you should see the
agent respond `HEARTBEAT_OK`. If it doesn't, check:

```bash
docker logs openclaw-gateway --tail 50
```

---

## What's next

| Add-on | What it enables | Guide |
|--------|----------------|-------|
| **Slack** | Agents post and receive messages via Slack | [Slack-Integration.md](Integrations/Slack-Integration.md) |
| **Gmail** | Personal email assistant agent | [Google-Integration.md](Integrations/Google-Integration.md) + [GOG-Integration.md](Integrations/GOG-Integration.md) |
| **Google Drive / Sheets / Contacts** | Agents read and write Google Workspace | [GOG-Integration.md](Integrations/GOG-Integration.md) |
| **Web search / fetch / browser** | Agents search the web, read URLs, and fill forms | [WebTools-Integration.md](Integrations/WebTools-Integration.md) |
| **MCP servers** | Connect external tools and data sources via HTTP | [MCP-Integration.md](Integrations/MCP-Integration.md) |

Each guide tells you which credential bind mounts to uncomment in
`gateway/docker-compose.yml` and what to add to `config.yaml`.

---

## Scheduling (optional but recommended)

Out of the box, OpenClaw's only proactive trigger is the heartbeat — it fires
every 15 minutes regardless of whether there's work to do. For recurring scheduled
tasks (daily reports, timed reminders, calendar-driven actions), add the scheduling
layer: a bash script (`check-todos.sh`), a `CALENDAR.md` per agent, and a crontab
entry. This is covered in Enhancement 3 of the full tutorial.

---

## Local LLM (GPU server)

If you're running vLLM locally (e.g., on an NVIDIA DGX Spark), use the override
compose file to connect the gateway to your inference server:

```bash
docker compose -f docker-compose.yml -f docker-compose.local.yml up -d
```

Set `model: vllm/<your-model-id>` in `config.yaml` and re-run `apply-config.sh`.
The `VLLM_NETWORK` variable in `.env` must match the Docker network created by
your vLLM docker-compose (`<compose-dir>_nim_net`).

**Finding the right model identifier.** The model string OpenClaw expects may not
match what you see in the vLLM registry UI. The registry may show a display name like
`argo:gpt-5.4` while the `internal_name` used by the API is `gpt54` — and OpenClaw
needs the latter, formatted as `argo/gpt54`. To find the correct string, query the
vLLM models endpoint directly from inside the gateway container:

```bash
docker compose --profile cli run --rm openclaw-cli \
  curl -s http://nim:8000/v1/models | python3 -m json.tool
```

Look for the `id` field in each model object — that is the value to use after `vllm/`
in `config.yaml`.

**Verify the model works before wiring it up.** The docs can't establish that a
given model is reachable and functional on your specific machine — only a real
inference call can. Before applying the config, confirm the endpoint responds:

```bash
docker compose --profile cli run --rm openclaw-cli \
  curl -s http://nim:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "<your-model-id>", "messages": [{"role":"user","content":"hi"}], "max_tokens": 10}'
```

A valid JSON response with a `choices` array means the model is reachable. An
error or timeout here means the problem is in your vLLM setup, not OpenClaw — fix
it before proceeding.

---

## Useful commands

```bash
# Restart the gateway
docker compose restart openclaw-gateway

# View logs
docker logs openclaw-gateway -f

# Open a CLI session (for running openclaw commands directly)
docker compose --profile cli run --rm openclaw-cli

# Stop everything
docker compose down

# Stop and delete all state (destructive — removes openclaw.json and all memory)
docker compose down -v
```

## Re-authenticating Google accounts

Google OAuth refresh tokens can be revoked in several situations: if you change your
Google account password, if the OAuth app is still in "Testing" mode (tokens expire
after 7 days), or if Google's security systems flag a suspicious event. Symptoms are
`HTTP 400 invalid_grant` or `HTTP 401 Unauthorized` errors in agent logs.

Each Google account that your agents use has its own renewal script in `ops/`:

| Account | Used by | Renewal command |
|---------|---------|-----------------|
| `tpc26agent@gmail.com` | chattpc26 scripts (gog CLI) | `bash ops/OAuth-renew.sh` |
| `cecatlett@gmail.com` | cecat scripts (gmail_api.py + gog CLI) | `bash ops/cecat-oauth-renew.sh` |

Run from your server terminal (the `-t` flag is required when running via ssh, to accept
the pasted redirect URL):

```bash
# For tpc26agent@gmail.com
ssh -t spark-ts 'bash ~/code/spark-ai-agents/ops/OAuth-renew.sh'

# For cecatlett@gmail.com (renews both token.json and gog personal client)
ssh -t spark-ts 'bash ~/code/spark-ai-agents/ops/cecat-oauth-renew.sh'
```

Each script prints a Google authorization URL. Open it in a browser, sign in, approve
the permissions, then copy the full redirect URL from the address bar (even though the
page says "can't connect") and paste it back into the terminal.

> **Note:** `cecat-oauth-renew.sh` performs two separate browser authorizations —
> once for `token.json` (used by Python scripts) and once for the gog keyring token
> (used by gog CLI commands). Both use the same Google account but different OAuth
> app registrations, so two consent flows are required.
