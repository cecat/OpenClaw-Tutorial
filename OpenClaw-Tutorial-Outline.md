# Getting Started with OpenClaw: A Safety-First Approach
*Revised organization*


## Contents

| | Title |
|---|---|
| **Section 1: OpenClaw** | |
| Module 1 | What Is OpenClaw and Why This Approach? |
| Module 2 | The Design Charter |
| Module 3 | Agent Identity: The Workspace and the Sacred-8 Files |
| **Section 2: Enhancements** | |
| Enhancement 1 | Security, Safety, and Containment as Design Considerations |
| Enhancement 2 | Ease of Management: config.yaml and apply-config.sh |
| Enhancement 3 | Scheduling: The Shell Owns the Clock |
| Enhancement 4 | Separation of Responsibilities: Runbooks, Scripts, and Cron |
| Enhancement 5 | Multi-layer Oversight: The Outbox and Review Pattern |
| **Section 3: Integrations** | |
| | Slack and Google |
| **Appendix** | Lessons Learned |



---

# SECTION 1: OpenClaw

*This section introduces OpenClaw — what it is, the design principles that guide our deployment, and how agents are configured and maintained.*

## Module 1 — What Is OpenClaw and Why This Approach?

### 1.1 What OpenClaw Is (and Isn't)
- OpenClaw is an **agent framework** that provides persistent memory, tools, and communication channels to multiple agents, each implemented via a model
- Open source, self-hosted: you own the configuration, credentials, history, and behavior
- Enables each agent to use a different model, including local models (e.g., via vLLM) and models provided via cloud APIs (e.g., Anthropic Claude, OpenAI, Google)
- Communicates through multiple optional channels including Slack, a web dashboard, Telegram, and dozens of other channels
- For each agent, the model provides cognition; OpenClaw provides orchestration, file access, tool execution, memory, and limited native scheduling (a periodic heartbeat that fires every N minutes — more on the limitations of this in Module 5)

### 1.2 What an Agent Actually Does (that a chatbot doesn't)
- A chatbot waits for a question and answers it. An agent acts between conversations — on a schedule, in response to events, and on its own judgment about what to do next.
- An agent can process and summarize (and send email), post to Slack, read your calendar, modify files, and call APIs — autonomously, while you sleep.
- This is genuinely useful. It is also qualitatively different from a chatbot, and must be designed accordingly.
- OpenClaw agents can be partially autonomous without being fully autonomous — calibrating that balance deliberately is an important facet of this tutorial.

### 1.3 Five Areas Where We Enhanced OpenClaw

OpenClaw provides the runtime and the channel integrations — but out of the box it makes no security decisions on your behalf, its scheduling is limited to a blunt periodic heartbeat, and its configuration lives in a large JSON file. We found five areas where deliberate design decisions significantly improve reliability, safety, and maintainability. These are covered in depth in Section 2 — here is the brief version.

**Security, safety, and containment.** OpenClaw can be deployed and run without any of the network isolation, sandboxing, or behavioral guardrails we describe. The platform does not enforce them by default. We chose to implement them deliberately, drawing on a documented threat model — and real-world incidents in the OpenClaw ecosystem — that make the case for treating security as a design consideration from day one, not a retrofit. *(Section 2, Enhancement 1)*

**Scheduling.** Out of the box, OpenClaw supports three types of triggers: incoming messages (a Slack DM, channel message, or web dashboard conversation), the heartbeat (a periodic timer — the only proactive trigger native to OpenClaw), and channel-specific events like Slack reactions. The heartbeat is blunt: it fires on every interval whether or not there is work to do, and the agent must reason about what (if anything) to do each time. With a 15-minute interval, that is 672 heartbeats per week for a task that fires once. We built a scheduling layer — a bash script, a CALENDAR.md file, and a TODO.md queue — that moves the scheduling decision entirely out of the LLM and into deterministic code. *(Section 2, Enhancement 3)*

**Separation of responsibilities.** Models are excellent at judgment, composition, and natural language understanding. They are a poor fit for deterministic operations: fetching structured data from an API, counting rows, routing files, or deciding whether the current time matches a schedule. Mixing these concerns produces non-deterministic behavior and wastes inference budget. We separate them explicitly: code handles anything procedural, the LLM handles anything requiring judgment. Runbooks and scripts are the mechanism. *(Section 2, Enhancement 4)*

**Layered oversight.** Any agent action with real-world consequences — sending email, posting to a channel, modifying external records — should go through a review step before execution. We implement this via an outbox/approval/send pattern: the agent queues a draft, a reviewer (another agent or a human) approves or rejects, and a deterministic cron script sends approved items. The goal is to start with review enabled everywhere and progressively delegate that review as confidence in agent behavior grows. *(Section 2, Enhancement 5)*

**Configuration scaffolding.** OpenClaw stores its full configuration — agent model assignments, Slack channel bindings, sandbox settings, and more — in a single large JSON file (`openclaw.json`) inside the running container. Editing it directly through the dashboard's small JSON editor is error-prone and tedious. We built `config.yaml` and `apply-config.sh` as a clean abstraction: human-readable YAML captures the assignments you care about, a script patches the live JSON and restarts the gateway. We expect this list of managed settings to grow over time as we find other configuration values worth exposing this way. *(Section 2, Enhancement 2)*

### 1.4 Safety as a Design Principle
- Security and containment have historically been afterthoughts with exciting new technology — something addressed after adoption, once incidents accumulate, rather than before. OpenClaw is no exception to this pattern. The platform is powerful, the demos are compelling, and the natural instinct is to get something running and worry about hardening later. Rapid adoption amplifies this risk: Jensen Huang remarked at GTC in March 2026 that OpenClaw is the fastest-adopted open source project of all time. [*verify and cite*] The faster a platform spreads, the faster it becomes a target.
- The consequences are already well-documented. Module 2 presents a dedicated slide on real incidents — exposed instances, a skills registry poisoning campaign, and a remotely-exploitable vulnerability in improperly-bound gateways — all traceable to skipped hardening steps. The details are in Module 2; the point here is that they are not hypothetical.
- We cover safety, containment, and alignment in Module 2 — before setup, before configuration, before anything else — because the decisions you make in the first hour determine the blast radius of everything that goes wrong later. Understanding the risks is part of understanding the platform.
- **The right mental model:** your agent acts in your name. Every email it sends is from you. Every Slack post is from you. Design accordingly — not by neutering the agent, but by designing the right boundaries. *(Module 2 covers this in full.)*

### 1.5 Architecture Summarized

An OpenClaw deployment consists of one or more large language models (LLMs) and a gateway, connected by a Docker internal network that is not exposed to the outside world. Each agent can be backed by a different model — the same gateway can run one agent on a local GPU and another on a cloud API — allowing you to match model capability and cost to each agent's actual needs. The gateway manages agent identities, conversation sessions, tool execution, and channel integrations (Slack, web dashboard, and others). Agent personality, memory, and task procedures live in plain markdown files in a workspace directory on the host — readable, auditable, and well-suited to version control.

OpenClaw provides the gateway, the agent definition (characteristics, tasks, etc.) and execution environment, the sandbox, and many skills such as channel integrations (Slack, etc.). Using these capabilities, one can create multiple agents, each with a unique set of responsibilities, styles of interaction, and even cost/performance trade-offs where agents with simpler duties use lower-cost models and those with more important duties (e.g., reviewing the work of other agents) use frontier AI models.

Implementers are also responsible for developing their workflows, managing authentication with external services, and making decisions such as regarding routing of messages between an external service (e.g., Slack), and the agents that use that service. Implementers must also develop procedures that agents will follow, such as an email outbox/approval workflow or reporting data to groups via a Slack channel.

We also found two areas where a small amount of additional infrastructure significantly improves the deployment. The first is scheduling (the heartbeat gap described in §1.3). The second is a mechanism to provide LLMs with deterministic tools: scripts that perform procedural tasks such as fetching email from the Gmail API, syncing rows between Google Sheets, or queuing a draft for approval. Both mechanisms follow the same design philosophy: keep the LLM doing what it does best, and use code for everything else. The third addition — layered oversight — builds on both. These are described in Modules 5, 6, and 7.

A central element of containment is the **sandbox**. When an agent executes a script or shell command, OpenClaw spawns an ephemeral Docker container specifically for that execution — separate from the gateway container — and discards it when the command completes. This is standard OpenClaw behavior (configurable via `sandbox.mode` in `openclaw.json`); we enable it globally and layer iptables rules on top. Those rules block outbound connections from sandbox containers to your local network (LAN) and to the Tailscale virtual address range (100.64.0.0/10), preventing a compromised or misbehaving script from pivoting to other machines on your infrastructure. TCP port 22 (SSH) is blocked regardless of destination as an additional protection. Internet-bound connections — needed for calling Google APIs, the Anthropic API, and similar services — are permitted. The gateway itself is bound only to the host's Tailscale interface (not `0.0.0.0`), so the dashboard and agent API are reachable only from devices enrolled in your Tailscale network. (Tailscale is a prerequisite; see [tailscale.com/download](https://tailscale.com/download).)

*The full architecture diagram and setup walkthrough are in Enhancement 1.*

---

## Module 2 — The Design Charter

> **This module is the foundation.** Every architecture decision in Sections 2 and 3 — sandboxing, iptables rules, the outbox pattern, SOUL.md, the scheduling design, the scaffolding — exists because of the principles established here. We cover it second, not last, because you should understand *why* before you are asked to execute *what*.

