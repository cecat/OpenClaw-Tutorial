# Getting Started with OpenClaw: A Safety-First Approach
## Tutorial Outline — 90-Minute Lecture

**Target audience:** Developers with working knowledge of GitHub, Python, shell scripting, Docker, and Linux (crontab). No prior OpenClaw experience required.

**Central theme:** This tutorial approaches OpenClaw with deliberate attention to safety, containment, and alignment from the very first step — not as an afterthought.

**Source repositories (reference, not modified):**
- `spark-ai` — infrastructure: Docker compose, scripts, configuration docs
- `spark-ai-agents` — agent workspaces: identity files, runbooks, cron scripts
- `OpenClaw-Gmail` — standalone worked example: Gmail/Contacts agent

---

## Module 1 — What Is OpenClaw and Why This Approach? *(10 min)*

### 1.1 What OpenClaw Is (and Isn't)
- OpenClaw is an **agent framework**, not a model — it gives a language model hands, memory, and a schedule
- Open source, self-hosted; you own the configuration, credentials, and history
- Compatible with local models (via vLLM) and cloud APIs (Anthropic, OpenAI)
- Communicates through Slack, a web dashboard, Telegram, and more
- The model does cognition; OpenClaw handles orchestration, file access, tool execution, and memory

### 1.2 Why "Safety-First"?
- Autonomous agents are qualitatively different from chatbots: they take actions between conversations
- A poorly configured agent can send emails, modify files, and reach network services — autonomously, while you sleep
- The common tutorials skip the threat model and jump to the demo; this one doesn't
- **This tutorial treats your agent the way you'd treat a new employee with access to your files, calendar, and email**: trust is earned, access is earned, autonomy is bounded

### 1.3 The Architecture in One Paragraph
- A local or cloud LLM is served behind an OpenClaw gateway
- The gateway manages agent identities, sessions, tool execution, and channel integrations
- Agent "personality" and memory live in plain markdown files on disk — readable, auditable, version-controlled
- All code execution runs inside ephemeral Docker sandbox containers — not on the host
- Cron scripts on the host handle all scheduling decisions; the LLM handles judgment

### 1.4 What We'll Cover in This Lecture
- **First:** The safety, containment, and alignment design charter — this drives every other decision
- Platform setup: Docker stack, Tailscale, choosing a model with `config.yaml`
- Agent identity: the 8 files that define who your agent is
- Behavioral reliability: when and why agents don't do what you told them
- Scheduling architecture: why the LLM should never decide when to act
- Integrations: Slack, Gmail, runbooks, and the email outbox pattern
- Lessons learned from a working production deployment

*The hands-on lab (a separate session) will put this into practice using OpenClaw-Gmail as the worked example.*

---

## Module 2 — Safety, Containment, and Alignment: The Design Charter *(15 min)*

> **This module is not a safety checklist appended to the end of the tutorial. It is the foundation.**
> Every architecture decision in Modules 3–7 — sandboxing, iptables rules, the outbox pattern, `SOUL.md`, the two-repo split, the scheduling design — exists because of the principles established here. We cover it second, not last, because you should understand *why* before you're asked to execute *what*.

### 2.1 The Threat Model: What Can Go Wrong
Four distinct risk categories when running an autonomous agent:

1. **Prompt injection** — malicious content in files, email, or Slack tricks the agent into executing unintended commands
2. **Lateral movement** — a compromised sandbox container reaches your LAN, other Tailscale nodes, or SSH destinations
3. **Runaway external actions** — the agent sends emails, posts to public channels, or modifies critical files without human review
4. **Credential exfiltration** — secrets (API keys, OAuth tokens) leak via outbound HTTP from within the sandbox

*These aren't hypothetical. We'll walk through each mitigation in Module 3.*

### 2.2 Alignment: What the Agent Should Never Do
The core alignment document is `SOUL.md` — the behavioral invariants loaded into every inference call.

Key rules to establish from day one:
- **Never proceed without explicit acknowledgment.** "DO NOT PROCEED," "WAIT," and "STOP" mean exactly that — not "acknowledge and continue."
- **Never affect the gateway or other agents without explicit permission.** No restarting containers, no modifying gateway config.
- **Never send external communications without human approval** (email, public Slack posts).
- **Never sleep or block** — write deferred tasks to `TODO.md` instead; the heartbeat executes them.
- **The cost of being wrong is far greater than the cost of waiting.** Hardcode this into your agent's soul.

