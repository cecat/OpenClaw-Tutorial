# Getting Started with OpenClaw: A Safety-First Approach
## Tutorial Outline — 90-Minute Lecture

**Target audience:** Developers with working knowledge of GitHub, Python, shell scripting, Docker, and Linux (crontab). No prior OpenClaw experience required.

**Central theme:** Autonomous agents are powerful and genuinely useful — and that power demands deliberate design. This tutorial builds a working OpenClaw deployment with safety, containment, and alignment as first-class design goals, not afterthoughts.

**Tutorial repository:** `github.com/cecat/OpenClaw-Tutorial` — contains this outline, the slide deck, and (when the hands-on lab is developed) a `gateway/` directory with platform configuration and an `agents/` directory with a parameterized agent workspace ready to deploy.

---

## Module 1 — What Is OpenClaw and Why This Approach? *(10 min)*

### 1.1 What OpenClaw Is (and Isn't)
- OpenClaw is an **agent framework**, not a model — it gives a language model memory, tools, communication channels, and a schedule
- Open source, self-hosted: you own the configuration, credentials, history, and behavior
- Compatible with local models (via vLLM) and cloud APIs (Anthropic Claude, OpenAI)
- Communicates through Slack, a web dashboard, Telegram, and 50+ other channels
- The model provides cognition; OpenClaw provides orchestration, file access, tool execution, and memory

### 1.2 What an Agent Actually Does (that a chatbot doesn't)
- A chatbot waits for a question and answers it. An agent acts between conversations — on a schedule, in response to events, and on its own judgment about what to do next.
- An agent can send your email, post to Slack, read your calendar, modify files, and call APIs — autonomously, while you sleep.
- This is genuinely useful. It is also qualitatively different from a chatbot, and must be designed accordingly.
- OpenClaw agents can be partially autonomous without being fully autonomous — calibrating that balance deliberately is what this tutorial is about.

### 1.3 Why "Safety-First"?
- Security and containment have historically been afterthoughts with exciting new technology — something addressed after adoption, once incidents accumulate, rather than before. OpenClaw is no exception to this pattern. The platform is powerful, the demos are compelling, and the natural instinct is to get something running and worry about hardening later.
- The consequences of that pattern are well-documented in the OpenClaw ecosystem. By early 2026, SecurityScorecard's STRIKE team identified over 135,000 OpenClaw instances exposed to the public internet, with over 15,000 specifically vulnerable to remote code execution (CVE-2026-25253, the "ClawJacked" WebSocket hijack). The ClawHavoc campaign poisoned roughly 20% of the ClawHub skills registry with packages delivering credential stealers and backdoors — exploiting the reasonable but mistaken assumption that a public registry was curated or reviewed.
- We cover safety, containment, and alignment in Module 2 — before setup, before configuration, before anything else — because the decisions you make in the first hour determine the blast radius of everything that goes wrong later. Understanding the risks is part of understanding the platform.
- **The right mental model:** your agent acts in your name. Every email it sends is from you. Every Slack post is from you. Design accordingly — not by neutering the agent, but by designing the right boundaries. *(Module 2 covers this in full.)*

### 1.4 Architecture Summarized

An OpenClaw deployment consists of a language model and a gateway, connected by a Docker internal network that is deliberately not exposed to the outside world. The gateway manages agent identities, conversation sessions, tool execution, and channel integrations (Slack, web dashboard, and others). Agent personality, memory, and task procedures live in plain markdown files in a workspace directory on the host — readable, auditable, and well-suited to version control.

When an agent executes code or runs a script, it does so inside an ephemeral Docker sandbox container that is discarded after the task completes. That sandbox has only the network access you explicitly grant it — and we explicitly block it from reaching your LAN, other hosts, and SSH services via iptables rules. The gateway itself listens only on your Tailscale interface (a private VPN IP), making it unreachable from the public internet or your local network. Tailscale also provides HTTPS via its `serve` command, so the dashboard is accessible securely from your other devices without opening any ports.

Scheduling is entirely host-side: cron scripts running on the host decide when to queue work; the LLM decides what to do with it. This separation — shell owns the clock, LLM owns judgment — is one of the design principles we'll return to throughout the tutorial.

*The diagram and full setup walkthrough are in Module 3.*

### 1.5 What This Lecture Covers
1. The design charter: safety, containment, alignment, and separation of powers
2. Platform architecture and choosing your model
3. Agent identity: the files that define who your agent is
4. Scheduling: why the shell owns the clock
5. The scaffolding: runbooks, scripts, and cron
6. Integrations: Slack and Google
7. Lessons learned from a working deployment

*The hands-on lab (a separate session) puts this into practice using `OpenClaw-Gmail` as the worked example.*

---

## Module 2 — The Design Charter *(15 min)*

> **This module is the foundation.** Every architecture decision in Modules 3–7 — sandboxing, iptables rules, the outbox pattern, SOUL.md, the scheduling design, the scaffolding — exists because of the principles established here. We cover it second, not last, because you should understand *why* before you are asked to execute *what*.

### 2.1 The Threat Model
Four risk categories when running an autonomous agent. These are not hypothetical:

1. **Prompt injection** — malicious content in email, files, or Slack instructs the agent to execute unintended commands. The agent cannot distinguish a legitimate instruction from one embedded in adversarial input unless the system is designed to limit what it can do with that input. This is a structural vulnerability: current LLMs do not enforce separation between instructions they are meant to execute and data they are meant to process. [*cite: Zverev et al., "Can LLMs Separate Instructions From Data?", ICLR 2025*]

2. **Lateral movement** — a sandbox container that can reach your LAN, other Tailscale nodes, or SSH services can be used as a pivot point. This is especially relevant when the agent's sandbox runs on a machine that also has access to other internal services.

3. **Runaway external actions** — the agent sends email, posts to public channels, or mutates external state without human review. It does this not out of malice but because it is trying to be helpful, and "helpful" was not bounded precisely enough.

4. **Credential exfiltration** — API keys, OAuth tokens, and other secrets passed to the sandbox can leak via outbound HTTP if the sandbox has unrestricted network access.