### The Five Principles (§2.1–2.5)

Five principles drive every design and implementation decision in this deployment. Each is explained in the subsections that follow.

1. **Know the threat model** — understand the attack surface before building defenses; the threats are not hypothetical. *(§2.1)*
2. **Alignment** — encode behavioral invariants structurally before connecting any tools, and reinforce them with mechanical constraints that do not depend on the LLM reasoning correctly. *(§2.2)*
3. **Separation of responsibilities** — anything deterministic or procedural is implemented in code; anything requiring judgment, creativity, or language is handled by the model. *(§2.3)*
4. **Containment** — network, execution, filesystem, and config isolation are in place before anything external is connected. *(§2.4)*
5. **Trust but Verify** — any action with real-world consequences goes through a review layer; the reviewer and criteria evolve as confidence grows. *(§2.5)*

---

### 2.1 The Threat Model

These five risk categories apply to any autonomous agent system with tool access and external connectivity — they are not specific to OpenClaw. We considered each explicitly when designing our deployment.

1. **Prompt injection** — malicious content in email, files, or Slack instructs the agent to execute unintended commands. The agent cannot distinguish a legitimate instruction from one embedded in adversarial input unless the system is designed to limit what it can do with that input. This is a structural vulnerability: current LLMs do not enforce separation between instructions they are meant to execute and data they are meant to process. [*cite: Zverev et al., "Can LLMs Separate Instructions From Data?", ICLR 2025*]

2. **Lateral movement** — a sandbox container that can reach your LAN, other Tailscale nodes, or SSH services can be used as a pivot point. This is especially relevant when the agent's sandbox runs on a machine that also has access to other internal services.

3. **Runaway external actions** — the agent takes actions with real-world consequences without human review: sending email, posting to public channels, modifying files, or changing configuration. A mistake (such as deleting files or changing configs) by an agent is not intentionally malicious, but is the model trying to be helpful, where "helpful" was not bounded precisely enough. A common example: ask the agent to diagnose a problem and it will often try to fix it, touching things you did not ask it to touch. *(This is Lesson 4 in the Appendix.)*

4. **Credential exfiltration** — API keys, OAuth tokens, and other secrets passed to the sandbox can leak via outbound HTTP if the sandbox has unrestricted network access.

5. **Supply chain / skills registry poisoning** — third-party skills installed from a public registry (ClawHub) may contain malicious code. The ClawHavoc campaign (January–February 2026) poisoned approximately 20% of ClawHub's registry with skills delivering credential stealers and backdoors; the only requirement to publish had been a GitHub account one week old, with no code review or signing. Any skill installed from a registry is code that runs inside your sandbox with the access you have granted it. Treat it accordingly.

### 2.2 Alignment

Alignment is ensuring that a system's goals, values, methods, and behavior remain compatible with human intentions — not just in the literal task at hand, but across the full range of situations the system may encounter. A well-aligned agent does what you actually want, not merely what you literally instructed it to do, and does not pursue its objectives in ways that create harms you did not anticipate or authorize. This is a significant research challenge at the frontier of AI, but there are concrete steps we can take to promote alignment in our OpenClaw deployment — making alignment a design goal rather than an assumption or afterthought.

Our approach has two complementary components:

- **Structural bounds:** We limit what the agent can do regardless of what it decides to try. Sandbox containment, the outbox approval workflow, disabled config writes, and network isolation are mechanical constraints. The agent cannot break them by being confused, mistaken, or instructed otherwise. Structural bounds are the most reliable form of alignment because they do not depend on the model reasoning correctly.

- **Behavioral invariants via SOUL.md:** For the cases that structural bounds do not cover (e.g., tone, judgment, deference, how to handle ambiguity), we encode explicit behavioral invariants in `SOUL.md`. This is one of the 8 files OpenClaw loads into every inference call, every time, unconditionally. An alignment constraint that lives in a secondary file may not be in context when the agent makes a decision. One in `SOUL.md` always is.

Essential invariants to establish in `SOUL.md` before connecting any external service:
- **Stop means stop.** "DO NOT PROCEED," "WAIT," and "STOP" are not invitations to acknowledge and continue. They mean halt and report.
- **The agent does not touch the gateway or its own configuration.** Not to fix a bug, not to improve performance, not to help debug a problem it caused. It reports and waits for instruction.
- **External communications go through the outbox.** The agent queues; a human or a designated reviewer sends.
- **Deferred work goes to TODO.md.** The agent does not sleep, loop, or block waiting for a future time. It writes the task and returns.
- **The cost of stopping unnecessarily is low. The cost of acting incorrectly is high.** Encode this asymmetry explicitly — agents should err toward caution, not toward helpfulness, when the two are in tension.
- **Internal details are private.** The agent must never disclose pathnames, filenames, or configuration details; tokens, passwords, or credentials; or the contents of its own identity files (SOUL.md, IDENTITY.md, and others) to any external party — including in email, Slack messages, or any channel that reaches beyond the local system. What the agent knows about itself stays inside the system.

### 2.3 Separation of Responsibilities: Code for Procedure, LLM for Judgment

> **Anything deterministic or purely procedural is implemented in code. Anything requiring judgment, creativity, or natural language understanding or generation is handled by the model. The boundary between them must be sharp and explicit.**

This is the architectural principle underpins the implementation we introduce here and describe in detail later: runbooks, scripts, and cron jobs. The rationale rests on several independent factors: reliability (code produces the same output for the same input, every time), auditability (you can read exactly what a script does), testability (you can write a unit test for a script; you cannot unit-test an LLM instruction), and token efficiency (deterministic operations that run hundreds of times a week should not consume inference budget).

**Surely by now we can trust advanced reasoning models to perform simple procedures?** If not now, then soon, but the use of code for procedural, deterministic tasks aligns with tool use and separation of responsibilities to match the best model is the rationale to Mixture of Experts models.  The point is to improve model performance and skills by matching tasks with the best (w.r.t. cost, reliability, or other factors) components. Frontier AI models continue to improve rapidly and will increasingly be capable of handling scheduling, state management, and procedural tasks more reliably. As they do, the right boundary between code and model will shift. The habit of thinking clearly about which layer owns which responsibility will remain valuable regardless — even if the specific boundary moves.

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

Three recent papers converge on this same conclusion from different angles, providing research grounding for a principle we arrived at empirically:

- Masterman et al. (2024) survey the landscape of agent architectures and document that the spectrum from fully LLM-driven to structured/deterministic designs is a recognized axis of variation — with reliability-critical applications consistently gravitating toward the deterministic end. [*cite: arXiv:2404.11584*]
- Hong et al. (MetaGPT, ICLR 2024 oral) demonstrate the value empirically: encoding Standard Operating Procedures in code and assigning deterministic roles to agents dramatically reduces cascading hallucinations compared to naively chaining LLMs. [*cite: arXiv:2308.00352*]
- Qiu et al. (Blueprint First, Model Second, 2025) make the architectural case most directly: a deterministic orchestration engine manages workflow structure while the LLM handles bounded sub-tasks — producing verifiable, auditable behavior. [*cite: arXiv:2508.02721*]

The token efficiency argument is quantified concretely in Module 5 with a table showing waste rates for common scheduling patterns.


**Implementation here is a layered approach:**