Why `SOUL.md` and not a system note somewhere? Because `SOUL.md` is one of the 8 files automatically loaded into every inference call. Rules that live only in secondary files may be ignored. *(Explained in depth in Module 4.)*

### 2.3 Separation of Powers: Code for Procedure, LLM for Judgment

This is the architectural principle that justifies the entire superstructure of runbooks, scripts, and cron jobs. It is not primarily about cost or efficiency — it is about **reliability, auditability, and testability**.

> **Anything deterministic is owned by code. Anything requiring judgment, creativity, or natural language understanding or generation is owned by the model. The boundary between them must be sharp and explicit.**

The two sides of the boundary:

| Belongs to code | Belongs to the LLM |
|---|---|
| When to act (cron schedule) | Whether this email needs a reply |
| Whether a TODO item is due (bash date comparison) | What the reply should say |
| Fetching email from the Gmail API | Identifying action items in fetched email |
| Routing rows in a spreadsheet by column value | Drafting a summary for ambiguous content |
| Counting, sorting, filtering structured data | Inferring priority from unstructured text |
| Moving a file from outbox to sent | Deciding whether to approve a draft |
| Sending an HTTP request with a known payload | Composing the payload from context |

**Why this matters more than just efficiency:**
- Code is deterministic: the same input always produces the same output. LLM output is probabilistic — shaped by context, session history, and model version.
- Code is auditable: you can read exactly what it does. LLM reasoning is opaque — you can observe inputs and outputs, but not the path between them.
- Code is testable: you can write a unit test. LLM behavior cannot be unit-tested; it can only be observed.
- Code changes are precise: change one line, one behavior changes. LLM instruction changes are suggestions embedded in context — they usually work, until they don't.

**The practical implication — the three-layer superstructure:**

This principle is what generates the entire scaffolding you'll build in this tutorial:

```
┌─────────────────────────────────────────────────────────────┐
│  CRON SCRIPTS  (host — zero tokens, fully deterministic)    │
│  check-todos.sh · reset-sessions.sh · send-approved-emails  │
│  "When to act" — the scheduling engine                      │
├─────────────────────────────────────────────────────────────┤
│  PYTHON/BASH SCRIPTS  (sandbox — deterministic tools)       │
│  gmail_api.py · contacts_api.py · sync-track-sheets.py      │
│  "How to move and transform data" — no LLM involved         │
├─────────────────────────────────────────────────────────────┤
│  RUNBOOKS  (markdown read fresh at trigger time)            │
│  RUNBOOK_EMAIL_DIGEST.md · RUNBOOK_INBOX_ANALYSIS.md        │
│  "Which tools to call and in what order" — procedure        │
├─────────────────────────────────────────────────────────────┤
│  LLM (the OpenClaw agent)                                   │
│  Compose · Summarize · Triage · Decide · Communicate        │
│  "What does this mean and what should we say" — judgment    │
└─────────────────────────────────────────────────────────────┘
```

Each layer calls down only — the LLM never calls the scheduler, scripts never call the LLM, cron never calls anything that requires context. Information flows up; decisions flow down.

This structure also has a safety benefit: **the LLM cannot schedule itself**. It can write a TODO item requesting future work, but the decision of whether and when that item becomes `READY` is made by `check-todos.sh` on the host — outside the agent's reach entirely.

### 2.4 Containment: The Technical Layers
A layered defense — each layer is explained in detail when it appears in setup:

| Layer | Mechanism | Drives which setup decision |
|---|---|---|
| Network | Tailscale-only gateway binding | Module 3: `.env` `TAILSCALE_IP` binding |
| Network | iptables DOCKER-USER rules | Module 3: Step 9, security hardening |
| Execution | Sandbox mode: `mode: all` | Module 3: Step 10, `openclaw.json` |
| Execution | No `docker.sock` in sandbox | Module 3: sandbox config |
| Config | Disable `commands.config` writes | Module 3: Step 10, first hardening step |
| Credentials | Separate dedicated accounts | Module 6: Gmail / Slack credential setup |
| Outbox | Human approval for email | Module 6: outbox pattern |

*When you encounter each of these in Module 3 or 6, the "why" is already established here. We won't repeat it — we'll just execute it.*

### 2.6 The Human-in-the-Loop Pattern
The **outbox pattern** is the cornerstone of supervised external action:

```
Agent writes email JSON → shared/outbox/  (status: "pending")
Supervisor agent or human reviews → sets "approved" or "rejected"
Cron script (send-approved-emails.sh) sends approved emails → moves to sent/
```

