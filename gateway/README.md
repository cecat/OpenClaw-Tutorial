# gateway/

This directory contains everything needed to run the OpenClaw gateway — the
Docker-based process that hosts agents, handles Slack integration, routes messages,
and manages sandboxed tool execution.

For first-time setup, follow [Quickstart.md](../Quickstart.md). For a deeper
explanation of how these pieces fit together, see the
[full tutorial](../OpenClaw-Tutorial.md).

---

## Files at a glance

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Main gateway service definition (cloud API edition) |
| `docker-compose.local.yml` | Override that connects the gateway to a local vLLM inference server |
| `.env.example` | Template for `.env` — copy and fill in before first launch |
| `secrets.yaml.example` | Template for `secrets.yaml` — copy and add your API key(s) |
| `config.yaml` | Agent model assignments, Slack channel bindings, web tools, and browser config |
| `apply-config.sh` | Script that patches the live `openclaw.json` and restarts the gateway |

---

## First-time setup (short version)

```bash
cp .env.example .env          # fill in TAILSCALE_IP, OPENCLAW_WORKSPACE, USER_HOME, DOCKER_GID
cp secrets.yaml.example secrets.yaml   # add your Anthropic (or other) API key

docker compose up -d          # pull and start the gateway
pip install --break-system-packages pyyaml   # one-time dependency for apply-config.sh
python3 apply-config.sh       # apply config.yaml → restarts gateway
```

Open the dashboard at `http://<your-tailscale-ip>:18789` and complete the setup wizard.

Full walkthrough: [Quickstart.md](../Quickstart.md), Steps 2–5.

---

## The two files you edit regularly

### `config.yaml`

This is your primary control surface. Edit it to:
- Change which model each agent uses
- Add a fallback model (used automatically if the primary is unreachable)
- Enable or disable web tools (`web_search`, `web_fetch`, browser)
- Bind Slack channels to agents
- Define custom model providers (e.g., a local vLLM server, Argonne Argo)

After any edit, apply with:

```bash
python3 apply-config.sh
```

`apply-config.sh` reads `config.yaml` and `secrets.yaml`, patches `openclaw.json`
inside the running Docker volume, and restarts the gateway. It watches for crash
loops and auto-reverts if the config is invalid. **Do not edit `openclaw.json`
directly** — `apply-config.sh` is the only reliable update path.

### `secrets.yaml`

Contains API keys. It is gitignored — never committed. `apply-config.sh` reads
it and injects keys into the Docker volume at apply time; keys are never written
to the host filesystem. Add one line per provider:

```yaml
anthropic_api_key: sk-ant-...
brave_search_api_key: BSA...     # required if web_search is enabled
argo_api_key: your-username      # required if using the Argo custom provider
```

---

## Local vLLM (GPU server)

If you have a vLLM inference server running locally (e.g., on an NVIDIA DGX Spark),
use the override compose file to connect the gateway to it:

```bash
docker compose -f docker-compose.yml -f docker-compose.local.yml up -d
```

Then set `model: vllm/<your-model-id>` in `config.yaml` and run `apply-config.sh`.
The network name in `docker-compose.local.yml` must match the Docker network created
by your vLLM compose (`<compose-dir>_nim_net`); override with the `VLLM_NETWORK`
variable in `.env` if needed.

See [Quickstart.md — Local LLM](../Quickstart.md#local-llm-gpu-server) for details.

---

## Adding integrations

As you add integrations, you will uncomment bind mounts in `docker-compose.yml`
(credential files) and add configuration to `config.yaml`. The integration guides
walk through both:

| Integration | Guide |
|-------------|-------|
| Slack | [Integrations/Slack-Integration.md](../Integrations/Slack-Integration.md) |
| Gmail / Google Contacts | [Integrations/Google-Integration.md](../Integrations/Google-Integration.md) |
| Google Drive / Sheets / Docs | [Integrations/GOG-Integration.md](../Integrations/GOG-Integration.md) |
| Web search / fetch / browser | [Integrations/WebTools-Integration.md](../Integrations/WebTools-Integration.md) |

---

## Security notes

- **Never change the port binding to `0.0.0.0`** — the gateway binds to your
  Tailscale IP only, keeping the dashboard off your LAN and off the internet.
- **Never commit `.env` or `secrets.yaml`** — both are gitignored.
- **Do not override `user:`** in docker-compose.yml — OpenClaw must run as
  non-root (uid 1000).
- Credential bind mounts follow a strict pattern: specific config subdirectory,
  read-only (`:ro`). See `docker-compose.yml` comments and the integration guides
  before adding new mounts.