The LLM sits atop three layers of deterministic functionality. Each layer calls down only: the LLM invokes runbooks, runbooks invoke scripts, scripts are triggered by cron. Information flows back up; control flows down.

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
| Network | Tailscale-only gateway binding (not `0.0.0.0`) | Dashboard reachable only from your Tailscale-enrolled devices; also prevents WebSocket hijack from malicious websites. Dashboard access additionally requires a pairing token introduced in recent OpenClaw versions. See ["ClawJacked" — The Hacker News](https://thehackernews.com/2026/02/clawjacked-flaw-lets-malicious-sites.html) for the real-world exploit (CVE-2026-25253) that targets gateways bound to `0.0.0.0`. |
| Network | iptables DOCKER-USER rules | Sandbox containers cannot make outbound connections to your local network (LAN) or to Tailscale nodes (100.64.0.0/10). Connections to the internet (e.g., Google APIs, Anthropic API) are permitted — agents need them. SSH connections (port 22) are additionally blocked regardless of destination. |
| Execution | Sandbox mode: `mode: all` | Agent commands run in throwaway containers, not on the host |
| Execution | No `docker.sock` in sandbox | No container escape to host Docker daemon |
| Config | Disable `commands.config` writes | Agent cannot modify gateway configuration via chat |
| Filesystem | docker-compose bind mounts (explicit, narrow) | Agent's sandbox can only access the specific directories you mount — it cannot reach the rest of your host filesystem, config files, or other services |
| Credentials | Separate dedicated service accounts | If a credential is compromised, the attacker's access is limited to that account's permissions on that one service — they cannot pivot to your personal email, other accounts, or other systems. The "blast" is the damage; the "radius" is how far it can spread. Separate accounts keep the radius small. |

**A note on sandbox architecture and network isolation.** The OpenClaw gateway is a long-running Docker container. When the agent executes a script or shell command, OpenClaw spawns a separate, ephemeral Docker container for that execution — the sandbox — which is discarded when the command completes. The gateway and sandbox containers communicate over a Docker-internal bridge network.

The iptables rules operate at the IP layer (L3) on the host. They block outbound connections from Docker-subnet traffic (the containers) to two destination ranges: (1) your LAN subnet (e.g., 10.0.4.0/22), and (2) the Tailscale CGNAT virtual address range (100.64.0.0/10). A third rule blocks TCP port 22 (SSH) outbound regardless of destination IP. These rules prevent a script running in a sandbox from making TCP connections to other machines on your local network or to the virtual IPs assigned by Tailscale to your enrolled devices.

Critically, these rules do not block outbound connections to the public internet. Sandbox containers can reach external services — Google APIs, the Anthropic API, Slack webhooks — over standard HTTPS. That is intentional and necessary.

Tailscale operates as a WireGuard-based overlay network (L3 VPN). It assigns virtual IP addresses in the 100.64.0.0/10 CGNAT range to each enrolled device and creates a `tailscale0` virtual interface on the host. The OpenClaw gateway port is bound to the host's Tailscale IP address, not to 0.0.0.0. Traffic reaching that port therefore arrives only via the Tailscale interface — meaning only from devices enrolled in your Tailscale network. The iptables rules ensure that containers cannot forge connections to those virtual IPs and thereby bypass the network isolation. *(See FIXES.md for notes on further egress hardening if needed.)*

*CVE (Common Vulnerabilities and Exposures) identifiers are standardized labels assigned to publicly-disclosed security vulnerabilities, maintained by MITRE. Links to original coverage are provided for each CVE cited.*

There is a growing body of "how-to" material for deploying OpenClaw, and it is striking how little attention is paid to security and containment. A 2025 survey — [The 2025 AI Agent Index](https://arxiv.org/abs/2602.17753) (MIT) — found that the majority of deployed agentic AI systems document no sandboxing or containment mechanisms at all. We treat the layers above as non-negotiable precisely because the default is to omit them.

### 2.5 Trust but Verify: Layered Review

The long-term goal of a personal agent deployment is to remove yourself from routine decisions — not to keep you perpetually in the loop. But trust is earned, not assumed. We start with review at every consequential action and progressively delegate that review as we gain confidence that the agent's behavior aligns with the intent encoded in its descriptor files (more in §4).

The **outbox pattern** is the mechanism we use for review.  An agent writes a draft to queue in an outbox folder (shared among agents), a reviewer approves or rejects, and a cron script handles the actual sending (if approved). The reviewer is either another agent assigned to the task (with precise revivew criteria)  or a human.

```
Agent writes draft JSON → outbox/  (status: "pending")
Reviewer approves → updates JSON in-place (status: "approved", approved_at: <ts>)
Reviewer rejects  → updates JSON, moves file to rejected/ (includes rejected_reason)
Cron script sends approved drafts → moves to sent/; logs to shared/logs/email-send.log
```

Approved drafts move from `outbox/` to `sent/`. Rejected drafts are updated with `status: "rejected"` and a `rejected_reason` field, then moved to `rejected/` — the reason is written into the JSON file itself, not the log. The `shared/logs/email-send.log` records delivery confirmations only (timestamps, recipients, send status). A Slack DM notifies the operator of any rejection. The complete mechanism — JSON format, review criteria, directory structure, logging — is covered in Module 7.

In our deployment we have three agents: **main** (supervisor — reviews outbox drafts, handles administrative queries), **admin-agent** (clerical — tracks form submissions, produces reports, sends notifications via Slack and email), and **gmail-agent** (Gmail assistant — reads email, drafts replies, manages contacts). All examples in this outline and in the hands-on lab use this three-agent setup.

**The supervisory agent (reviewer) is a design choice, not a fixed role.** In our current deployment we use human review for some things and agent review for others:

- **Agent review** for outbound email from our clerical agent, whose routine reporting uses template-based messages with light natural language fills. The review criteria — appropriate language, no more than a handful of emails per day to any recipient, recipient must be in the contacts database — are defined in `EMAIL.md` and applied by the supervisory agent during its heartbeat. The supervisory agent approves or rejects each queued draft (fully logged for diagnostics).

- **Human review** for outbound email from our Gmail assist agent, which reads incoming messages, interprets them, and drafts a substantive reply. The judgment required is higher and the stakes of an error are greater. The human reviews manually while we build a track record and fine-tune the model's instructions regarding email handling..

As the Gmail agent's drafts prove consistently good over time, we will delegate review to a second agent — likely a smaller, cheaper reasoning model suited to the specific review task. This is also an economic design decision: a capable reasoning model for drafting, a lighter model for reviewing against known criteria, and human oversight only where neither suffices yet.

**The principle generalizes beyond email.** Any outbound action with real-world consequences — a Slack post to a public channel, an API call that modifies an external record — can use the same pattern. We have partially implemented this for Slack (a `slack-outbox/` and `slack-sent/` directory exist; a review step is not yet in place). Module 7 covers the full pattern, including how to extend it to other channels.

This is a living architecture, not a final answer. What we describe here is our current state; it will change as our agents earn more autonomy.

---

## Module 3 — Agent Identity: The Workspace and the Sacred-8 Files

### 3.1 The Workspace
An agent's workspace is a directory of plain markdown files mounted into the OpenClaw gateway container. Everything the agent knows about itself, its user, its tools, and its duties lives in these files — readable, auditable, editable in any text editor, and (we recomment) version-controlled in git.  For long-term debugging and tuning, it's also good practice to keep a CHANGELOG.md file (or have your vibe-coding companion do so).

OpenClaw automatically loads eight of these files into the system prompt on every inference call for the agent. Everything else (task-specific guidance, etc.) is invisible to the model unless the agent explicitly reads it with a tool call.  These eight files define the agent's identify, personality, style, and roles.

**The Sacred Eight: files OpenClaw auto-loads (in order):**

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

### 3.2 Recommendations on What Each File Should (and Should Not) Contain

*Content strategy — what goes where and why — is covered here. The risks of getting it wrong (duplicated facts, drifting instructions) are treated in §4.4.  In this repository each of these files contain initial boilerplate that you should review and customize.*  

**`SOUL.md`** describes the behavioral invariants that must hold unconditionally. If a rule must never be violated, it belongs here — not in a secondary file. Keep it focused: these are principles, not comprehensive rules for every conceivable situation. 

**`IDENTITY.md`** describes who the agent is — its name, its Slack behavior contract, and a concise list of its recurring duties. Duties are listed here (what and when), but the how belongs in runbooks (read separately, covered in Module 6). Do not duplicate rules from SOUL.md here — reference them.

**`TOOLS.md`** is the agent's tool manual. It explains how to call each script, what arguments it takes, and what it returns. When a new script is added to the workspace, TOOLS.md gets a new entry.

**`USER.md`** holds everything the agent needs to know about you: your background, preferences, communication style, and context. This file is what makes the agent feel personal rather than generic.

**`HEARTBEAT.md`** contains only always-on reflexes — things the agent does on every heartbeat for the lifetime of the agent. Check the TODO queue for READY items. Scan for rejected emails. Infrastructure-level reflexes only. Day-of-week logic, time-window checks, and task-specific procedures do not belong here.

**`BOOTSTRAP.md`** is the first thing a fresh agent reads. It walks the agent through establishing its identity — choosing a name, filling in USER.md, understanding its tools. Best practice: make this the subject of the very first message to a new agent, before anything else. After initial setup, BOOTSTRAP.md becomes dormant orientation material.

**`MEMORY.md`** is runtime knowledge promoted from daily memory files (`memory/YYYY-MM-DD.md`). The agent writes daily entries during conversations; important facts get promoted here for long-term retention. This is one of the two files the agent actively writes to (the other being `TODO.md`).

### 3.3 The Files That Are NOT Auto-Loaded — and Why That Matters

Several important files live in the workspace but are not auto-loaded: they reach the agent only when explicitly referenced in one of the Sacred Eight files, in a runbook or procedure, or when the agent is directly asked to read them.

**`PATHS.md`** — the canonical registry of every absolute path used by the agent: workspace root, shared directory, script locations, outbox paths. We find that path strings inevitably get duplicated across TOOLS.md, runbooks, and HEARTBEAT.md and drift out of sync as the workspace evolves. PATHS.md is the single source of truth; all other files reference it rather than hard-coding paths. The agent reads PATHS.md at the start of any task involving file I/O.

**`CHANNELS.md`** — a registry of which Slack channel maps to which agent, including channel IDs and routing rules. Needed as soon as you have more than one agent sharing a Slack workspace.

**`CALENDAR.md`** — the recurring duty schedule. Critically, this file is not read by the LLM to decide what to do — it is read by `check-todos.sh` (a bash script running on the host) to determine when to queue work for the agent. The separation is deliberate and central to the scheduling architecture in Module 5.

**`EMAIL.md`, templates, runbooks** — task-specific reference material read on demand. Because these are not auto-loaded, rules or paths that only appear in them may never reach the agent in a given session.

The implication: **anything the agent must do correctly and consistently must live in one of the 8 auto-loaded files.** Everything else is reference material.

### 3.4 Keeping Model Instructions Clean and Avoiding Confusion

One of the most common classes of agent configuration problem is the duplicated instruction. When the same rule, path, or fact appears in more than one file, they will eventually diverge — and the agent will silently blend contradictory signals into inconsistent behavior.

Examples of how this manifests:
- `SOUL.md` says "always get approval before sending email." `IDENTITY.md` has an older version of the same rule, phrased differently, with an implicit exception you added and forgot to add to SOUL.md. The agent interprets the combination as having a situational exception — sometimes it asks, sometimes it doesn't.
- A script path is hard-coded in `TOOLS.md`, `RUNBOOK_EMAIL_DIGEST.md`, and `HEARTBEAT.md`. The scripts directory is renamed. You update two of the three. The third silently fails on the next trigger.

The fix is architectural: **one authoritative home for each fact.** PATHS.md for paths. SOUL.md for behavioral invariants. IDENTITY.md for duties. Other files reference these — they do not repeat them.

### 3.5 How Reliably Does the Agent Follow the 8 Files?

More reliably than secondary files, but not unconditionally. This can be surprising, for instance if a template is changed and the agent uses the old template rather than the new one. There are two important nuances:

**Instructions vs. examples.** OpenClaw rebuilds the system prompt from the 8 files on every call — but it *also* sends the full session history (the accumulated back-and-forth since the last session reset). When instruction text and behavioral examples in the history conflict, examples often win. An agent with many examples of old behavior in its session history may not immediately adopt a newly-written rule, even one enshrined in SOUL.md.

**What this means for updates.** Updating a file on disk does not guarantee the agent immediately adopts the new behavior. Any behavioral change requires four steps: (1) edit the right file, (2) reset the session to clear counter-examples from history, (3) explicitly tell the agent about the change in the first message of the new session, (4) verify the next execution follows the new rule. `reset-session.sh` (covered in §4.6) automates step 2 and logs the reason for the reset.

**Reliability tiers:**

*Session management and daily session reset are covered in §4.6 immediately below.*

| Source | When agent sees it | Reliability |
|---|---|---|
| One of the 8 auto-loaded files | Every inference call | High — with session reset after changes |
| Non-auto-loaded file | Only when agent explicitly reads it | Indeterminate |
| A runbook (read at trigger time) | Fresh from disk on every trigger | Most reliable for procedural changes |
| Session history examples | Every call until session reset | Works against you after behavioral changes |


### 3.6 Session Management and Daily Reset

A **session** in OpenClaw is a conversation history file (`.jsonl`) stored in the gateway's persistent Docker volume. Each agent maintains its own session; different Slack channels and the heartbeat each have separate sub-sessions within it. Every exchange — messages, responses, tool calls, results — is appended to this file, and OpenClaw replays the entire history on every inference call. Sessions survive gateway restarts.

**Why this matters for agent setup:** The session history the agent has accumulated shapes its behavior just as much as the Sacred Eight files. An agent mid-session may exhibit behavior locked in by earlier examples even after you update its identity files. This is why we reset sessions as a deliberate part of deploying configuration changes (see §4.5).

**The latency dimension:** LLM generation runs sequentially at roughly 50–100 tps on a capable local GPU. Prefill — processing the entire input, including session history — runs in parallel and is much faster, typically 1–3 seconds for a typical context. But session history grows without a hard cap, while the system prompt is bounded at 150K characters. A long-running session eventually makes every interaction noticeably slower.

**Our practice:** We reset sessions nightly via `reset-sessions.sh` (run at 3 AM via cron). The script archives any session file larger than 512 KB and truncates it to zero; the agent starts fresh from its Sacred Eight files on the next heartbeat. This also reinforces the scheduling architecture: the agent cannot rely on remembering what it did in a previous session — any state that must survive a reset must live in the Sacred Eight files, `PATHS.md`, `CALENDAR.md`, or the shared filesystem.

**Resetting after a behavioral change:** When you edit an agent's `.md` files and need the agent to adopt the new behavior immediately (not wait for the 3 AM nightly reset), use `reset-session.sh`:

```bash
bash scripts/reset-session.sh <agent-id> --reason "Updated EMAIL.md rate limits"
```

The script archives all session files for the named agent, truncates them to zero, and logs the reset with the agent ID, timestamp, and reason to `shared/logs/sessions-reset.log`. This is the second step of the four-step behavioral change procedure (edit → reset → tell → verify); the reason log provides an audit trail of when and why each reset happened.

**Timing matters:** Schedule your nightly reset at least one to two hours before your earliest daily scheduled task fires. A freshly-reset agent needs one heartbeat to re-establish its session context; giving it time to do this before any CALENDAR.md entries come due prevents a race condition where the agent executes a task with a thin or empty session.

```bash
# Session management scripts (run via cron on host — see §5.5 for full crontab)
*/5  * * * *  monitor-sessions.sh   # log .jsonl sizes; alert if growing large
0    3 * * *  reset-sessions.sh     # archive >512KB sessions; truncate to zero
*/30 * * * *  seed-sessions.sh      # restore missing heartbeat session files
```

---



---

# SECTION 2: Enhancements

*Five areas where we augmented OpenClaw beyond its defaults: security and containment, ease of configuration management, scheduling, separation of responsibilities, and layered oversight. Each enhancement has a rationale, a strategy, and an implementation.*

## Enhancement 1 — Security, Safety, and Containment as Design Considerations

OpenClaw can be deployed and used without any of the network isolation, iptables rules, Tailscale binding, or sandbox hardening described in this section. None of these are defaults or requirements imposed by the platform. They are deliberate design choices we made — and the first thing we put in place before connecting any external service. This section covers both the *deployment* of the OpenClaw platform and the *security decisions* we layered on top. The two are inseparable: the right time to harden a system is before it has credentials and channel access, not after.

### E1.1 The Docker Stack

```
[You, via Tailscale — browser or Slack]
        |
        v  HTTPS via tailscale serve → port 18789 on Tailscale IP only
[OpenClaw gateway container]           ← dashboard + agent runtime
        |
        v  http://nim:8000/v1 — Docker-internal network only, never exposed to host
[vLLM container]  ← only present for local-model deployments
        |
        v
[GPU + unified memory]
```

You interact with the gateway in two ways: through Slack (the primary day-to-day interface), and occasionally through the web dashboard at port 18789 for configuration and monitoring. Both are accessible only via Tailscale — the gateway never binds to `0.0.0.0`. The model API is accessible only within the Docker-internal network and is never exposed to the host or the outside world.

**A practical note on gateway configuration:** Changes to `openclaw.json` — the live gateway configuration — are more awkward than they should be. The dashboard provides a small JSON editor that is functional but inconvenient for anything beyond minor edits. Our workflow: extract the current config from the running container, edit it with a proper editor or with Claude Code, copy it back into the container, and restart the gateway. We have scripts that handle the extract and restart steps; the editing is manual. This is a real friction point worth acknowledging — it is not unique to our setup, and there is no elegant solution yet.

**Tailscale prerequisite:** Tailscale must be installed and running on your host before deploying OpenClaw. See [tailscale.com/download](https://tailscale.com/download) for installation — the setup takes about five minutes and is well-documented. Once installed, `tailscale serve` handles the HTTPS proxy automatically.

### E1.2 Choosing Your Model

In our deployment we use a local GPU for some agents and a cloud API for others — and the choice can be changed at any time with a single config edit. The table below illustrates the tradeoffs:

| Path | Hardware requirement | Model examples | Characteristic |
|---|---|---|---|
| Local GPU | Machine with dedicated GPU and sufficient memory (e.g., NVIDIA DGX Spark GB10, high-end workstation, Mac with Apple Silicon M-series (and large memory)) | Qwen3-Coder-Next-FP8 via vLLM | Low per-token cost; throughput depends on hardware; typically <100 tps |
| Cloud API | Any Linux host (no GPU required) | Anthropic Claude (claude-sonnet-4-6, claude-haiku-4-5) | Pay-per-token; consistently fast; easier setup |

**On running a local model:** The key constraint is memory, not just GPU. A 32 GB unified memory machine (e.g., M-series Mac) can run smaller quantized models but will see lower throughput and may struggle with large context windows. Throughput matters for scheduled tasks — a 50-tps model can produce a 500-token email digest in 10 seconds; a 5-tps model takes 100 seconds. Anything below about 50 tps is not really viable.  Your chats will feel like you are talking over a 300 baud modem. If you are not already confident your hardware can run a capable model, the cloud API path removes this variable entirely and lets you focus on the agent architecture, which is the point of this tutorial.

**Our recommendation for the tutorial:** Use the cloud API (e.g., Anthropic Claude) unless you already have local model inference working. The architecture is identical — the only difference is one line in `config.yaml`.

### E1.3 Quick and Easy Model Switch 

In order to make it easy to swap models in for individual, multiple, or all agents, we created `config.yaml` to specify which model to use for each agent, and a set of scripts to push those assignments into OpenClaw. This is introduced here as part of platform setup; Module 8 covers config.yaml in depth as a broader configuration scaffolding layer.

OpenClaw stores per-agent model assignments inside `openclaw.json` — a large JSON configuration file that lives inside the running gateway container. Editing it directly means either using the dashboard's small JSON editor window or manually extracting the file, editing it, copying it back, and restarting the gateway. For a single change that's manageable; for experimenting across agents and models it becomes tedious and error-prone.

Our solution: `config.yaml`, a small human-readable file that lives in the infrastructure repo alongside your other config. `apply-config.sh` reads it, patches `openclaw.json` in the container, and restarts the gateway. The source of truth for model assignments is `config.yaml`, not the raw JSON.  We expect that over time we will find other configuration details that would be handy to pull out into config.yaml for the same reason.

```yaml
# Example of config.yaml — per-agent model assignments 
agents:
  main:
    model: anthropic/claude-sonnet-4-6    # cloud API
  gmail-agent:
    model: vllm/Qwen/Qwen3-Coder-Next-FP8  # local GPU
```

**`secrets.yaml`** (gitignored, never committed) holds API keys:
```yaml
anthropic_api_key: sk-ant-...
```

**Applying a change — step 1, preview:**
```bash
./apply-config.sh --dry-run
```
This prints the JSON diff that *would* be written to `openclaw.json` — useful for pasting to an AI assistant or reviewing manually to confirm the change looks right. It is a preview only; nothing is applied.

**Step 2, apply (only after reviewing the dry-run output):**
```bash
./apply-config.sh
```
Applies the patch to `openclaw.json` and restarts the gateway. Do not run both commands in a single copy/paste — the dry-run output needs to be inspected before applying.

**Reverting to a known-good configuration:** Before experimenting with a new model or provider, save your current working `config.yaml` as `config.yaml.stable`. If an experiment goes wrong — a model behaves poorly, you exhaust API credits, a provider has an outage — restore `config.yaml.stable` and run `apply-config.sh`. This pattern works whether your stable baseline is a local model or a cloud provider.

A key point: model assignment is not a permanent choice (and no one should be forced to edit 2-300 line json files in a tiny browswer edit window). Any agent can be switched to a different model in under 30 seconds without touching `docker-compose.yml` or fiddling with the dashboard or any other configuration.

### E1.4 Structure of this Repository

This repository is structured so that you can implement OpenClaw and an initial set of agents. The structure separates the OpenClaw and (if used) local model configurations -- the platform -- and the agent-specific configurations.

```
OpenClaw-Tutorial/               ← your private repo
├── gateway/                     ← platform config
│   ├── docker-compose files
│   ├── config.yaml              # per-agent model assignments + Slack channel bindings
│   ├── secrets.yaml             # gitignored — API keys
│   └── apply-config.sh
├── shared/                      ← runtime state shared across all agents
│   ├── outbox/ sent/ rejected/  # email review queue
│   ├── slack-outbox/ slack-sent/
│   ├── todos/                   # calendar-state.json (scheduling state)
│   └── logs/                    # all script and cron logs (single location)
├── main/                        ← supervisory agent (full file listing shown here)
│   ├── SOUL.md                  # Sacred Eight
│   ├── IDENTITY.md
│   ├── TOOLS.md
│   ├── USER.md
│   ├── HEARTBEAT.md
│   ├── BOOTSTRAP.md
│   ├── MEMORY.md
│   ├── AGENTS.md
│   ├── EMAIL.md                 # outbox review criteria (not auto-loaded)
│   ├── PATHS.md                 # canonical path registry
│   ├── CHANNELS.md              # Slack channel → agent routing
│   └── runbooks/
├── admin-agent/                 ← clerical agent (same structure as main)
└── gmail-agent/                 ← Gmail assist agent (same structure + scripts/)
    └── scripts/                 # gmail_api.py, contacts_api.py, etc.
```

`admin-agent` and `gmail-agent` have the same Sacred Eight files and supporting files as `main`. Only `gmail-agent` adds a `scripts/` directory for deterministic API tools.

**`shared/logs/` — the single log directory.** All cron scripts and pipeline scripts write their output to `shared/logs/` rather than to separate subdirectories. This makes troubleshooting straightforward: when something goes wrong, there is exactly one place to look. The log files are named by function: `notify.log` (nightly sync pipeline), `todos.log` (task lifecycle — READY promotions, COMPLETED removals), `todos-cron.log` (raw cron stdout from `check-todos.sh`), `sessions-cron.log`, `sessions-monitor.log`, `sessions-seed.log`, `sessions-reset.log`, `slack-posts.log`, and `email-send.log`. The `shared/todos/` and `shared/sessions/` directories continue to hold their non-log data files (`calendar-state.json`, session `.jsonl` files, etc.).

### E1.5 Security Hardening Checklist

Done once, before connecting any external service. Four verifications before calling the platform ready. These test the protections we have built into this deployment, but this does not meant that the system is involnerable - all of this is a work in progress.  But you are still better off with these protections and without them!

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

## Enhancement 2 — Ease of Management: config.yaml and apply-config.sh

OpenClaw stores its full runtime configuration in `openclaw.json` — a large JSON file inside the gateway container that controls model assignments, Slack channel bindings, sandbox settings, agent definitions, and more. Editing it requires either using the dashboard's small JSON editor (practical only for minor edits) or a manual extract-edit-copy-restart cycle. As our deployment grew in complexity, this friction became a real obstacle: changing one agent's model or updating a Slack channel binding required careful JSON surgery every time.

`config.yaml` and `apply-config.sh` are our solution — a clean abstraction that keeps human-readable configuration in a versioned file and pushes changes to the live gateway automatically. This is the fourth component of our scaffolding.

### E2.1 What config.yaml Manages

**Model assignments** — which LLM backs each agent:
```yaml
agents:
  main:
    model: anthropic/claude-sonnet-4-6
  admin-agent:
    model: vllm/Qwen/Qwen3-Coder-Next-FP8
  gmail-agent:
    model: anthropic/claude-haiku-4-5
```

**Slack channel bindings** — which channel routes to which agent (replaces the human-maintained `CHANNELS.md`):
```yaml
channels:
  - id: C09KGGMS116
    name: "#tpc-channel"
    agent: admin-agent
  - id: C0AJ1EL2KJ5
    name: "#testing"
    agent: admin-agent
```

**API keys** — in `secrets.yaml` (gitignored, never committed):
```yaml
anthropic_api_key: sk-ant-...
```

### E2.2 The apply-config.sh Workflow

```bash
./apply-config.sh --dry-run   # preview the patch to openclaw.json
./apply-config.sh             # apply and restart the gateway
```

`--dry-run` prints the JSON diff without applying it — useful for review before committing the change. Running without `--dry-run` patches `openclaw.json` in the container and restarts the gateway. Do not chain both commands: inspect the dry-run output first.

**Stable baseline pattern:** Before experimenting, save `config.yaml` as `config.yaml.stable`. If an experiment goes wrong — model behaves poorly, API credits exhausted, provider outage — restore `config.yaml.stable` and run `apply-config.sh`.

### E2.3 Why This Matters and Where It Goes Next

The value of this layer is that changes to running agent configuration take under 30 seconds and never require touching `docker-compose.yml`, restarting Docker stacks, or editing raw JSON. The barrier to experimenting with models and routing is essentially zero.

We expect this pattern to grow. Other configuration values in `openclaw.json` — sandbox settings, heartbeat intervals, tool permissions — are also candidates for config.yaml abstraction. Any value that a deployer might want to change regularly and safely without dashboard access is a good candidate. The mechanism is established; the scope can expand.

---

## Enhancement 3 — Scheduling: The Shell Owns the Clock

### E3.1 The Core Principle

> **The decision of when to act is always owned by deterministic code. The act of doing the work is owned by the LLM. These two responsibilities must never be mixed.**

Here we describe one of three useful ways that we have extended the utility of OpenClaw's base capabilities: Scheduling.  Why is this needed? An LLM has no clock. It has no reliable memory of having acted in a previous session. If you ask it to decide whether it is time to run a task, it will reason through that question every time it runs — and it will sometimes be wrong.


### E3.2 The Token-Waste Problem

A 15-minute heartbeat fires **672 times per week**. If `HEARTBEAT.md` instructs the agent to check whether it's time to run a weekly report, the agent reasons through that question 672 times — and acts on 1 of them:

| Task frequency | Heartbeats/week | Useful fires | Wasted LLM calls |
|---|---|---|---|
| Once a week | 672 | 1 | 671 (99.9% waste) |
| Mon + Thu | 672 | 2 | 670 (99.7% waste) |
| Weekdays at 9am | 672 | 5 | 667 (99.3% waste) |

Beyond token cost: the agent must remember whether it already ran the task today — state that does not survive a session reset. LLM-driven scheduling fires twice, then not at all, then twice again.

### E3.3 Three Tiers of Scheduling

| Tier | File | Owned by | Lifecycle | Use for |
|---|---|---|---|---|
| Always-on | `HEARTBEAT.md` | Human | Permanent | Infrastructure reflexes on every heartbeat |
| Recurring | `CALENDAR.md` | Human | Season or project | Day-of-week, daily, monthly recurring tasks |
| One-shot | `TODO.md` | Agent (at runtime) | Ad hoc | Single deferred tasks written during a conversation |

**The human analogy:** HEARTBEAT.md is brushing your teeth — every day, forever, no thought required. CALENDAR.md is "send the Monday/Thursday reports through June." TODO.md is "call the dentist" or "pick up eggs."

**The critical distinction for TODO.md vs. CALENDAR.md:** If a task needs to happen again next week, it belongs in CALENDAR.md. An agent that re-schedules recurring tasks by writing new TODO.md entries will lose track of them after a session reset — they disappear silently.

**What is a session?** In OpenClaw, a session is a conversation history file (`.jsonl`) stored in the gateway's persistent Docker volume. Each agent maintains its own session; within that, different Slack channels and the heartbeat each have their own sub-session. A session accumulates every exchange — user messages, agent responses, tool calls, and results — as an append-only log. Crucially, sessions survive gateway restarts: when the gateway comes back up, it picks up the history where it left off.

**Session reset** is not a native OpenClaw operation — it is our addition. OpenClaw never resets sessions on its own; they grow indefinitely. Our `reset-sessions.sh` script (run nightly via cron) archives any session file larger than 512 KB and truncates it to zero. The agent starts fresh on the next heartbeat, reading its identity and context from the Sacred Eight files rather than from accumulated history. When you deliberately change agent behavior, you should reset the relevant session immediately rather than waiting for the nightly job.

**Why HEARTBEAT.md must be stable:** Beyond the token-efficiency argument (the 672-heartbeats-per-week table), HEARTBEAT.md is consulted on every heartbeat for the lifetime of the agent. A mistake introduced while editing it — a typo, a malformed step, a removed closing line — can silently impair or disable the agent's core reflexes until caught and corrected. Keep HEARTBEAT.md lean and stable: infrastructure reflexes only, never task-specific procedures. Those belong in runbooks, where they can be changed without risking the heartbeat loop.

### E3.4 The Scheduling Engine: check-todos.sh

The scheduling engine rests on two agent-specific files in the workspace directory — `TODO.md` and `CALENDAR.md` — and a cron script (`check-todos.sh`) that bridges between them. Typically a human authors `CALENDAR.md` and the agent manages its own `TODO.md`, though allowing the agent to co-manage `CALENDAR.md` is possible (see below).

**A note on CALENDAR.md and agent modification:** `CALENDAR.md` lives in the agent's workspace, which OpenClaw mounts read-write. There is no technical enforcement preventing the agent from modifying it — the prohibition is a behavioral invariant encoded in `SOUL.md`. If you want your agent to be able to add or remove calendar entries, you can permit this by updating the relevant `SOUL.md` rule. The current convention (human-only authorship) is our implementation choice, not an OpenClaw constraint.

**Logging:** `check-todos.sh` appends a record to `shared/logs/todos.log` each time it promotes an entry to READY and each time it removes a COMPLETED line. The agent appends a completion record to the same log when it executes a READY task and marks the line COMPLETED in `TODO.md`; `check-todos.sh` removes COMPLETED lines on its next 5-minute run. The log is append-only and provides a full chronological history of what was scheduled, when it fired, and when it completed.

**File formats:**

`TODO.md` and `CALENDAR.md` use simple pipe-delimited notation specifying the date/time and the task (which typically references a runbook for its execution instructions).

```
# CALENDAR.md — human-authored, never modified by scripts or agents
MON,THU 14:00 | Send notifications per runbooks/RUNBOOK_NOTIFICATIONS.md

# TODO.md — agent writes one-shots; check-todos.sh promotes to READY
# Agent writes a one-shot:
2026-04-07T18:00:00Z | Follow up with Jones re: proposal

# check-todos.sh promotes it (replaces the line in-place):
READY | 2026-04-07T18:00:00Z | Follow up with Jones re: proposal

# check-todos.sh also appends recurring items from CALENDAR.md:
READY | 2026-04-07T14:00:00Z | Send notifications per runbooks/RUNBOOK_NOTIFICATIONS.md
```

**The execution flow — precisely:**

Every 5 minutes, `check-todos.sh` (run by cron) inspects both files and promotes any due items to `READY` status in `TODO.md`. The heartbeat then picks up READY items at its next 15-minute interval. CALENDAR.md is never modified by this process — it is the permanent human-authored record of recurring duties.

```
check-todos.sh  (cron, every 5 min, zero tokens)
  ├─ Removes COMPLETED lines from TODO.md, logging each removal to todos.log
  │
  ├─ Reads CALENDAR.md
  │    For each entry: is it due today/now AND not already fired today?
  │    (checks shared/todos/calendar-state.json for last-fired timestamp)
  │    If due: appends  READY | <timestamp> | <task>  to TODO.md
  │    Updates calendar-state.json. CALENDAR.md is never modified.
  │
  └─ Reads TODO.md
       For each timestamp line: is the time past?
       If yes: rewrites that line in-place as  READY | <timestamp> | <task>

OpenClaw heartbeat  (every 15 min, tokens only when READY lines exist)
  └─ Agent reads TODO.md, looks for lines beginning with READY
       ├─ None found → replies HEARTBEAT_OK  (fast, minimal tokens)
       └─ READY line found → executes the task per its runbook
            → marks the line COMPLETED in TODO.md (deterministic sed replace)
            → logs the completed task to shared/logs/todos.log
            → check-todos.sh removes the COMPLETED line on its next 5-min run
```

`CALENDAR.md` is the human-authored source of recurring duties — it is never modified by scripts or agents. `TODO.md` is the runtime queue; the full line lifecycle is: pending → READY (promoted by check-todos.sh) → COMPLETED (marked by agent) → removed (by check-todos.sh on next run), or FAILED (left in place for human review). Removal is deterministic and handled by the script — the LLM marks completion but does not delete lines.

**State Tracking: calendar-state.json**

`check-todos.sh` needs to know whether a given `CALENDAR.md` entry has already fired today — otherwise it would re-promote recurring items on every 5-minute run. It solves this with a small JSON file on the host: `shared/todos/calendar-state.json`. Each time a recurring entry fires, the script records the entry's key and the current timestamp in this file. On subsequent runs it checks the file before promoting — if the entry already fired within its scheduling window (e.g., today for a DAILY entry, or this week for a MON entry), it is skipped. Because this state lives on the host filesystem rather than in the agent's memory, it survives session resets. An agent that loses its session history at 3 AM will still not re-fire its Monday report on Tuesday, because `calendar-state.json` knows it already ran.

### E3.5 The Host-Side Crontab

All scheduling and session management runs on the host as cron jobs — never inside any container. Crontab supports comment lines, so the entries are grouped by purpose:

```
TZ=America/Chicago

# ── Scheduling engine (see §5.4) ─────────────────────────────────────────────
*/5  * * * *  check-todos.sh           # promote due CALENDAR entries and past-due
                                        # TODO entries to READY; zero LLM tokens

# ── Outbox processing (see §7.2) ─────────────────────────────────────────────
*/30 * * * *  send-approved-emails.sh  # send approved outbox drafts via Gmail API;
                                        # move to sent/; archive rejected/

# ── Session management (see §5.5) ────────────────────────────────────────────
*/5  * * * *  monitor-sessions.sh      # log session .jsonl sizes every 5 min;
                                        # alert if approaching reset threshold
0    3 * * *  reset-sessions.sh        # archive sessions >512KB, truncate to zero;
                                        # agent restarts clean from Sacred Eight files
*/30 * * * *  seed-sessions.sh         # restore missing heartbeat session files
                                        # after gateway restarts
```

Nothing in this crontab requires the LLM to be running. The scheduling engine, outbox sender, and session manager all operate independently of whether any agent is active.

---

## Enhancement 4 — Separation of Responsibilities: Runbooks, Scripts, and Cron

Module 5 established that the shell owns the clock. This module establishes what happens below the LLM when the clock fires: a layered system of runbooks, deterministic scripts, and cron that handles everything procedural so the model never has to. These three layers, combined with the scheduling layer, are the scaffolding that makes our deployment reliable and maintainable. Each has a distinct role and a distinct location in the repository; this module maps the architecture to implementation.

### E4.1 The Three-Layer Pattern in Practice

The layered architecture from §2.3 is now concrete. The LLM sits atop three deterministic layers; this module describes those three layers and how they work together.

| Layer | Location | Role |
|---|---|---|
| **LLM** | OpenClaw gateway | Judgment, composition, language — invokes runbooks for procedural tasks |
| **Runbooks** | `agent/runbooks/RUNBOOK_*.md` | Step-by-step procedures: which scripts to call, in what order, with what arguments, and what to do with the output |
| **Scripts** | `agents/scripts/*.py`, `*.sh` | Deterministic work: fetch, transform, count, route, call APIs, return structured results |
| **Cron** | Host crontab | Decides when: promotes READY items, sends approved outputs, manages sessions |

The LLM never decides when to act. Cron never decides what to say. Scripts never decide what action to take — they execute deterministically and return results. Each layer does only what belongs to it.

### E4.2 Runbooks: Procedures the Agent Follows

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

### E4.3 Scripts: Deterministic Tools for the Agent

Scripts are registered tools — the agent calls them via `exec` inside the sandbox, reads the output, and acts on it. They are invoked by runbooks, not by HEARTBEAT.md directly.

Key properties of well-designed agent scripts:
- **No LLM dependency** — the script does not call the model. It fetches, transforms, counts, or routes.
- **Structured output** — returns JSON the LLM can reliably parse
- **No third-party dependencies if avoidable** — `gmail_api.py` uses Python stdlib + `urllib` only; no pip install in the sandbox
- **Single responsibility** — one script, one job. The LLM composes multiple script calls; scripts do not compose each other.

**Example scripts from the worked example:**
- `gmail_api.py` — search and fetch email via Gmail API; stdlib only
- `contacts_api.py` — search and create contacts via Google People API; stdlib only
- `notify.py` — run the notification pipeline: call `sync-track-sheets.py`, compose emails, post Slack summary. **Script assembles all structural email content deterministically** (salutation, count line, submission list, URLs, closer, footer); LLM contributes one bounded optional sentence per email for warmth/context. Output is validated (length, no newlines, no URLs, no markdown); if rejected or LLM unavailable the sentence slot is blank and the email is still complete and correct. Calls LLM API directly; must run inside Docker where `nim:8000` is accessible via `run-notify.sh`. See Lesson 7.
- `sync-track-sheets.py` — read master Google Sheet, route new submissions to per-track sheets; fully deterministic, no LLM

**Dry-run mode as standard practice.** Any script that modifies external state (writes to Google Sheets, queues emails, posts to Slack) should support a `--dry-run` flag that exercises all reads but skips all writes. This is especially important for pipelines that run on a daily cron schedule — a bug introduced by a change to the script may not surface until the next scheduled run, potentially 24 hours later. A dry run lets you verify the full pipeline immediately after a change.

In our notify pipeline, `run-notify.sh --dry-run`:
- Reads all Google Sheets normally (tests connectivity and data access)
- Skips all `gog_append` calls — track sheets are not modified; the next real run still detects the same submissions as new
- Skips the `wg-bof-state.json` state write — preserves the baseline for the next real run
- Posts the Slack summary to `#openclaw-test` (configured in `config.yaml` under `notify: dry_run_slack_channel_id`) with a `[DRY RUN]` banner
- Writes emails to outbox redirected to the system owner (configured in `config.yaml` under `notify: system_owner_email`) with `[DRY RUN]` subject prefix — the full outbox/send pipeline is exercised

The test channel and system owner email live in `config.yaml` so they can be changed for a new deployment without touching the scripts themselves. `run-notify.sh` extracts these values at startup and passes them as environment variables into the Docker container.

**`--test` for composition verification.** `--dry-run` exercises the full pipeline against real Google Sheets data. A complementary `--test` flag uses a built-in fake-submission fixture so the pipeline can be run at any time — even when there are no new real submissions — to verify LLM output format and email structure without touching Sheets, the outbox, or Slack. This is especially useful after prompt or template changes. Note: `--test` must be run via `run-notify.sh --test` (not `python3 notify.py --test` directly) because the LLM endpoint `nim:8000` is only accessible inside the Docker container network.

**Smoke-test suite after any notify.py change** (also useful for verifying a new deployment). A single script runs all five tests and prints a clear PASS/FAIL summary — no need to read through log output. Run from `~/code/spark-ai-agents` on spark-ts:

```bash
scripts/run-tests.sh
```

The five tests and what each covers:

| Test | What it checks | External side effects |
|------|---------------|----------------------|
| Test 1 | Email composition and LLM warm sentence — uses built-in fixture, output to stdout | None |
| Test 2 | Google Sheets connectivity and Slack post to `#tpc-openclaw` | Slack post to `#tpc-openclaw` |
| Test 3 | Full pipeline dry-run: real Sheets, Slack to `#openclaw-test`, email to system owner if new submissions exist | Slack post to `#openclaw-test`; email to system owner if new items |
| Test 4 | Log check — scans last 100 entries for ERROR/ABORT/FAIL lines | None |
| Test 5 | Outbox → Gmail send pipeline — writes a dummy approved email to the outbox and confirms it reaches `sent/` | Email to system owner |

Tests 3 and 5 make no sheet writes and send no emails to real recipients. The test suite exits with a one-line summary: `ALL TESTS PASSED` or `N TEST(S) FAILED` with per-test details for any failures.

---

## Enhancement 5 — Multi-layer Oversight: The Outbox and Review Pattern

The outbox/review/send pattern is our third scaffolding component, built on top of scheduling (Module 5) and deterministic scripts (Module 6). It is the mechanism by which the deployment earns the right to operate with progressively less human involvement. We start with review at every consequential outbound action. As behavior proves consistent and review criteria become well-defined, we delegate the reviewer role from human → supervisor agent → lighter model. The mechanism — the outbox directory, the JSON format, the cron-based sender — stays constant at every stage of that progression.

### E5.1 The Email Outbox — Mechanism and Format

The email outbox is our reference implementation of the pattern. The drafting agent writes a JSON file to a shared queue; the reviewing agent (or human) approves or rejects by updating that file; a cron script sends approved drafts and archives the results. No LLM is involved in the send step.

**A note on human vs. agent review and file manipulation:** The reviewing agent (main) is the sole entity that writes to, moves, or deletes files in `outbox/` and `rejected/`. This applies to human-directed decisions as well: when a human wants to approve or reject an email, they communicate their decision to main via Slack DM, and main performs the file operation. This keeps the audit trail consistent (all file state changes come from one actor) and prevents a human from accidentally corrupting a JSON file with a manual edit. If main finds an outbox file that appears to have been edited manually, it flags the file and holds it for inspection rather than processing it.

**JSON format** (written by the drafting agent as an atomic `.tmp` rename):
```json
{
  "to": "recipient@example.com",
  "subject": "Subject line",
  "body": "Full email body, plain text",
  "from_agent": "gmail-agent",
  "context": "Why this email is being sent — for reviewer only, not included in the email",
  "status": "pending",
  "created_at": "2026-04-07T14:00:00Z"
}
```

**After approval** (reviewing agent updates the file in-place using `jq`):
```json
{ ..., "status": "approved", "approved_at": "2026-04-07T14:03:00Z" }
```

`jq` is a lightweight command-line JSON processor — standard on Linux. It allows scripts and agents to read and update JSON files safely without risk of corrupting the structure. The reviewing agent always uses `jq` to update outbox files; manual editing is prohibited.

**After rejection** (reviewing agent updates and moves to `rejected/`):
```json
{ ..., "status": "rejected", "rejected_at": "...", "rejected_reason": "Rate limit exceeded: 10 emails/24h to this recipient" }
```

**Directory structure:**
```
shared/
├── outbox/        ← pending drafts
├── sent/          ← delivered emails (moved here by cron after send)
└── rejected/      ← rejected drafts; rejected_reason field in each JSON file
```

**`send-approved-emails.sh`** (cron, every 30 min — our addition, not native OpenClaw): scans `outbox/` for files with `status: approved`, sends via Gmail API, moves to `sent/`, appends to `shared/logs/email-send.log`. Recipient resolution at send time: if the outbox JSON contains a `to_name` field (full name), the script calls `gog contacts search` to look up the email address from Google Contacts — Google Contacts is the single source of truth for email addresses. If the name is not found, the email is moved to `rejected/` with a clear log entry. If the outbox JSON contains a `to` field (email address directly, used for dry-run overrides), it is used as-is.

**Review criteria — our implementation choices, not scaffolding requirements:** The review logic lives in the supervisor agent's `EMAIL.md`. What follows is our current policy; every deployment will define its own criteria.
- No more than 10 emails per 24 hours to any single recipient (rate limit)
- Recipient must exist in the Google Contacts database (tpc26agent@gmail.com) — enforced at send time via name lookup
- Appropriate language and tone (defined in writing guidelines, also in `EMAIL.md`)
- Any rejection triggers an immediate Slack DM to the operator with reason and an override option

These are examples. A different deployment might permit 50 emails per day, accept any valid email address, or apply entirely different content criteria. The criteria are data; the mechanism is the scaffolding.

### E5.2 Adding Review to Slack: A Step-by-Step Example

The Slack outbox infrastructure already exists in our deployment (`shared/slack-outbox/`, `shared/slack-sent/`, `shared/logs/slack-posts.log`), providing an audit trail for all outbound posts. A review step has not yet been added. The following is how to implement one — and how to extend the same pattern to any other channel.

**Step 1 — Create the review criteria file.** Add `SLACK.md` to the supervisor agent's (`main/`) workspace directory. Define the criteria: acceptable channels, rate limits, content rules. This mirrors the role of `EMAIL.md` for email.

**Step 2 — Add a review step to the supervisor's heartbeat.** In `main/HEARTBEAT.md`, after the existing email outbox review step, add: "Check `shared/slack-outbox/` for pending posts. For each, apply the criteria in SLACK.md. Approve (update status to `approved`) or reject (update status to `rejected`, add `rejected_reason`, move to `shared/slack-rejected/`)."

**Step 3 — Create the rejected directory.** `mkdir -p shared/slack-rejected/`

**Step 4 — Update `send-slack-posts.sh`.** Modify the script to check for `status: approved` before posting (currently it posts all pending items). This is the only code change required.

**Step 5 — No new cron entry needed.** `send-slack-posts.sh` already runs every 5 minutes. The supervisor's heartbeat runs every 15 minutes and will now review slack drafts alongside email drafts.

The same five-step pattern — criteria file, heartbeat step, rejected directory, updated send script, no new cron entry — applies to any outbound channel supported by OpenClaw.

### E5.3 The Pattern Across Channels

| Component | Email (implemented) | Slack (partial — review not yet added) | Generic |
|---|---|---|---|
| Agent writes to | `outbox/<file>.json` | `slack-outbox/<file>.json` | `<channel>-outbox/` |
| Review criteria | `EMAIL.md` in supervisor | `SLACK.md` *(to add)* | `<CHANNEL>.md` |
| Reviewer heartbeat step | In `main/HEARTBEAT.md` | *(to add)* | Add to supervisor's HEARTBEAT.md |
| Send script | `send-approved-emails.sh` | `send-slack-posts.sh` | Write per channel |
| Archive | `sent/`, `rejected/` | `slack-sent/`, `slack-rejected/` *(to add)* | Per channel |
| Audit log | `shared/logs/email-send.log` | `shared/logs/slack-posts.log` | `shared/logs/` |

The deterministic send script and the append-only audit log are always present. The criteria file, the heartbeat step, and the rejected directory are what you add for each new channel.

---



---

# SECTION 3: Integrations

*Good starting points for connecting external services.*

## Integrations: Slack and Google

### I.1 Slack

Slack is connected to the **OpenClaw gateway** — not directly to individual agents. The gateway receives all incoming Slack messages and routes them to the appropriate agent based on channel or DM. The agents never interact with Slack directly; everything flows through the gateway and `openclaw.json` configuration.

*(Full step-by-step setup is in `Slack-Integration.md` in this repository. The following is an overview sufficient for slide-level discussion.)*

**On the Slack side** — three things to configure at api.slack.com/apps:
- Create a Slack app with Socket Mode enabled. Socket Mode lets OpenClaw receive events over a persistent WebSocket rather than requiring a public inbound URL.
- Grant the bot the required OAuth scopes: `channels:history`, `chat:write`, `users:read`, `groups:history`. Do not add `assistant:write` — this triggers a Slack-managed AI UI that conflicts with OpenClaw's handling.
- Note the bot token (`xoxb-...`) and app token (`xapp-...`) that Slack generates. These go into `openclaw.json`.

**On the OpenClaw side** — three things to configure in `openclaw.json`:
- Add the bot token and app token under `channels.slack`.
- Register each Slack channel the bot should respond in (by channel ID) under `channels.slack.channels`. OpenClaw will ignore messages from unlisted channels.
- Use `bindings` to route specific channels to specific agents. One agent is marked `"default": true` and handles all DMs and unrouted messages.

**Channel routing in `config.yaml`** — Slack channel bindings are managed in `config.yaml` alongside model assignments, using the same `apply-config.sh` workflow. Each entry maps a Slack channel ID to an agent and includes a human-readable name for reference. `apply-config.sh` rebuilds the `bindings[]` array in `openclaw.json` from this list on every apply. The default agent (marked `"default": true` in `openclaw.json`) handles all DMs and any channel not explicitly listed — it does not need an entry. `CHANNELS.md` in the workspace provides a quick-reference table derived from this config; the authoritative source is `config.yaml`.

```yaml
# config.yaml — channel routing section
channels:
  - id: C09KGGMS116
    name: "#tpc-channel"
    agent: admin-agent
  - id: C0AJ1EL2KJ5
    name: "#testing"
    agent: admin-agent
```

**The outbound post problem:** Agents can reply within active Slack sessions. They cannot initiate a post when no session is active (e.g., a scheduled overnight report). The slack-outbox pattern handles this: the agent writes JSON to `shared/slack-outbox/` and `send-slack-posts.sh` (cron, every 5 minutes) posts via the Slack `chat.postMessage` API and archives to `slack-sent/`.

### I.2 Credential Handling for External Services

Any integration that requires authentication — Google, Slack, or other services — must make credentials available to the agent's sandbox container. OpenClaw's sandbox is ephemeral and filesystem-isolated; credentials do not appear automatically. The mechanism is explicit bind mounts in `openclaw.json` per agent:

- **Read-only mounts** for credentials and scripts (e.g., OAuth `credentials.json`, Python scripts): the sandbox can read but not modify these.
- **Read-write mounts** for token files (e.g., OAuth `token.json`): the sandbox must be able to write back refreshed tokens, or the OAuth session will expire on the next run.

This applies identically to Google OAuth tokens, Slack bot tokens stored on disk, or any other credential file. Each integration section below notes its specific mount requirements. *(See `Google-Integration.md` and `Slack-Integration.md` in this repository for full setup detail.)*

### I.3 Google — Two Integration Paths

We use two different tools to reach Google services, for specific reasons:

| Tool | Repository | Best for | Why |
|---|---|---|---|
| `gsuite-mcp` | [github.com/MarkusPfundstein/mcp-gsuite](https://github.com/MarkusPfundstein/mcp-gsuite) | Gmail, Google Contacts | OAuth browser flow; writes `token.json`; works well with Gmail `messages.list` API |
| `gog` CLI | [github.com/ditto-assistant/gog](https://github.com/ditto-assistant/gog) | Google Sheets, Docs, Drive | Better for bulk read/write on structured documents |

**Why two tools?** `gog` uses `threads.list` for Gmail, which returns only the first message of a thread and ignores `in:sent` filters — unsuitable for inbox and sent-mail workflows. `gsuite-mcp` handles OAuth and token refresh cleanly; our scripts (`gmail_api.py`, `contacts_api.py`) read the token it writes and call the Gmail API directly via Python `urllib` (no third-party pip dependencies in the sandbox). For Drive and Sheets, `gog` is more capable. If your use case doesn't require Gmail inbox access, `gog` alone may suffice.

**Google Cloud setup** (done once before the lab — full steps in `Google-Integration.md`):
- Create a Google Cloud project and enable the APIs you need: Gmail, People, Drive, Sheets as applicable
- Configure the OAuth consent screen (internal or external depending on your Google Workspace plan)
- Download `credentials.json` from the API credentials page
- Run `gsuite-mcp auth login` (browser-based OAuth flow, one time) to generate `token.json`
- Bind-mount both files into the agent sandbox per §8.2 above

---



---

# Appendix

## Appendix: Lessons Learned

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

**Lesson 7: Don't prompt-engineer LLMs not to break structure — invert control instead**

The general principle: asking an LLM to compose an entire output (an email, a report, a Slack post) while instructing it via prompt *not* to corrupt specific elements (URLs, data fields, subject lines, signatures) is playing whack-a-mole. Each constraint you add is another thing the LLM can violate on any given run. The LLM has full control over the output — and LLMs do not reliably follow formatting constraints across many invocations.

How it manifests:
- Prompt says "use plain-text URLs only — do NOT use Markdown `[text](url)` formatting." A fraction of the time the LLM uses Markdown links anyway. Gmail doesn't render them. Recipients see `[Track sheet](https://...)` instead of the URL.
- Prompt says "sign it {AGENT_NAME}." The LLM signs with `[Your Name]` because that's what it's seen in training examples of email templates.
- Prompt says "include this list verbatim." The LLM rephrases it, reorders it, or drops entries that seem redundant to it.
- Every time you fix one constraint failure, you introduce the risk of another — because you're fighting for control of something the LLM fully owns.

The fix: invert control. The script assembles everything that must be correct — salutation, counts, data, URLs, closer, footer. The LLM gets one narrow, bounded task (e.g., "write one warm sentence") with output validation (length, no newlines, no URLs, no markdown). If the LLM fails or its output looks wrong, the slot is blank and the email is still complete and correct. The LLM cannot break URLs it was never asked to write.

This is a specific application of the separation-of-responsibilities principle (§2.3), but it is worth stating separately because the failure mode is subtle: the output looks correct *most* of the time, making it easy to miss the pattern of occasional failures until recipients start noticing them.

The pattern generalizes: wherever you find yourself writing prompt instructions that say "do not," consider whether the script should own that element entirely rather than instructing the LLM not to corrupt it.

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
| `config.yaml` | `gateway/` | N/A | Per-agent model assignments and Slack channel bindings |
| `secrets.yaml` | `gateway/` | N/A | API keys (gitignored) |
| `check-todos.sh` | `agents/scripts/` | N/A | Bash scheduling engine (cron); promotes READY entries; removes COMPLETED lines |
| `send-approved-emails.sh` | `agents/scripts/` | N/A | Sends human-approved outbox emails |
| `reset-sessions.sh` | `agents/scripts/` | N/A | Daily session archive and truncate (nightly cron) |
| `reset-session.sh` | `agents/scripts/` | N/A | Manual single-agent session reset with reason logging (behavioral changes) |

---

*Outline version: 2026-03-20 rev 14 — E5.1 updated: to_name/Google Contacts name-resolution pattern documented; EMAIL.md updated to match.*

---

## References

[1] Zverev et al., "Can LLMs Separate Instructions From Data? — Formalizing and Testing Instruction Hierarchy," ICLR 2025. *(Cited in §2.1 — prompt injection as a structural vulnerability in current LLM architectures)*

[2] Masterman, T.; Besen, S.; Sawtell, M.; and Chao, A., "The Landscape of Emerging AI Agent Architectures for Reasoning, Planning, and Tool Calling: A Survey," arXiv:2404.11584, 2024. *(Cited in §2.3 — reliability-critical applications systematically require more deterministic orchestration)*

[3] Hong, S.; Zhuge, M.; et al., "MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework," ICLR 2024 (oral). arXiv:2308.00352. *(Cited in §2.3 — SOPs encoded in code reduce cascading hallucinations from naively chained LLMs)*

[4] Qiu, L.; Ye, Y.; Gao, Z.; et al., "Blueprint First, Model Second: A Framework for Deterministic LLM Workflow," arXiv:2508.02721, 2025. *(Cited in §2.3 — deterministic orchestration engine + bounded LLM sub-tasks = verifiable, auditable agent behavior)*

[5] Kapoor et al., "The 2025 AI Agent Index: Documenting Technical and Safety Features of Deployed Agentic AI Systems," MIT / arXiv:2602.17753, 2025. *(Cited in §2.4 — majority of deployed agents document no sandboxing or containment mechanisms)*

[6] Shankar et al., "Agentic AI Security: Threats, Defenses, Evaluation, and Open Challenges," arXiv:2510.23883, 2025. *(Cited in Lesson 6 — containment and runtime monitoring are essential requirements, not optional hardening)*

[7] Yomtov, O. (Koi Security), "ClawHavoc: Large-Scale Poisoning Campaign Targeting the OpenClaw Skill Market," February 2026. Covered by Trend Micro, CyberPress, SecurityWeek, and others. *(Cited in §1.3 and §2.1 — real-world consequence of missing supply-chain discipline: ~20% of ClawHub registry compromised)*

[8] SecurityScorecard STRIKE Team, "ClawJacked: WebSocket Hijack Vulnerability in OpenClaw Gateways Exposed to Internet," February 2026 (CVE-2026-25253). [Coverage: The Hacker News](https://thehackernews.com/2026/02/clawjacked-flaw-lets-malicious-sites.html). *(Cited in §1.3 and §2.4 — 135,000+ publicly exposed instances; 15,000+ vulnerable to RCE; Tailscale binding is the direct mitigation)*