- The LLM never directly sends email; it writes to a queue
- A human (or a designated supervisor agent) reviews and approves
- A deterministic cron script sends and archives
- Full audit trail in outbox/sent/rejected directories

This same pattern applies to any external action where errors have real-world consequences.

### 2.7 The Right Mental Model
> Your agent is a diligent assistant, not an autonomous decision-maker. It can do a lot, but it acts in your name. Every email it sends is from you. Every Slack post is from you. Design accordingly.

**Charter summary — five principles that drive every decision in this tutorial:**
- *Separation of powers:* code owns anything deterministic; the LLM owns judgment, creativity, and language — the boundary is sharp and explicit
- *Containment first:* network, execution, and config isolation before connecting anything external
- *Alignment by design:* `SOUL.md` behavioral invariants are non-negotiable, not suggestions
- *Human in the loop:* any external action with real-world consequences goes through an approval queue
- *Verify, don't assume:* the LLM does not send acknowledgment receipts

---

## Module 3 — Platform Architecture and Setup *(20 min)*

### 3.1 Hardware and Model Options
Three viable configurations:

| Configuration | Hardware | Model | Notes |
|---|---|---|---|
| Local GPU | NVIDIA DGX Spark (GB10) | Qwen3-Coder-Next-FP8 via vLLM | ~50 tps generation; ~$4K hardware |
| Cloud API | Any Linux host | Anthropic Claude (claude-sonnet-4-6, etc.) | Pay-per-token; no GPU required |
| Hybrid | Local GPU default + cloud for specific agents | Per-agent model config via config.yaml | Best of both |

*The tutorial uses the DGX Spark as a concrete example; the patterns apply to any configuration.*

### 3.2 The Docker Stack
```
[Your Mac via Tailscale]
    |
    v  port 18789, Tailscale IP only
[OpenClaw container — gateway + CLI]
    |
    v  http://nim:8000/v1, Docker-internal only
[vLLM container — Qwen3-Coder-Next-FP8]
    |
    v
[GB10 GPU + 128GB unified memory]
```

Key design choice: the model API is **never exposed to the host or network** — it is accessible only within the Docker internal network. OpenClaw is exposed **only on the Tailscale interface**.

### 3.3 The Two-Repo Pattern
```
~/code/
├── spark-ai/                    # PUBLIC — infrastructure
│   ├── docker-compose files
│   ├── config.yaml              # per-agent model assignments
│   ├── secrets.yaml             # gitignored — API keys
│   └── openclaw/                # OpenClaw config helpers
└── spark-ai-agents/             # PRIVATE — agent workspaces
    ├── main/                    # Agent workspace (identity, memory, calendar)
    ├── chattpc26/               # Second agent workspace
    ├── scripts/                 # Cron scripts (run on host, not in sandbox)
    └── shared/                  # Runtime state shared between agents
```

Why two repos?
- Infrastructure is public and reusable; agent workspaces are private (contain personal context)
- You can update the platform without touching agent identities, and vice versa
- git history for agent evolution is separate from infrastructure git history

### 3.4 First-Time Setup Walkthrough
*Summary of the 14-step process (full detail in `spark-ai/README.md`):*

**Steps 1–5: Platform**
1. Clone repos (`spark-ai`, `spark-ai-agents`, `spark-vllm-docker`)
2. Build the vLLM Docker image (source build for GB10 kernel support — 20-40 min)
3. Create `.env` files — `TAILSCALE_IP` and `OPENCLAW_WORKSPACE` (host path!)
4. Download the model (~46 GB via HuggingFace, resumable)
5. Fix OpenClaw volume permissions (Docker creates as root; OpenClaw runs as uid 1000)

**Steps 6–8: Gateway**
6. Run OpenClaw onboarding: `docker compose run --rm openclaw-cli onboard --no-install-daemon`
7. Set up Tailscale Serve for HTTPS dashboard access
8. Confirm workspace uses **host paths** (critical gotcha — see sidebar)

> **The host-path gotcha:** When OpenClaw spawns sandbox containers, it passes the workspace path directly to Docker as a bind-mount source. Docker resolves that path on the **host**, not inside the gateway. If you use the container-internal path (`/home/node/agents/main`), Docker creates an empty directory on the host — your agent starts with no identity files.

**Steps 9–10: Security hardening (do this before connecting anything)**
9. Block container lateral movement with iptables DOCKER-USER rules
10. Harden openclaw.json: disable config writes, enable sandbox mode globally