5. **Supply chain / skills registry poisoning** — third-party skills installed from a public registry (ClawHub) may contain malicious code. The ClawHavoc campaign (January–February 2026) poisoned approximately 20% of ClawHub's registry with skills delivering credential stealers and backdoors; the only requirement to publish had been a GitHub account one week old, with no code review or signing. Any skill installed from a registry is code that runs inside your sandbox with the access you have granted it. Treat it accordingly.

### 2.2 Alignment: Defining What the Agent Must Never Do
The core alignment document is `SOUL.md` — one of exactly 8 files loaded into every inference call, every time. Alignment rules that live anywhere else may be invisible to the model.

Essential rules to establish before connecting anything external:
- **Stop means stop.** "DO NOT PROCEED," "WAIT," and "STOP" are not invitations to acknowledge and continue. They mean halt and report.
- **The agent does not touch the gateway or its own configuration.** Not to fix a bug, not to improve performance, not to help with debugging. It reports and waits.
- **External communications go through the outbox.** The agent queues; a human or deterministic script sends.
- **Deferred work goes to TODO.md.** The agent does not sleep, loop, or block waiting for a future time. It writes the task and returns.
- **The cost of stopping unnecessarily is low. The cost of acting incorrectly is high.** Encode this asymmetry explicitly.

### 2.3 Separation of Powers: Code for Procedure, LLM for Judgment

This is the architectural principle that generates the entire scaffolding of runbooks, scripts, and cron jobs built in Modules 5 and 6. It is not primarily about token cost — it is about reliability, auditability, and testability.

> **Anything deterministic is owned by code. Anything requiring judgment, creativity, or natural language understanding or generation is owned by the model. The boundary between them must be sharp and explicit.**

The two sides of the boundary:

| Belongs to code | Belongs to the LLM |
|---|---|
| When to act (cron schedule) | Whether this email needs a reply |
| Whether a TODO item is due (bash date comparison) | What the reply should say |
| Fetching messages from the Gmail API | Identifying action items in fetched email |
| Routing rows in a spreadsheet by column value | Summarizing ambiguous content |
| Counting, sorting, filtering structured data | Inferring priority from unstructured text |
| Moving a file from outbox to sent | Deciding whether to approve a draft |
| Sending an HTTP request with a known payload | Composing the payload from context |

**Why this matters more than token cost:** Code is deterministic — the same input always produces the same output. LLM output is probabilistic, shaped by context, session history, and model version. Code is auditable — you can read exactly what it does. Code changes are precise — change one line, one behavior changes on the next execution, every time. An LLM "rule" is a suggestion embedded in context; it usually holds, until session history, model drift, or a long conversation erodes it.

This design principle has a growing body of support in the research literature. Three converging lines of work are worth calling out:

- **Architecture surveys** confirm that the spectrum from fully LLM-driven to structured/deterministic agent designs is a recognized axis of variation, and that reliability-critical applications systematically require more deterministic orchestration. [*cite: Masterman et al., "The Landscape of Emerging AI Agent Architectures for Reasoning, Planning, and Tool Calling: A Survey," arXiv:2404.11584, 2024*]

- **MetaGPT** (ICLR 2024 oral) provides a compelling existence proof: by encoding Standard Operating Procedures (SOPs) into code and assigning deterministic roles to agents, it dramatically reduces cascading hallucinations caused by naively chaining LLMs — the same class of error that motivates our runbook + script scaffolding. [*cite: Hong et al., "MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework," ICLR 2024, arXiv:2308.00352*]

- **Blueprint First, Model Second** makes the argument most directly: a deterministic orchestration engine manages workflow structure while the LLM handles bounded, discrete sub-tasks — transforming agent behavior from unpredictable exploration into a verifiable and auditable process. [*cite: Qiu et al., "Blueprint First, Model Second: A Framework for Deterministic LLM Workflow," arXiv:2508.02721, 2025*]

**A note on where the field is going:** Language models are improving rapidly and will increasingly be capable of calling tools, managing state, and making scheduling decisions reliably on their own. We value the code/LLM separation now because it gives us precision, auditability, and token efficiency with today's models. As models improve, the right boundary will shift — but the habit of thinking clearly about which layer owns which responsibility will remain valuable regardless.

*(The "token waste" problem is quantified in Module 5 with a concrete table.)*

