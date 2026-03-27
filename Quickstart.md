# OpenClaw Quickstart

Get OpenClaw running on a Linux machine in about 15 minutes, then add integrations.

**Assumes:** Docker Engine 24+, Docker Compose v2, and basic Linux sysadmin comfort.

---

## Prerequisites

**Tailscale** — The gateway binds only to your Tailscale IP, not `0.0.0.0`. This
keeps the dashboard off your LAN and off the internet. If you haven't used Tailscale
before, install it first: https://tailscale.com/download

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

The first launch runs a setup wizard that creates `openclaw.json` inside the
`openclaw-config` Docker volume. Complete it before proceeding.

---

## Step 5 — Configure agents and apply settings

Edit `config.yaml` to set your agent name and model. The default is one agent
(`main`) using `anthropic/claude-sonnet-4-6`. Adjust to taste.

Apply the configuration (patches the live `openclaw.json` and restarts the gateway):

```bash
pip install --break-system-packages pyyaml    # one-time
python3 apply-config.sh
```

If the gateway crash-loops after applying config, the script detects it and prints
the error. Fix the config and re-run.

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
| **Slack** | Agents post and receive messages via Slack | [Slack-Integration.md](Slack-Integration.md) |
| **Gmail** | Personal email assistant agent | [Google-Integration.md](Google-Integration.md) + [GOG-Integration.md](GOG-Integration.md) |
| **Google Drive / Sheets / Contacts** | Agents read and write Google Workspace | [GOG-Integration.md](GOG-Integration.md) |

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