**Steps 11–14: Integrations (optional, covered in Module 6)**
11. Slack
12. Google Workspace (via gog or gsuite-mcp)
13. Agent self-scheduling (TODO.md + check-todos.sh)
14. Session management (daily session reset cron)

### 3.5 Choosing Your Model: config.yaml

`config.yaml` is the single place you control which LLM backs each agent. It lives in `spark-ai/` (infrastructure repo, not agent workspace) and is applied without manually editing `openclaw.json`.

**The student choice — make it early, make it explicit:**

| Path | Requirements | Cost | When to choose |
|---|---|---|---|
| Local GPU (vLLM) | NVIDIA GPU, ~46 GB storage, build time | Hardware amortized | You have a DGX Spark or similar |
| Cloud API (Anthropic Claude) | `ANTHROPIC_API_KEY`, any Linux host | ~$3–10/month | Most students — no GPU needed |
| Hybrid | Both | Both | Run most on local, route specific agents to Claude |

*In this tutorial, when we say "cloud API" we mean Anthropic Claude. The patterns are identical for OpenAI if you prefer.*

**config.yaml structure:**
```yaml
agents:
  main:
    model: anthropic/claude-sonnet-4-6   # cloud path
  chattpc26:
    model: vllm/Qwen/Qwen3-Coder-Next-FP8   # local path
```

Supported model formats: `vllm/<model-id>`, `anthropic/claude-haiku-4-5`, `anthropic/claude-sonnet-4-6`, `anthropic/claude-opus-4-6`.

**secrets.yaml (gitignored — never committed):**
```yaml
anthropic_api_key: sk-ant-...
```
This file lives alongside `config.yaml` in `spark-ai/`. It is excluded from git via `.gitignore`. Copy from `secrets.yaml.example`, paste your key, done. For the local-GPU path it can remain empty.

**Applying a change — the reload script:**
```bash
./apply-config.sh --dry-run   # preview the openclaw.json patch
./apply-config.sh             # apply and restart the gateway
```
`apply-config.sh` patches `openclaw.json` inside the Docker volume directly — no manual JSON editing, no config file archaeology. It reads `config.yaml` and `secrets.yaml` together and generates the right gateway config. The gateway restarts automatically.

**Emergency revert** (if something goes wrong after switching to cloud):
```bash
./revert-to-local.sh
```
Removes all remote-model config from `openclaw.json` and restarts the gateway. Safe to run any time.

**The key point for students:** You do not need to choose at onboarding and live with it. `config.yaml` + `apply-config.sh` let you switch any agent to a different model at any time, in under 30 seconds, without touching `docker-compose.yml`. For the hands-on lab, students using the cloud path can be working alongside students using a local GPU — same agent workspace, different one line in `config.yaml`.

### 3.6 Security Verification Checklist
Run these four checks before considering the platform production-ready:

```bash
# 1. vLLM NOT exposed to host
ss -ltnp | grep :8000          # must show nothing

# 2. OpenClaw bound to Tailscale IP only
ss -tlnp | grep 18789          # must show TAILSCALE_IP:18789, not 0.0.0.0

# 3. No SSH keys in gateway container
docker compose exec openclaw-gateway ls /home/node/.ssh 2>&1   # must say no such file

# 4. Three DOCKER-USER DROP rules in place
sudo iptables -L DOCKER-USER -n | grep DROP   # must show 3 DROP rules
```

---

## Module 4 — Agent Identity and the System Prompt *(15 min)*

### 4.1 The 8 Auto-Loaded Files
On every inference call — every Slack message, every heartbeat — OpenClaw rebuilds the system prompt from exactly these files:

| # | File | Purpose |
|---|---|---|
| 1 | `AGENTS.md` | Sub-agent definitions |
| 2 | `SOUL.md` | Persona, values, hard behavioral invariants |
| 3 | `TOOLS.md` | How to use external tools (gog, scripts, etc.) |
| 4 | `IDENTITY.md` | Who the agent is, Slack behavior contract |
| 5 | `USER.md` | User context, preferences, and background |
| 6 | `HEARTBEAT.md` | Always-on reflexes (runs every heartbeat) |
| 7 | `BOOTSTRAP.md` | First-run orientation guidance |
| 8 | `MEMORY.md` | Accumulated runtime knowledge |

**Everything else is NOT auto-loaded.** `CALENDAR.md`, `EMAIL.md`, runbooks, templates, `PATHS.md` — invisible to the model unless the agent explicitly reads them via a tool call.

Budget: 20,000 characters per file; 150,000 characters total. If a file exceeds this, OpenClaw keeps the first 70% and last 20%.

