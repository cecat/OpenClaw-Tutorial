# TUT_TODO.md — Tutorial Work Items

Tasks remaining to complete this tutorial repository before the hands-on lab
can be built. Items are ordered roughly by priority.

---

## Pending

**Complete the integration guide stubs**

Two stub documents still need full step-by-step walkthroughs with commands,
screenshots, and verification steps:

- `Integrations/Slack-Integration.md` — app creation, scopes, tokens, Socket Mode, channel
  routing, CHANNELS.md, testing
- `Integrations/Google-Integration.md` — Google Cloud project, API enablement, OAuth consent
  screen, gsuite-mcp auth, bind-mount config, verification

These will be the basis for the hands-on lab steps.

---

**Build the hands-on lab document (HANDS-ON.md)**

The 90-minute lecture outline is complete. The companion hands-on lab should:
- Reference the integration guides above for prerequisites
- Walk through standing up a three-agent deployment (main, admin-agent, gmail-agent)
  using the gateway/ and agents/ directories in this repo
- Include exercises: adding a CALENDAR.md entry, observing check-todos.sh,
  the behavioral-change lock-in procedure, an outbox approval cycle end-to-end

---

**Populate agents/ directory**

The `gateway/` directory is now fully populated (docker-compose.yml, docker-compose.local.yml,
.env.example, config.yaml, secrets.yaml.example, apply-config.sh) and matches the Quickstart.
Remaining: decide whether to add an `agents/` skeleton directory with starter HEARTBEAT.md,
SOUL.md, IDENTITY.md templates for the hands-on lab (vs. pointing students to OpenClaw-Gmail/agent/).

---

## Completed

- Tutorial outline (OpenClaw-Tutorial-Outline.md) — 9 modules, rev 7
- timing.md — module timing table + deployment timing guidelines
- Integrations/Slack-Integration.md stub created
- Integrations/Google-Integration.md stub created
- Integrations/GOG-Integration.md stub created
- Integrations/GOG-Integration.md fully expanded — Parts 1–11: installation, auth, keyring
  passphrase setup, sandbox bind-mounts, example scripts, verification steps,
  headless re-auth procedure (reauth-cecat.py), OAuth Testing vs Published
  explanation (2026-03-25)
- Quickstart.md created — 7-step guide for sophisticated Docker users; Tailscale
  prerequisite, workspace, .env/.secrets, compose up, dashboard, apply-config.sh,
  first agent workspace; What's next integration table; local LLM section (2026-03-27)
- gateway/ directory fully populated — docker-compose.yml (Anthropic-only, parameterized),
  docker-compose.local.yml (vLLM override), .env.example, config.yaml, secrets.yaml.example,
  apply-config.sh (2026-03-27)
- OpenClaw-Gmail/agent/HEARTBEAT.md TODO lifecycle fixed — replaced old "append to
  todos.log / remove line" pattern with correct COMPLETED-replacement lifecycle (2026-03-27)
- OpenClaw-Gmail/agent/runbooks/RUNBOOK_INBOX_ANALYSIS.md TODO format fixed — replaced
  wrong `TODO [YYYY-MM-DDTHH:MM]` format with correct `YYYY-MM-DDTHH:MM:SSZ | task` (2026-03-27)
- OpenClaw-Gmail/agent/runbooks/RUNBOOK_TODO.md created — explains TODO lifecycle,
  correct format, and append command (2026-03-27)

---

## Out of Scope Here (System Work — Claude Code in spark-ai-agents)

The following improvements belong in the spark-ai-agents repository, not here.
They are tracked in ENHANCEMENTS.md in this repo for visibility but will be
implemented via Claude Code in /users/catlett/code/spark-ai-agents
and /users/catlett/code/spark-ai.

**Completed in spark-ai-agents:**
- Completed-line deletion: now handled by check-todos.sh (not LLM)
- Session reset script: implemented as reset-sessions.sh with crontab entry

**Still open:**
- Outbox file ownership: reviewing agent as sole manipulator of outbox files
- CHANNELS.md to config.yaml: manage channel routing via apply-config.sh
