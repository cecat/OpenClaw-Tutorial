# Integrations/

Step-by-step setup guides for connecting external services to OpenClaw.
Each guide covers both the service-side configuration and the OpenClaw-side
changes (`docker-compose.yml` bind mounts, `config.yaml` entries, and any
credential setup).

The [full tutorial](../OpenClaw-Tutorial.md) (Section 3: Integrations) gives
an architectural overview of each integration. Come here for the step-by-step.

---

## Guides

| Guide | What it sets up |
|-------|----------------|
| [Slack-Integration.md](Slack-Integration.md) | Connect a Slack workspace to the gateway; route channels to agents; outbound posting via the slack-outbox pattern |
| [Google-Integration.md](Google-Integration.md) | Google OAuth credentials for Gmail and Contacts; bind mounts; token renewal |
| [GOG-Integration.md](GOG-Integration.md) | `gog` CLI for Google Sheets, Drive, Docs, and Gmail sending; OAuth keyring setup; credential flow across sandbox containers |
| [WebTools-Integration.md](WebTools-Integration.md) | `web_search` (Brave), `web_fetch`, and the built-in browser tool; per-agent access control; Google Form filling two-step pattern |