### 4.2 What Goes Where

| Content type | Right home | Wrong home |
|---|---|---|
| Hard behavioral rules ("never send without approval") | `SOUL.md` or `IDENTITY.md` | `EMAIL.md`, templates |
| Output format specs for Slack | `IDENTITY.md` | template files |
| Step-by-step task procedures | `runbooks/` (read at trigger time) | `HEARTBEAT.md` |
| Recurring schedule | `CALENDAR.md` (read by bash) | `HEARTBEAT.md` |
| Email body templates | `templates/` (agent reads explicitly) | any of the 8 files |

Rule of thumb: **Anything the agent must do correctly without fail belongs in the 8 auto-loaded files.** Anything variable or task-specific belongs in files the agent reads on demand.

### 4.3 Behavioral Reliability: The Three Tiers

| Change location | When agent sees it | Reliability |
|---|---|---|
| One of the 8 system-prompt files | Next inference call | High — but session history can override |
| Non-auto-loaded file (`EMAIL.md`, templates) | Only when agent explicitly re-reads | Indeterminate — may never happen |
| A RUNBOOK | Read fresh from disk on every trigger | **Reliable** — no caching between runs |
| Session history (old behavior examples) | Every call, until session is reset | Works against you |

### 4.4 The Session History Problem
The system prompt is rebuilt fresh on every call — but OpenClaw also sends the **full conversation history** (the accumulated back-and-forth from the session `.jsonl` file). The LLM weighs them together.

When instruction text and behavioral examples conflict, **examples often win.** A long-running session with many examples of old behavior can override a freshly-updated `SOUL.md` instruction.

Consequence: updating a file on disk does not guarantee the agent immediately adopts the new behavior.

### 4.5 How to Lock In a Behavioral Change
Four steps, every time:

1. **Put the rule in the right place** — critical rules go in `SOUL.md` or `IDENTITY.md`, not secondary files
2. **Reset the session immediately** — truncate the `.jsonl` to remove accumulated examples of old behavior:
   ```bash
   docker exec openclaw-gateway truncate -s 0 \
     /home/node/.openclaw/agents/<agent-id>/sessions/main.jsonl
   ```
3. **If the rule is in a non-auto-loaded file, tell the agent** — DM it to re-read the file in the fresh session
4. **Verify** — confirm the next execution follows the new rule; do not assume

### 4.6 The Context Duality (Architectural Implication)
The same mechanism that allows the agent to understand nuance and write in your voice also means you cannot precisely update one rule in isolation. The model integrates everything simultaneously.

This is the behavioral face of the separation of powers principle established in §2.3. An LLM instruction is a suggestion embedded in probabilistic context — it usually holds, until session history, a long conversation, or model drift erodes it. A Python script has no context and no drift: change one line, one behavior changes, on the next execution, every time.

The practical implication is not "LLMs are unreliable" — it's **know which layer owns each responsibility and don't let them bleed into each other**. The agent composes the email; the script sends it. The agent decides a task is done; the cron script archives the record. The agent writes the TODO; bash decides when it fires.

### 4.7 BOOTSTRAP.md: Setting Up the First Agent
- `BOOTSTRAP.md` runs on the first conversation with a fresh agent
- It walks the agent through picking a name, establishing personality, filling in `USER.md`
- Best practice: send this as the very first message — *before* you ask the agent anything else
- After initial setup, `BOOTSTRAP.md` becomes dormant orientation material

---

## Module 5 — Scheduling Architecture *(15 min)*

### 5.1 The Core Principle
> **LLM inference is expensive and non-deterministic. Bash is free and exact.**
>
> The *decision of when to act* should always be made by deterministic code. The *act of doing the work* should be handled by the LLM. These two responsibilities must not be mixed.

### 5.2 The Token-Waste Problem
A 15-minute heartbeat fires **672 times per week**. If an agent checks `HEARTBEAT.md` to decide whether it's time to run a weekly report, it reasons through the question 672 times — including the 671 times the answer is "no":

| Task frequency | Heartbeats/week | Useful fires | Wasted LLM calls | Waste rate |
|---|---|---|---|---|
| Once a week | 672 | 1 | 671 | 99.9% |
| Mon + Thu | 672 | 2 | 670 | 99.7% |
| Once a day (weekdays) | 672 | 5 | 667 | 99.3% |

Beyond token cost: LLM-based scheduling is fragile. The agent must remember whether it already ran the task today — state that doesn't survive a session reset. It will sometimes fire twice, sometimes not at all.