*[TODO: locate the specific talk or paper by "Hugo" on this topic — the presenter made a compelling quantitative argument for this separation, possibly in a 2024 NeurIPS or ICML context. Confirm speaker's last name and add citation.]*

**The scaffolding this principle generates:**

The LLM sits atop three layers of deterministic scaffolding. Each layer calls down only — the LLM invokes runbooks, runbooks invoke scripts, scripts are triggered by cron. Information flows back up; control flows down.

```
┌─────────────────────────────────────────────────────────────┐
│  LLM — the OpenClaw agent                                   │
│  Compose · Summarize · Triage · Decide · Communicate        │
│  "What does this mean, and what should we do?"              │
├─────────────────────────────────────────────────────────────┤
│  RUNBOOKS  (markdown, read fresh from disk at trigger time) │
│  "Which tools to call and in what order"                    │
├─────────────────────────────────────────────────────────────┤
│  SCRIPTS  (Python/bash, run deterministically in sandbox)   │
│  "How to move, fetch, and transform data"                   │
├─────────────────────────────────────────────────────────────┤
│  CRON  (host-side, zero tokens, fully deterministic)        │
│  "When to act — the scheduling engine"                      │
└─────────────────────────────────────────────────────────────┘
```

This is not a strict call hierarchy — cron triggers the heartbeat, which triggers the LLM, which reads a runbook, which calls a script, whose output the LLM then acts on. But the separation of responsibilities is strict: no layer does work that belongs to another.

### 2.4 Containment: The Technical Layers

A layered defense. Each layer is described in detail when it appears in setup — listed here so the rationale is clear before the mechanics:

| Layer | Mechanism | Threat it addresses |
|---|---|---|
| Network | Tailscale-only gateway binding | Dashboard not reachable from internet or LAN |
| Network | iptables DOCKER-USER rules | Sandbox containers cannot reach LAN, Tailscale, or SSH |
| Execution | Sandbox mode: `mode: all` | Agent commands run in throwaway containers, not on the host |
| Execution | No `docker.sock` in sandbox | No container escape |
| Config | Disable `commands.config` writes | Agent cannot modify gateway configuration |
| Credentials | Separate dedicated accounts | Compromised credential has limited blast radius |
| Gateway exposure | Tailscale-only binding (not `0.0.0.0`) | Prevents WebSocket hijack from malicious websites (CVE-2026-25253 / "ClawJacked" class of attack) |

*When you encounter each of these in Modules 3–7, the "why" is already established here.*

The gap between what is possible and what is commonly deployed is striking: a 2025 survey of deployed agentic AI systems found that the majority document no sandboxing or containment mechanisms at all. [*cite: "The 2025 AI Agent Index," MIT, arXiv 2602.17753, 2025*] We treat these layers as non-negotiable precisely because the default is to omit them.

### 2.5 The Human-in-the-Loop Pattern
The **outbox pattern** is the cornerstone of supervised external action:

```
Agent writes draft JSON → shared/outbox/  (status: "pending")
Human or supervisor agent reviews → marks "approved" or "rejected"
Deterministic cron script sends approved items → archives to sent/
Full audit trail preserved in outbox / sent / rejected
```

- The LLM never directly sends email; it writes to a queue
- A deterministic script sends and archives — no LLM judgment at send time
- The pattern applies to any action with real-world consequences: email, Slack posts, external API mutations

### 2.6 Charter Summary
Five principles that drive every decision in this tutorial:

1. **Separation of powers** — code owns anything deterministic; the LLM owns judgment, creativity, and language. The boundary is sharp and explicit.
2. **Containment first** — network, execution, and config isolation before connecting anything external.
3. **Alignment by design** — `SOUL.md` behavioral invariants are established before the agent is given tools, and are non-negotiable.
4. **Human in the loop** — any external action with real-world consequences goes through an approval queue.
5. **Verify, don't assume** — after any behavioral or configuration change, confirm the next execution follows the new rule. The agent does not send receipts.

---

## Module 3 — Platform Architecture and Model Choice *(15 min)*

### 3.1 The Docker Stack
```
[You, via Tailscale]
        |
        v  port 18789 — Tailscale interface only, never 0.0.0.0
[OpenClaw gateway container]
        |
        v  http://nim:8000/v1 — Docker-internal network only
[vLLM container]  ← only present for local-model deployments
        |
        v
[GPU + unified memory]
```

Two key design choices, both mandated by the design charter:
- The model API endpoint is never exposed to the host or network — only reachable within the Docker internal network
- The gateway listens only on the Tailscale interface — unreachable from the public internet or your LAN

**What is Tailscale?** Tailscale is a zero-config VPN that assigns each enrolled device a stable private IP (in the `100.x.x.x` range). By binding the OpenClaw port to your machine's Tailscale IP, the dashboard is only reachable from other devices on your Tailscale network (your laptop, phone, etc.) — not from the internet or your local LAN. It also provides HTTPS via `tailscale serve`. Students must have Tailscale installed and running before the hands-on lab.

### 3.2 Choosing Your Model

Students choose one path at setup time and can switch at any time with a single config change:

| Path | Hardware requirement | Model examples | Characteristic |
|---|---|---|---|
| Local GPU | Machine with dedicated GPU and sufficient memory (e.g., NVIDIA DGX Spark GB10, high-end workstation, Mac with Apple Silicon M-series) | Qwen3-Coder-Next-FP8 via vLLM | Low per-token cost; throughput depends on hardware; typically <100 tps |
| Cloud API | Any Linux host (no GPU required) | Anthropic Claude (claude-sonnet-4-6, claude-haiku-4-5) | Pay-per-token; consistently fast; easier setup |

**On running a local model:** The key constraint is memory, not just GPU. A 32 GB unified memory machine (e.g., M-series Mac) can run smaller quantized models but will see lower throughput and may struggle with large context windows. Throughput matters for scheduled tasks — a 50-tps model can produce a 500-token email digest in 10 seconds; a 5-tps model takes 100 seconds. If you are not already confident your hardware can run a capable model, the cloud API path removes this variable entirely and lets you focus on the agent architecture, which is the point of this tutorial.

**Our recommendation for the tutorial:** Use the cloud API (Anthropic Claude) unless you already have local model inference working. The architecture is identical — the only difference is one line in `config.yaml`.

### 3.3 config.yaml: Your Model Switch

`config.yaml` lives in the infrastructure repo. It is the single file that controls which model backs each agent. Changes are applied without editing `openclaw.json` by hand.

```yaml
# config.yaml — per-agent model assignments
agents:
  main:
    model: anthropic/claude-sonnet-4-6    # cloud path
  gmail-agent:
    model: vllm/Qwen/Qwen3-Coder-Next-FP8  # local path
```

**`secrets.yaml`** (gitignored, never committed) holds API keys:
```yaml
anthropic_api_key: sk-ant-...
```

**Applying a change:**
```bash
./apply-config.sh --dry-run   # preview the patch to openclaw.json
./apply-config.sh             # apply and restart the gateway
```

**Emergency revert to local model:**
```bash
./revert-to-local.sh          # strips all cloud config, restarts gateway
```

The point: you do not need to choose once and live with it. Switch any agent to a different model in under 30 seconds. Students using cloud Claude and students using a local GPU work identically from this point forward.

### 3.4 Repository Structure

For the tutorial, students work with a single private repository:

```
OpenClaw-Tutorial/               ← your private repo
├── spark-ai/                    ← companion public repo (submodule or clone)
│   ├── docker-compose files
│   ├── config.yaml              # model assignments — edit this
│   ├── secrets.yaml             # gitignored — API keys
│   └── apply-config.sh
└── gmail-agent/                 ← from OpenClaw-Gmail, parameterized for you
    ├── IDENTITY.md
    ├── SOUL.md
    ├── TOOLS.md
    ├── ...                      # the 8 auto-loaded files
    ├── runbooks/
    └── scripts/
```

A single repo keeps things simple. The public/private split (two repos) is an option for those who want to share their infrastructure config with others — it is not required and adds coordination overhead most students don't need.

### 3.5 Security Hardening Checklist

Done once, before connecting any external service. Four verifications before calling the platform ready:

```bash
# vLLM NOT exposed to host network
ss -ltnp | grep :8000          # must show nothing

# OpenClaw bound to Tailscale IP only
ss -tlnp | grep 18789          # must show TAILSCALE_IP:18789, not 0.0.0.0

# No SSH keys in gateway container
docker compose exec openclaw-gateway ls /home/node/.ssh   # must fail

# Sandbox container network isolation (3 DROP rules)
sudo iptables -L DOCKER-USER -n | grep DROP   # must show 3 rules
```

---

## Module 4 — Agent Identity: The Workspace and the 8 Files *(15 min)*

### 4.1 The Workspace
An agent's workspace is a directory of plain markdown files mounted into the OpenClaw gateway container. Everything the agent knows about itself, its user, its tools, and its duties lives in these files — readable, auditable, editable in any text editor, and version-controlled in git.

OpenClaw automatically loads exactly 8 of these files into the system prompt on every inference call. Everything else is invisible to the model unless the agent explicitly reads it with a tool call.

**The files OpenClaw auto-loads (in order):**

| # | File | What it holds |
|---|---|---|
| 1 | `AGENTS.md` | Sub-agent definitions and routing |
| 2 | `SOUL.md` | Persona, values, and hard behavioral invariants — the alignment document |
| 3 | `TOOLS.md` | How to invoke external tools: scripts, APIs, gog, gsuite-mcp |
| 4 | `IDENTITY.md` | Who the agent is; Slack behavior contract; list of recurring duties |
| 5 | `USER.md` | User context, preferences, communication style, and background |
| 6 | `HEARTBEAT.md` | Always-on reflexes executed on every heartbeat |
| 7 | `BOOTSTRAP.md` | First-run orientation — how to set up a fresh identity |
| 8 | `MEMORY.md` | Accumulated runtime knowledge promoted from daily memory files |

**Budget:** 20,000 characters per file; 150,000 characters total system prompt. If a file exceeds its budget, OpenClaw keeps the first 70% and last 20%.

**Everything else is NOT auto-loaded:** `CALENDAR.md`, `EMAIL.md`, `PATHS.md`, `CHANNELS.md`, runbooks, templates — invisible unless the agent reads them explicitly. More on this shortly.

### 4.2 What Each File Should (and Should Not) Contain

**`SOUL.md`** holds the behavioral invariants that must hold unconditionally. If a rule must never be violated, it belongs here — not in a secondary file. Keep it focused: this is constitutional law, not policy.

**`IDENTITY.md`** describes who the agent is — its name, its Slack behavior contract, and a concise list of its recurring duties. Duties are listed here (what and when), but the how belongs in runbooks (read separately, covered in Module 6). Do not duplicate rules from SOUL.md here — reference them.

**`TOOLS.md`** is the agent's tool manual. It explains how to call each script, what arguments it takes, and what it returns. When a new script is added to the workspace, TOOLS.md gets a new entry.

**`USER.md`** holds everything the agent needs to know about you: your background, preferences, communication style, and context. This file is what makes the agent feel personal rather than generic.

**`HEARTBEAT.md`** contains only always-on reflexes — things the agent does on every heartbeat for the lifetime of the agent. Check the TODO queue for READY items. Scan for rejected emails. Infrastructure-level reflexes only. Day-of-week logic, time-window checks, and task-specific procedures do not belong here.

**`BOOTSTRAP.md`** is the first thing a fresh agent reads. It walks the agent through establishing its identity — choosing a name, filling in USER.md, understanding its tools. Best practice: make this the subject of the very first message to a new agent, before anything else. After initial setup, BOOTSTRAP.md becomes dormant orientation material.

**`MEMORY.md`** is runtime knowledge promoted from daily memory files (`memory/YYYY-MM-DD.md`). The agent writes daily entries during conversations; important facts get promoted here for long-term retention. This is one of the two files the agent actively writes to (the other being `TODO.md`).

### 4.3 The Files That Are NOT Auto-Loaded — and Why That Matters

Several important files live in the workspace but are invisible to the agent unless explicitly requested:

**`PATHS.md`** — the canonical registry of every absolute path used by the agent: workspace root, shared directory, script locations, outbox paths. When we discovered that path strings were being duplicated across TOOLS.md, multiple runbooks, and HEARTBEAT.md — and drifting out of sync as the workspace evolved — we created PATHS.md as the single source of truth. All other files now reference PATHS.md rather than hard-coding paths. The agent reads PATHS.md at the start of any task involving file I/O.

**`CHANNELS.md`** — a registry of which Slack channel maps to which agent, including channel IDs and routing rules. Needed as soon as you have more than one agent sharing a Slack workspace.

**`CALENDAR.md`** — the recurring duty schedule. Critically, this file is not read by the LLM to decide what to do — it is read by `check-todos.sh` (a bash script running on the host) to determine when to queue work for the agent. The separation is deliberate and central to the scheduling architecture in Module 5.

**`EMAIL.md`, templates, runbooks** — task-specific reference material read on demand. Because these are not auto-loaded, rules or paths that only appear in them may never reach the agent in a given session.

The implication: **anything the agent must do correctly and consistently must live in one of the 8 auto-loaded files.** Everything else is reference material.

### 4.4 The Single-Source-of-Truth Principle in Practice

The most insidious class of agent configuration bug is the duplicated instruction. When the same rule, path, or fact appears in more than one file, they will eventually diverge — and the agent will silently blend contradictory signals into inconsistent behavior.

Examples of how this manifests:
- `SOUL.md` says "always get approval before sending email." `IDENTITY.md` has an older version of the same rule, phrased differently, with an implicit exception you added and forgot to add to SOUL.md. The agent interprets the combination as having a situational exception — sometimes it asks, sometimes it doesn't.
- A script path is hard-coded in `TOOLS.md`, `RUNBOOK_EMAIL_DIGEST.md`, and `HEARTBEAT.md`. The scripts directory is renamed. You update two of the three. The third silently fails on the next trigger.

The fix is architectural: **one authoritative home for each fact.** PATHS.md for paths. SOUL.md for behavioral invariants. IDENTITY.md for duties. Other files reference these — they do not repeat them.

### 4.5 How Reliably Does the Agent Follow the 8 Files?

More reliably than secondary files, but not unconditionally. There are two important nuances:

**Instructions vs. examples.** OpenClaw rebuilds the system prompt from the 8 files on every call — but it also sends the full session history (the accumulated back-and-forth since the last session reset). When instruction text and behavioral examples in the history conflict, examples often win. An agent with many examples of old behavior in its session history may not immediately adopt a newly-written rule, even in SOUL.md.

**What this means for updates.** Updating a file on disk does not guarantee the agent immediately adopts the new behavior. Any behavioral change requires four steps: (1) edit the right file, (2) reset the session to clear counter-examples from history, (3) explicitly tell the agent about the change in the first message of the new session, (4) verify the next execution follows the new rule.

**Reliability tiers:**

| Source | When agent sees it | Reliability |
|---|---|---|
| One of the 8 auto-loaded files | Every inference call | High — with session reset after changes |
| Non-auto-loaded file | Only when agent explicitly reads it | Indeterminate |
| A runbook (read at trigger time) | Fresh from disk on every trigger | Most reliable for procedural changes |
| Session history examples | Every call until session reset | Works against you after behavioral changes |

---

## Module 5 — Scheduling: The Shell Owns the Clock *(12 min)*

### 5.1 The Core Principle

> **The decision of when to act is always owned by deterministic code. The act of doing the work is owned by the LLM. These two responsibilities must never be mixed.**

An LLM has no clock. It has no reliable memory of having acted in a previous session. If you ask it to decide whether it is time to run a task, it will reason through that question every time it runs — and it will sometimes be wrong.

### 5.2 The Token-Waste Problem

A 15-minute heartbeat fires **672 times per week**. If `HEARTBEAT.md` instructs the agent to check whether it's time to run a weekly report, the agent reasons through that question 672 times — and acts on 1 of them:

| Task frequency | Heartbeats/week | Useful fires | Wasted LLM calls |
|---|---|---|---|
| Once a week | 672 | 1 | 671 (99.9% waste) |
| Mon + Thu | 672 | 2 | 670 (99.7% waste) |
| Weekdays at 9am | 672 | 5 | 667 (99.3% waste) |

Beyond token cost: the agent must remember whether it already ran the task today — state that does not survive a session reset. LLM-driven scheduling fires twice, then not at all, then twice again.

### 5.3 Three Tiers of Scheduling

| Tier | File | Owned by | Lifecycle | Use for |
|---|---|---|---|---|
| Always-on | `HEARTBEAT.md` | Human | Permanent | Infrastructure reflexes on every heartbeat |
| Recurring | `CALENDAR.md` | Human | Season or project | Day-of-week, daily, monthly recurring tasks |
| One-shot | `TODO.md` | Agent (at runtime) | Ad hoc | Single deferred tasks written during a conversation |

**The human analogy:** HEARTBEAT.md is brushing your teeth — every day, forever, no thought required. CALENDAR.md is "send the Monday/Thursday reports through June." TODO.md is "call the dentist" or "pick up eggs."

**The critical distinction for TODO.md vs. CALENDAR.md:** If a task needs to happen again next week, it belongs in CALENDAR.md. An agent that re-schedules recurring tasks by writing new TODO.md entries will lose track of them after a session reset — they disappear silently.

### 5.4 The Scheduling Engine: check-todos.sh

```
cron (every 5 minutes — zero tokens)
  └─ check-todos.sh
       ├─ reads CALENDAR.md → is a recurring entry due? → writes READY to TODO.md
       └─ reads TODO.md → is a one-shot past its time? → marks READY in-place

OpenClaw heartbeat (every 15 minutes — tokens only when READY work exists)
  └─ agent greps TODO.md for READY items
       ├─ none found → HEARTBEAT_OK      (cheap — no reasoning required)
       └─ READY found → execute task     (LLM earns its keep here)
```

**State tracking:** `check-todos.sh` records last-fired timestamps in `shared/todos/calendar-state.json` on the host — not in the agent's memory. Deduplication survives session resets.

**CALENDAR.md format:**
```
# DAYS HH:MM UTC | task description (points to its runbook)
DAILY    14:00 | Run daily sync per runbooks/RUNBOOK_DAILY_SYNC.md
MON,THU  14:00 | Send notifications per runbooks/RUNBOOK_NOTIFICATIONS.md
```

### 5.5 Session Management

OpenClaw replays the full conversation history on every inference call. As session history grows, latency grows — not because of context size per se, but because of unbounded growth in the history file.

**Prefill vs. generation** — a nuance worth understanding: LLM generation (output tokens) runs sequentially at ~50 tps on a capable local GPU. Prefill (processing the input — system prompt + history) runs in parallel across all input tokens and is much faster: 10,000 tokens prefill in ~1–3 seconds. Session history is the real latency risk because it grows without a hard cap, while the system prompt is bounded at 150K characters.

**Session management scripts** (run via cron on the host):
```
*/5  * * * *  monitor-sessions.sh    # log session file sizes every 5 min
0 3  * * *    reset-sessions.sh      # archive and truncate sessions >512KB daily
*/30 * * * *  seed-sessions.sh       # restore missing heartbeat session files
```

### 5.6 The Full Host-Side Crontab

Five deterministic scripts constitute the entire automation layer outside the LLM:

```
TZ=America/Chicago
*/5  * * * *  check-todos.sh           # scheduling engine — promotes READY items
*/5  * * * *  monitor-sessions.sh      # session size logging
*/30 * * * *  seed-sessions.sh         # heartbeat session file recovery
0    3 * * *  reset-sessions.sh        # daily archive and truncate
*/30 * * * *  send-approved-emails.sh  # send human-approved outbox emails
```

All run on the **host** as the deploying user — not inside any container. Nothing in this crontab requires the LLM to be running or responsive.

---

## Module 6 — The Scaffolding: Runbooks, Scripts, and Cron *(10 min)*

### 6.1 The Three-Layer Pattern Revisited

The scaffolding from §2.3 is now concrete. Each layer has a specific role and a specific location:

| Layer | Location | Role |
|---|---|---|
| Cron | Host crontab | Decides when. Promotes READY items. Sends approved outputs. |
| Scripts | `scripts/*.py`, `scripts/*.sh` | Does deterministic work: fetch, transform, count, route, send API calls. |
| Runbooks | `agent/runbooks/RUNBOOK_*.md` | Tells the agent which scripts to call, in what order, with what arguments, and what to do with the output. |

The LLM never decides when. The cron scripts never decide what to say. Scripts never decide what action to take — they execute the action deterministically and return results.

### 6.2 Runbooks: Procedures the Agent Follows

A runbook is a markdown file in the agent's `runbooks/` directory. It gives step-by-step instructions for a specific recurring task and is read **fresh from disk on every trigger** — not held in session memory. This makes runbooks the most reliable vehicle for procedural changes: edit the runbook, and the very next trigger follows the new procedure exactly.

**The three-file pattern for any new recurring duty:**
1. One line in `IDENTITY.md` — what the duty is and its trigger
2. One runbook file `runbooks/RUNBOOK_X.md` — how to execute it step by step
3. One line in `CALENDAR.md` — when to trigger it (read by bash, not the LLM)

**Example: RUNBOOK_EMAIL_DIGEST.md**
- Triggered: daily at 08:00 AM Central (via CALENDAR.md → check-todos.sh → HEARTBEAT.md)
- Step 1: Read `PATHS.md` to confirm all file paths
- Step 2: Run `scripts/gmail_api.py --query "in:inbox after:yesterday"` → returns JSON list of messages
- Step 3: Run `scripts/contacts_api.py --lookup` on each sender → identifies known contacts
- Step 4: LLM composes digest from the results, formats for email
- Step 5: Write draft to outbox (pending approval) or send directly per standing authorization

Steps 1–3 are deterministic. Step 4 is where the LLM earns its keep. Step 5 follows the outbox pattern.

### 6.3 Scripts: Deterministic Tools for the Agent

Scripts are registered tools — the agent calls them via `exec` inside the sandbox, reads the output, and acts on it. They are invoked by runbooks, not by HEARTBEAT.md directly.

Key properties of well-designed agent scripts:
- **No LLM dependency** — the script does not call the model. It fetches, transforms, counts, or routes.
- **Structured output** — returns JSON the LLM can reliably parse
- **No third-party dependencies if avoidable** — `gmail_api.py` uses Python stdlib + `urllib` only; no pip install in the sandbox
- **Single responsibility** — one script, one job. The LLM composes multiple script calls; scripts do not compose each other.

**Example scripts from the worked example:**
- `gmail_api.py` — search and fetch email via Gmail API; stdlib only
- `contacts_api.py` — search and create contacts via Google People API; stdlib only
- `notify.py` — compose and queue outbox email; calls LLM API directly for composition

---

## Module 7 — Integrations: Slack and Google *(8 min)*

### 7.1 Slack

**Setup requirements:**
- Slack app at api.slack.com/apps with Socket Mode enabled
- Bot scopes: `channels:history`, `chat:write`, `users:read`, `groups:history` (do not add `assistant:write`)
- Bot token and app token added to `openclaw.json` under `channels.slack`
- CHANNELS.md updated with the channel ID → agent mapping

**Routing multiple agents:** When you have more than one agent, OpenClaw uses `bindings` in `openclaw.json` to route specific Slack channels to specific agents. One agent is marked `"default": true` and handles all DMs and unrouted messages. CHANNELS.md tracks the mapping in a human-readable form so you are not hunting through JSON when you forget which channel ID is which.

**The outbound post problem:** Agents can reply within active Slack sessions. They cannot initiate a post to a channel when no session is active (e.g., a scheduled overnight summary). The **slack-outbox pattern** handles this identically to email:
- Agent writes JSON to `shared/slack-outbox/` (channel, text, status: "pending")
- `send-slack-posts.sh` (cron, every 5 minutes) posts via `chat.postMessage` and archives

### 7.2 Google — Two Integration Paths

We use two different tools to reach Google services, for specific reasons:

| Tool | Best for | Why |
|---|---|---|
| `gsuite-mcp` | Gmail, Google Contacts | OAuth browser flow; writes `token.json`; works well with `messages.list` API |
| `gog` CLI | Google Sheets, Docs, Drive | Better for bulk read/write operations on structured documents |

**Why not one tool for everything?** We found that `gog` uses `threads.list` for Gmail, which returns only the first message of a thread and ignores `in:sent` filters — making it unsuitable for inbox and sent-mail workflows. `gsuite-mcp` handles OAuth and token refresh cleanly; our scripts (`gmail_api.py`, `contacts_api.py`) read the token it writes and call the Gmail API directly via `urllib`. For Drive and Sheets, `gog` is more capable and is the better fit. If `gsuite-mcp` gains full Drive/Sheets support in the future, consolidating to one tool would be worth revisiting.

**Credential handling in the sandbox:** OAuth tokens and credential files must be bind-mounted into the agent's sandbox container. Scripts must be mounted read-only; token files must be writable (for refresh). These mounts are configured in `openclaw.json` per agent.

**Google Cloud setup prerequisites** (students do this before the lab):
- Create a Google Cloud project
- Enable the APIs you need: Gmail, People, Drive, Sheets as applicable
- Configure OAuth consent screen and download credentials JSON
- Run `gsuite-mcp auth login` (browser flow, one time) to generate `token.json`

---

## Module 8 — Lessons Learned *(5 min)*

These are not setup checklists or incident reports. Each is a general principle extracted from a specific problem — the kind of lesson that applies beyond the exact incident that generated it.

---

**Lesson 1: Agents need a single source of truth for every fact**

The general principle: any fact, rule, or path that appears in more than one file will eventually diverge. The agent sees all 8 system-prompt files simultaneously and will silently blend contradictory signals into inconsistent, unpredictable behavior.

How it manifests:
- A behavioral rule stated in SOUL.md is also stated, slightly differently, in IDENTITY.md. You refine one and forget the other. The agent begins to behave as if there is a situational exception — because the two versions imply one.
- A file path is hard-coded in TOOLS.md, a runbook, and HEARTBEAT.md. You rename the scripts directory. You update two of the three. The third silently fails on every trigger until you notice.

The fix: establish canonical ownership for every category of fact. PATHS.md owns all file paths — every other file references it. SOUL.md owns behavioral invariants — IDENTITY.md lists duties but defers to SOUL.md for rules. Write each fact once; reference it everywhere else.

---

**Lesson 2: Agents have no clock and no memory of having acted**

The general principle: a language model has no internal sense of time passing and no persistent memory across session resets. Any scheduling or recurrence logic embedded in the agent's identity files will be checked on every heartbeat, burning tokens on hundreds of no-ops, and will fail silently after a session reset wipes the agent's memory of when it last acted.

How it manifests:
- HEARTBEAT.md instructs the agent to "check if it's Monday and, if so, send the weekly report." The agent faithfully evaluates this condition 672 times a week, costs tokens on 671 non-Monday heartbeats, and has no reliable way to know whether it already sent the report earlier that day.
- After a session reset, the agent has no memory of tasks it completed in the previous session. A recurring task scheduled via TODO.md disappears permanently unless it was also added to CALENDAR.md.

The fix: shell owns the clock. CALENDAR.md defines recurring duties in a format that `check-todos.sh` reads and acts on deterministically. TODO.md is for one-shot tasks only. The agent is invoked only when work is ready — it never decides when to act.

---

**Lesson 3: Session history can override freshly-written instructions**

The general principle: the LLM does not treat system-prompt instructions and session-history examples as separate inputs with different authority. When examples of old behavior in the session history outnumber or contradict new instructions in the .md files, the examples often win. Updating a file on disk does not immediately change agent behavior.

How it manifests:
- You add "always queue email for approval before sending" to SOUL.md. The session history contains 15 examples of the agent sending email directly. The agent continues to send directly — the instruction is outvoted.
- You update IDENTITY.md to change the Slack output format. The agent's next several responses use the old format. The new format appears only after a session reset.

The fix: any behavioral change requires four steps in sequence — (1) edit the right file, (2) immediately reset the session history, (3) tell the agent explicitly in the first message of the fresh session, (4) verify the next execution. Skipping step 2 is the most common error.

---

**Lesson 4: Agents will try to "help" unless you explicitly bound what help means**

The general principle: language models are trained to be helpful and will extend that helpfulness beyond the scope you intended unless you define boundaries explicitly. When debugging a problem, the agent will not just diagnose — it will also propose fixes, and if it has the tools, it will execute them on things you did not ask it to touch.

How it manifests:
- You ask the agent to help diagnose a Slack message formatting issue. It reads `openclaw.json`, concludes the channel configuration is wrong, and proposes to update it. One poorly-worded response later and config writes have been attempted on the gateway — potentially crashing it.
- You ask the agent to review a failing cron script. It fixes the script (correctly), then refactors two adjacent scripts "while it's at it," introducing a subtle bug.

The fix: disable `commands.config` writes immediately after onboarding — before connecting any external channel. Put explicit "diagnose and report; do not attempt fixes without explicit instruction" rules in SOUL.md for infrastructure domains. Run all execution in sandbox so the blast radius of enthusiastic self-correction is contained.

---

**Lesson 5: The outbox is not optional for external communications**

The general principle: the first time an agent sends something on your behalf that you did not intend, you understand viscerally why the outbox pattern exists. Every external communication with real-world consequences — email sent, Slack post published, API mutation made — should go through a queue, human review, and deterministic execution.

How it manifests:
- An agent drafts and sends a "helpful" follow-up email to a contact who appeared in your inbox in an ambiguous context. The email is well-written and completely wrong for the situation. It cannot be unsent.
- A scheduled overnight summary task runs, and the agent's interpretation of "summarize" is slightly off. The result goes directly to a team Slack channel at 6am. Forty people see it before you wake up.

The fix: agent writes to `outbox/` with status `pending`. Nothing sends until status is set to `approved` — by a human, or by a designated supervisor agent operating under its own SOUL.md constraints. A deterministic cron script sends approved items and archives everything. The audit trail in `sent/` and `rejected/` gives you a complete record.

---

**Lesson 6: Sandbox is structural, not optional hardening**

The general principle: running agent code inside an ephemeral sandbox container is not a cautious choice for worried operators — it is the architectural guarantee that agent execution cannot affect the host filesystem, persist state between runs, or reach network resources you have not explicitly authorized. It should be enabled before the agent is given any tools. The research community has reached the same conclusion: Shankar et al. (2025) identify containment and runtime monitoring as essential requirements — not optional hardening — for any agent system with tool access. [*cite: "Agentic AI Security: Threats, Defenses, Evaluation, and Open Challenges," arXiv 2510.23883, 2025*]

How it manifests:
- Gateway-exec mode (the alternative to sandbox) runs agent commands directly on the host as your user. A script with a path bug writes to the wrong location on your host filesystem. There is no container boundary to catch it.
- A sandbox container misconfigured with an overly broad network bind-mount can reach services beyond its intended scope. Catching this early with iptables DOCKER-USER rules prevents the misconfiguration from becoming an incident.

The fix: set `sandbox.mode = "all"` in `openclaw.json` before connecting any external service. Add iptables DOCKER-USER rules to block sandbox containers from reaching your LAN, Tailscale network, or SSH. Verify with the four-command security checklist before calling the platform ready.

---

## What's Next: The Hands-On Lab

The hands-on session (developed separately as its own document in this repository) uses the materials in this repo directly — no external clones required. The tutorial repo will be organized as:

```
OpenClaw-Tutorial/
├── gateway/          # platform configuration (docker-compose, config.yaml, scripts)
│                     # derived from spark-ai; updated and parameterized for tutorial use
└── agents/           # agent workspace template
                      # derived from OpenClaw-Gmail; ready to parameterize and deploy
```

Students clone `OpenClaw-Tutorial`, configure their model path, and work entirely within this single repo. The lecture you just completed provides the full conceptual foundation — the lab is execution.

Students will:
1. Clone the tutorial repo; choose local or cloud model path; configure `config.yaml` and `secrets.yaml`; run `apply-config.sh`
2. Apply security hardening and verify the four-command checklist
3. Complete Google Cloud and OAuth setup (gsuite-mcp) — prerequisite steps done before lab day
4. Set up the Slack app and configure channel routing
5. Parameterize the agent workspace (one substitution pass through `agents/`)
6. Install the host-side crontab and observe `check-todos.sh` promote a calendar entry to READY
7. Practice the behavioral change lock-in procedure: edit → reset → tell → verify
8. Walk through an outbox approval cycle end-to-end

*The hands-on lab will be developed as a separate document (`HANDS-ON.md`) in this repository, structured for independent self-paced use or instructor-led delivery. The `gateway/` and `agents/` directories will be populated by harvesting and updating the relevant files from the reference repositories used to develop this curriculum.*

---

## Appendix: Key Files Quick Reference

| File | Location | Auto-loaded? | Purpose |
|---|---|---|---|
| `SOUL.md` | `<agent>/` | Yes | Hard behavioral invariants |
| `IDENTITY.md` | `<agent>/` | Yes | Who the agent is; duty list |
| `TOOLS.md` | `<agent>/` | Yes | How to call scripts and tools |
| `USER.md` | `<agent>/` | Yes | User context and preferences |
| `HEARTBEAT.md` | `<agent>/` | Yes | Always-on reflexes |
| `BOOTSTRAP.md` | `<agent>/` | Yes | First-run orientation |
| `MEMORY.md` | `<agent>/` | Yes | Long-term accumulated knowledge |
| `AGENTS.md` | `<agent>/` | Yes | Sub-agent definitions |
| `PATHS.md` | `<agent>/` | No — read explicitly | Canonical path registry (single source of truth) |
| `CHANNELS.md` | `<agent>/` | No — read explicitly | Slack channel → agent routing map |
| `CALENDAR.md` | `<agent>/` | No — read by bash | Recurring duty schedule |
| `TODO.md` | `<agent>/` | No — read by agent | One-shot deferred tasks |
| `RUNBOOK_X.md` | `<agent>/runbooks/` | No — read at trigger | Step-by-step task procedures |
| `config.yaml` | `gateway/` | N/A | Per-agent model assignments |
| `secrets.yaml` | `gateway/` | N/A | API keys (gitignored) |
| `check-todos.sh` | `agents/scripts/` | N/A | Bash scheduling engine (cron) |
| `send-approved-emails.sh` | `agents/scripts/` | N/A | Sends human-approved outbox emails |
| `reset-sessions.sh` | `agents/scripts/` | N/A | Daily session archive and truncate |

---

*Outline version: 2026-03-18 rev 4.*

---

## References

[1] Zverev et al., "Can LLMs Separate Instructions From Data? — Formalizing and Testing Instruction Hierarchy," ICLR 2025. *(Cited in §2.1 — prompt injection as a structural vulnerability in current LLM architectures)*

[2] Masterman, T.; Besen, S.; Sawtell, M.; and Chao, A., "The Landscape of Emerging AI Agent Architectures for Reasoning, Planning, and Tool Calling: A Survey," arXiv:2404.11584, 2024. *(Cited in §2.3 — reliability-critical applications systematically require more deterministic orchestration)*

[3] Hong, S.; Zhuge, M.; et al., "MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework," ICLR 2024 (oral). arXiv:2308.00352. *(Cited in §2.3 — SOPs encoded in code reduce cascading hallucinations from naively chained LLMs)*

[4] Qiu, L.; Ye, Y.; Gao, Z.; et al., "Blueprint First, Model Second: A Framework for Deterministic LLM Workflow," arXiv:2508.02721, 2025. *(Cited in §2.3 — deterministic orchestration engine + bounded LLM sub-tasks = verifiable, auditable agent behavior)*

[5] Kapoor et al., "The 2025 AI Agent Index: Documenting Technical and Safety Features of Deployed Agentic AI Systems," MIT / arXiv:2602.17753, 2025. *(Cited in §2.4 — majority of deployed agents document no sandboxing or containment mechanisms)*

[6] Shankar et al., "Agentic AI Security: Threats, Defenses, Evaluation, and Open Challenges," arXiv:2510.23883, 2025. *(Cited in Lesson 6 — containment and runtime monitoring are essential requirements, not optional hardening)*

[7] Yomtov, O. (Koi Security), "ClawHavoc: Large-Scale Poisoning Campaign Targeting the OpenClaw Skill Market," February 2026. Covered by Trend Micro, CyberPress, SecurityWeek, and others. *(Cited in §1.3 and §2.1 — real-world consequence of missing supply-chain discipline: ~20% of ClawHub registry compromised)*

[8] SecurityScorecard STRIKE Team, "ClawJacked: WebSocket Hijack Vulnerability in OpenClaw Gateways Exposed to Internet," February 2026 (CVE-2026-25253). Covered by The Hacker News. *(Cited in §1.3 and §2.4 — 135,000+ publicly exposed instances; 15,000+ vulnerable to RCE; Tailscale binding is the direct mitigation)*

*[TODO: Hugo — locate the specific talk or paper making the quantitative argument for code/LLM separation in agent scheduling. First name Hugo, context likely a 2024 major ML conference (NeurIPS or ICML). Confirm last name and add citation in §2.3. Note: not among the references in Qiu et al. 2025 — may be a conference talk rather than a paper.]*