### 5.3 Three Tiers of Scheduling

| Tier | File | Owned by | Lifecycle | Use for |
|---|---|---|---|---|
| Always-on | `HEARTBEAT.md` | Human | Forever, while agent lives | Infrastructure reflexes run every heartbeat |
| Recurring | `CALENDAR.md` | Human (or agent with approval) | Season or finite duty | Day-of-week or daily recurring tasks |
| One-shot | `TODO.md` | Agent (runtime) | Ad hoc | Single deferred tasks written at runtime |

**The human analogy:**
- HEARTBEAT.md = brushing teeth (every day, forever, no thought required)
- CALENDAR.md = "send the Mon/Thu reports through June"
- TODO.md = "call the dentist," "pick up eggs"

### 5.4 The Scheduling Engine: check-todos.sh

```
cron (every 5 min, zero tokens)
  └─ check-todos.sh
       ├─ reads CALENDAR.md → is a recurring entry due? → writes READY to TODO.md
       └─ reads TODO.md     → is a one-shot entry past-due? → marks READY in-place

OpenClaw heartbeat (every 15 min, tokens only when READY work exists)
  └─ agent greps TODO.md for ^READY lines
       ├─ none found → HEARTBEAT_OK  (cheap: no reasoning required)
       └─ READY found → execute task  (this is where the LLM earns its keep)
```

Result: 671 wasted reasoning sessions collapse to 671 near-free grep operations.

**State tracking:** `check-todos.sh` records last-fired timestamps in `shared/todos/calendar-state.json` — not in the agent's memory. Deduplication survives session resets.

**CALENDAR.md format:**
```
# Format: DAYS HH:MM UTC | task description
DAILY    14:00 | Run data-sync script per runbooks/RUNBOOK_DAILY_SYNC.md
MON,THU  14:00 | Send notification emails per runbooks/RUNBOOK_NOTIFICATIONS.md
```

### 5.5 What Belongs Where (Scheduling)
- **HEARTBEAT.md:** Things the agent should do on *every* heartbeat, for the lifetime of the agent. Check the TODO queue. Scan for rejected emails. Infrastructure-level reflexes only.
- **HEARTBEAT.md does NOT belong:** Day-of-week logic, time-window checks, "only on Mondays." This creates a fragile implicit state machine and wastes tokens.
- **TODO.md does NOT belong:** Recurring tasks. If a task needs to happen again next week, it belongs in CALENDAR.md. An agent that re-schedules recurring tasks in TODO.md will forget after a session reset.

### 5.6 Session Management (Critical for Local Models)
OpenClaw replays the full conversation history on every call. As history grows, every interaction gets slower.

**Prefill vs. generation** (why this matters more than you think):
- *Generation* (output tokens) runs at ~50 tps on a GB10 — the number quoted in benchmarks
- *Prefill* (processing input) runs in parallel across all input tokens simultaneously: 10,000 tokens prefills in **1–3 seconds**, not 200 seconds
- Session history is the real latency risk: unlike the system prompt (bounded at 150K chars), session history grows without a hard cap

**Session management cron jobs:**
```bash
# Monitor session sizes every 5 min
*/5 * * * *  monitor-sessions.sh

# Archive and truncate sessions >512KB daily at 3am
0 3 * * *    reset-sessions.sh

# Seed missing heartbeat session files after container restarts
*/30 * * * * seed-sessions.sh
```

### 5.7 The Full Crontab
Five cron scripts constitute the entire "automation layer" outside the LLM:

```
*/5  * * * * check-todos.sh        # scheduling engine
*/5  * * * * monitor-sessions.sh   # session size logging
*/30 * * * * seed-sessions.sh      # heartbeat session recovery
0 3  * * *   reset-sessions.sh     # daily session archive/truncate
*/30 * * * * send-approved-emails.sh  # human-approved email sender
```

All run on the **host** as the deploying user — not inside any Docker container.

---

## Module 6 — Runbooks, Scripts, and Integrations *(10 min)*

### 6.1 RUNBOOKs: Procedures the Agent Follows
A `RUNBOOK_<topic>.md` file in the agent's `runbooks/` directory gives step-by-step instructions for a specific recurring task.

The three-file authoring pattern for any new recurring duty:
1. One line in `IDENTITY.md` — *what* the duty is
2. One file `runbooks/RUNBOOK_X.md` — *how* to execute it
3. One line in `CALENDAR.md` — *when* to execute it

RUNBOOKs are **read fresh from disk on every trigger** — not held in memory. This makes them the most reliable vehicle for procedural changes. Change the runbook, and the next trigger follows the new procedure exactly.

*Example: `RUNBOOK_EMAIL_DIGEST.md`*
- Triggered daily at 08:00 AM Central
- Step 1: Fetch today's inbox via `gmail_api.py` (deterministic — script, not LLM)
- Step 2: Check each sender against contacts (script)
- Step 3: LLM composes a digest from the results (judgment — LLM earns its keep here)
- Step 4: Send email (pre-approved standing action — no per-message confirmation)

### 6.2 Scripts: Deterministic Tools for the Agent

| Operation type | Owner | Why |
|---|---|---|
| *When* to act | `check-todos.sh` (bash) | Deterministic; zero tokens |
| *How* to move/transform data | `scripts/*.py` (Python) | Deterministic; no LLM drift |
| *What* to say / *whether* to act | LLM | Requires judgment |

Scripts are essentially **registered tools** for the agent — like function-calling in an API context, but running deterministically inside the sandbox. The RUNBOOK tells the agent which tool to call and what to do with the output.

Key scripts in the worked example:
- `gmail_api.py` — search, read, and send email via Gmail API (stdlib-only, no pip)
- `contacts_api.py` — search and create contacts via Google People API (stdlib-only)
- `sync-track-sheets.py` — reads master Google Sheet, routes rows, writes counts JSON
- `notify.py` — calls the LLM API directly (for email composition), writes outbox JSON

### 6.3 Slack Integration
- Create a Slack app at api.slack.com/apps with **Socket Mode** enabled
- Required bot scopes: `channels:history`, `chat:write`, `users:read`, etc. (**do not add `assistant:write`**)
- `botToken` and `appToken` go into `openclaw.json`
- Multiple agents on one Slack app: use `bindings` to route specific channels to specific agents
- One agent marked `"default": true` handles all DMs and unrouted messages

**Outbound (agent-initiated) posts:** Agents can reply within active sessions, but cannot initiate posts to a channel when no session is active. The **slack-outbox pattern** handles this:
- Agent writes JSON to `shared/slack-outbox/`
- `send-slack-posts.sh` (cron every 5 min) posts and archives
- Suitable for automated overnight summaries, scheduled notifications

### 6.4 Google/Gmail Integration
Two approaches, and why we moved from one to the other:

| Tool | API used | Problem |
|---|---|---|
| `gog` CLI | `threads.list` | Returns first message of thread; ignores `in:sent` filter |
| Direct API via `urllib` | `messages.list` | Correctly honors all Gmail query operators |

Current approach: `gsuite-mcp` handles OAuth (browser flow → writes `token.json`); `gmail_api.py` and `contacts_api.py` read that token directly. No third-party dependencies, no keyring daemon.

Sandbox bind mounts needed:
```
~/.local/share/gsuite-mcp  →  /tmp/.local/share/gsuite-mcp  (rw — token refresh)
~/.config/gsuite-mcp       →  /tmp/.config/gsuite-mcp       (ro — credentials)
/usr/local/bin/gmail-agent →  /scripts                       (ro — agent scripts)
```

### 6.5 OpenClaw-Gmail as a Worked Example
The `OpenClaw-Gmail` repo is a self-contained, parameterized starting point for anyone who wants a Gmail agent without building from scratch. Key features:
- Daily email digest (8 AM local)
- Contact hygiene: harvest sender/recipient addresses from sent mail
- Writing style learning: analyze sent mail, maintain a style guide
- Monthly style refresh
- Inbox triage: action items and urgency flags surfaced to Slack twice daily
- All placeholder variables in one `sed` substitution

*This repo will be the basis for the hands-on lab.*

---

## Module 7 — Operational Excellence and Lessons Learned *(5 min)*

### 7.1 Daily Operations
```bash
# Start (vLLM first, then OpenClaw)
cd ~/code/spark-ai/qwen3-coder-next && docker compose up -d
cd ~/code/spark-ai/openclaw && docker compose up -d

# Stop
cd ~/code/spark-ai/openclaw && docker compose down
cd ~/code/spark-ai/qwen3-coder-next && docker compose down

# Config change workflow
docker exec openclaw-gateway cat /home/node/.openclaw/openclaw.json > /tmp/edit.json
# edit /tmp/edit.json
docker cp /tmp/edit.json openclaw-gateway:/home/node/.openclaw/openclaw.json
docker compose restart openclaw-gateway
```

### 7.2 Update Strategy
Neither script downloads or pulls anything without `--update`:
- `check_openclaw.sh` — compare running container digest vs. GHCR manifest (HEAD request only)
- `check_model.sh` — compare local vs. remote HuggingFace commit hash

Before any update: read the release notes, use the embedded upgrade prompt generator to get AI-assisted impact analysis.

### 7.3 Top Lessons Learned from a Working Deployment

1. **Use host paths for workspace** — not container-internal paths. Docker resolves bind-mount sources on the host. *(Cost of ignoring: empty sandbox, agent with no identity)*

2. **Disable config writes before connecting Slack** — the first thing to do after onboarding. A single confused agent response can crash the gateway. *(Cost of ignoring: crash loop, potential config corruption)*

3. **Always sandbox** — never use `tools.exec.host: "gateway"`. Sandbox mode is not a nice-to-have; it is the primary containment layer. *(Cost of ignoring: agent execs run on the host as your user)*

4. **Session history is the main reliability and latency risk** — not context size. Reset sessions after any important behavioral change, and daily via cron. *(Cost of ignoring: old behavior persists; latency grows unbounded)*

5. **Critical rules belong in SOUL.md** — rules in secondary files may never be re-read. *(Cost of ignoring: approval requirements silently stop being enforced)*

6. **Respect the separation of powers** — "LLM does it" is not a plan for data movement, counting, scheduling, or API calls. If it is deterministic, write a script. The principle is established in §2.3; the cost of eroding it compounds silently. *(Cost of ignoring: non-deterministic behavior, token waste, unreproducible failures)*

7. **TODO.md vs. CALENDAR.md** — one-shots vs. recurring. An agent that re-schedules recurring tasks in TODO.md will forget after a session reset. *(Cost of ignoring: tasks silently disappear after the next reset)*

8. **The outbox pattern for anything with real-world consequences** — email, Slack announcements, external API mutations. Queue, review, send. *(Cost of ignoring: your agent sends email in your name, unsupervised)*

9. **Prompt injection is a real threat for email agents** — malicious email body content can instruct the agent to run commands. Sandbox mode limits the blast radius, but a dedicated low-permission account limits it further.

10. **Verify, don't assume** — after any configuration or behavioral change, verify the next execution follows the new rule. The LLM does not send acknowledgment receipts.

---

## What's Next: The Hands-On Lab

The hands-on session (to be developed separately) will use `OpenClaw-Gmail` as the worked example. Participants will:
1. **Choose their model path** — edit `config.yaml` and `secrets.yaml` for local GPU (vLLM) or cloud API (Anthropic Claude); run `apply-config.sh` and verify the gateway picks it up
2. Stand up the OpenClaw gateway with security hardening applied
3. Set up Google Cloud credentials and OAuth via gsuite-mcp
4. Configure Slack bot integration
5. Parameterize and deploy the Gmail agent workspace (one `sed` substitution)
6. Add a cron-driven scheduling entry and observe it fire
7. Practice the behavior lock-in procedure: edit → reset → verify
8. Review the email outbox approval workflow end-to-end

*Students using cloud Claude and students using a local GPU can work in parallel — the only difference is one line in `config.yaml`.*

---

## Appendix: Key Files Quick Reference

| File | Location | Purpose |
|---|---|---|
| `SOUL.md` | `<agent>/` | Hard behavioral invariants; loaded every call |
| `IDENTITY.md` | `<agent>/` | Who the agent is; Slack contract; loaded every call |
| `HEARTBEAT.md` | `<agent>/` | Always-on reflexes; loaded every call |
| `CALENDAR.md` | `<agent>/` | Recurring duties; read by bash, not LLM |
| `TODO.md` | `<agent>/` | One-shot tasks; written by agent; gitignored |
| `PATHS.md` | `<agent>/` | Canonical absolute paths; read explicitly |
| `RUNBOOK_X.md` | `<agent>/runbooks/` | Step-by-step procedures; read at trigger time |
| `check-todos.sh` | `scripts/` | Bash scheduling engine; run via cron on host |
| `send-approved-emails.sh` | `scripts/` | Sends human-approved outbox emails |
| `reset-sessions.sh` | `scripts/` | Archives + truncates large session files daily |
| `openclaw.json` | Inside gateway container | Live gateway configuration |
| `config.yaml` | `spark-ai/` | Per-agent model assignments |
| `secrets.yaml` | `spark-ai/` | API keys; gitignored |

---

*Outline version: 2026-03-18 rev 2. Developed from spark-ai, spark-ai-agents, and OpenClaw-Gmail repositories.*
