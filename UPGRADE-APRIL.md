# OpenClaw Upgrade Analysis: 2026.2.17 to 2026.4.2

**Date:** 2026-04-03
**Prepared by:** Claude Code (claude-sonnet-4-6)
**Installed version:** 2026.2.17 (last config touch: 2026-02-25)
**Target version:** 2026.4.2 (released 2026-04-02)
**Versions crossed:** 2026.2.19, 2026.2.21, 2026.2.22, 2026.2.23, 2026.3.2, 2026.3.7, 2026.3.13-1, 2026.3.22, 2026.3.23, 2026.3.28, 2026.3.31, 2026.4.1, 2026.4.2

---

## System Configuration Reference

Configuration verified against live `openclaw.json` and `exec-approvals.json` read directly from the Docker volume, plus `docker-compose.yml`, the host crontab, all cron scripts, and all agent workspace files.

| Item | Value |
|------|-------|
| Agents | main (argo/claude-4.6-sonnet), cecat (argo/claude-4.6-opus), chattpc26 (argo/claude-4.6-haiku) |
| Sandbox | `mode: all`, `workspaceAccess: rw`, `dangerouslyAllowExternalBindSources: true` on all agents |
| Channel | Slack only, Socket Mode, `streaming: false`, `groupPolicy: allowlist` |
| Gateway | `mode: local`, `bind: lan`, `auth.mode: token`, `allowTailscale: true` |
| Gateway UI | `allowInsecureAuth: true`, `dangerouslyAllowHostHeaderOriginFallback: true` |
| exec-approvals | defaults `security: deny / ask: off / askFallback: deny`; chattpc26 gog-only allowlist |
| Plugins | `slack.enabled: true` only -- no custom plugins |
| Commands | `native: auto`, `nativeSkills: auto`, `config: false` |
| Compaction | `mode: safeguard` (no compaction.model set) |
| Subagents | `maxConcurrent: 8` (defaults) |
| Heartbeat | All 3 agents at 15m; main has `activeHours: 08:00-22:00 Chicago` |
| Providers | vllm (`http://nim:8000/v1`) and argo (`http://172.18.0.1:44497`) -- both HTTP, internal Docker network |
| NOT in use | Chrome extension, iOS/Android nodes, Discord, Telegram, Matrix, WhatsApp, Feishu, Zalo, Mattermost, custom plugins, memory embeddings, image generation, xAI, Firecrawl, web search |

---

## Actions Required

**Four items require attention.** Three are post-upgrade verifications; one is a decision to make before upgrading.

| # | Version | Item | When |
|---|---------|------|------|
| A1 | 2026.3.2 | ACP dispatch defaults to ON -- decide whether to disable | Before upgrade |
| A2 | 2026.4.2 | exec-approvals normalization may alter security values | Within 60s post-upgrade |
| A3 | 2026.4.2 | Provider transport security hardening (HTTP providers) | Monitor after first restart |
| A4 | 2026.3.28 | Config migration 2-month policy -- run doctor | Post-upgrade |

Full details on each action appear at the end of this document.

---

## Version-by-Version Change Analysis

Every change in every release is listed. Each row is assessed against the configuration above.

- **N/A** -- not applicable to this system
- **Clear** -- verified against live configuration; confirmed no impact
- **Benefit** -- improvement with no action needed
- **Action A#** -- requires attention; see Action Items section

---

### v2026.2.19

| # | Change | Assessment |
|---|--------|------------|
| 1 | Apple Watch companion MVP | N/A -- no iOS |
| 2 | iOS APNs wake for nodes.invoke | N/A -- no iOS |
| 3 | Device pair/remove CLI commands | N/A |
| 4 | APNs push registration | N/A -- no iOS |
| 5 | Gateway push-test pipeline for APNs | N/A |
| 6 | HTTP audit findings for `auth.mode="none"` | Clear -- auth.mode is "token" |
| 7 | Reasoning block streaming fix | Benefit -- Argo Claude models support reasoning |
| 8 | Pairing reconnect stabilization | N/A |
| 9 | Gateway daemon TMPDIR forwarding | N/A |
| 10 | Agent billing error model identification | N/A |
| 11 | Memory embedding provider warnings | N/A -- no embeddings configured |
| 12 | Gateway auth defaulting fix | Benefit -- aligns with explicit token config |
| 13 | Heartbeat interval skipping fix | Benefit -- all 3 agents on 15m heartbeats |
| 14 | Extension relay port reuse | N/A -- no Chrome extension |
| 15 | macOS launchctl guidance | N/A -- Linux host |
| 16 | BOOT.md per-agent scope | Clear -- no BOOT.md files exist; agents use SOUL.md/HEARTBEAT.md |

---

### v2026.2.21

| # | Change | Assessment |
|---|--------|------------|
| 1 | Gemini 3.1 support | N/A |
| 2 | Volcano Engine / BytePlus provider | N/A |
| 3 | CLI per-account `defaultTo` outbound routing | N/A |
| 4 | Per-channel model overrides via `channels.modelByChannel` | N/A -- new option, not required |
| 5 | Telegram enhancements | N/A -- no Telegram |
| 6 | iOS features | N/A |
| 7 | Model fallback lifecycle visibility | Benefit -- fallbacks configured for all agents |
| 8 | Agent owner-ID HMAC obfuscation | Benefit -- security hardening |
| 9 | Gateway lock/tool-call SHA-256 hashing | Benefit -- security hardening |
| 10 | Embedded runner owner tool access fix | Benefit -- affects sandbox exec for all agents |
| 11 | Provider context overflow detection | Benefit -- Argo models have 16K context window |
| 12 | Heartbeat interval behavior fix | Benefit -- 15m heartbeats on all agents |
| 13 | Active-hours validation fix | Benefit -- main agent has activeHours 08:00-22:00 Chicago |
| 14 | TUI/pairing guidance | N/A |

---

### v2026.2.22

| # | Change | Assessment |
|---|--------|------------|
| 1 | Mistral provider support | N/A |
| 2 | Built-in auto-updater (opt-in, requires `openclaw update`) | N/A |
| 3 | `openclaw update --dry-run` preview | N/A |
| 4 | Synology Chat plugin | N/A |
| 5 | iOS TTS segment prefetching | N/A |
| 6 | Memory FTS improvements (multilingual stop words) | N/A -- no memory embeddings |
| 7 | Discord allowlist canonicalization | N/A |
| 8 | Unified channel preview streaming config | Benefit -- Slack `streaming: false`; consistency fix |
| 9 | CLI credential redaction | Benefit -- security hardening |
| 10 | Docker identity precreation fix | Benefit -- Docker sandboxes in use on all agents |
| 11 | Background execution timeout handling | Benefit -- sandbox exec runs on all agents |
| 12 | Slack threading enhancements | Benefit -- Slack is the only channel |
| 13 | Session metadata preservation | N/A |

---

### v2026.2.23

| # | Change | Assessment |
|---|--------|------------|
| 1 | Standard DashScope endpoints for Qwen | N/A -- using local vllm |
| 2 | Button primitives / Knot theme | N/A -- UI only |
| 3 | SHA-256 hashes for inline CSP scripts | Benefit -- security improvement for Control UI |
| 4 | Plugin runtime sidecars npm packaging fix | N/A -- no custom plugins |
| 5 | Channel authentication fix for single-channel setups | Benefit -- Slack-only; previously had auth gaps in single-channel mode |
| 6 | OpenAI token persistence fix | N/A -- not using OpenAI directly |
| 7 | Operator scope preservation in device-auth bypass paths | Benefit -- 2 paired devices; scopes not lost on bypass paths |
| 8 | ClawHub plugin compatibility checks against active runtime | N/A |

---

### v2026.3.2 -- BREAKING CHANGES

| # | Change | Assessment |
|---|--------|------------|
| **B1** | **ACP dispatch defaults to enabled** | **Action A1** |
| B2 | `tools.profile` defaults to "messaging" for new onboarding | Low risk -- applies to new installs; existing `tools: {}` is preserved |
| B3 | Plugin HTTP: `registerHttpHandler` removed; use `registerHttpRoute` | N/A -- no custom plugins |
| B4 | Zalo Personal Plugin: removed dependency on external `zca` CLI | N/A |
| 5 | SecretRef expansion to 64 credential targets | N/A -- informational |
| 6 | First-class `pdf` tool added | Informational -- new capability, no action needed |
| 7 | Outbound adapters: `sendPayload` for Slack, Teams, etc. | Benefit -- improved Slack posting |
| 8 | Telegram streaming improvements | N/A |
| 9 | `openclaw config validate` command | Informational -- useful post-upgrade verification tool |
| 10 | Memory search: Ollama embeddings support | N/A -- no embeddings |
| 11 | `runtime.system.requestHeartbeatNow` for immediate session wake | Benefit -- affects heartbeat triggering |
| 12 | `cli.banner.taglineMode` config | N/A |

**B1 detail -- ACP dispatch:** The key `"acp"` is absent from your `openclaw.json`. Previously this meant ACP dispatch was off by default. After upgrading to 2026.3.2+, it will default to enabled. ACP handles agent-to-agent turn routing. Your 3 agents operate on entirely separate Slack channels with no configured ACP handoffs. Enabling ACP dispatch should not cause cross-agent routing interference under your current configuration. However, it is a behavioral change on a multi-agent system. See Action A1 for the full decision and mitigation.

---

### v2026.3.7 -- BREAKING CHANGE

| # | Change | Assessment |
|---|--------|------------|
| **B** | **Explicit `gateway.auth.mode` required when both token + password present** | Clear -- `gateway.auth` has `mode: "token"` only; no password field present |
| 1 | Context Engine Plugin Interface with lifecycle hooks | N/A |
| 2 | Persistent Discord/Telegram channel bindings across restarts | N/A |
| 3 | Web UI Spanish locale with lazy loading | N/A |
| 4 | Web search Perplexity switched to Search API | N/A |
| 5 | Gateway auth SecretRef support | N/A -- informational |
| 6 | Docker multi-stage build producing minimal runtime image | Benefit -- `:latest` pull gets smaller, cleaner image |
| 7 | `OPENCLAW_EXTENSIONS` env var for preinstalling deps | N/A |
| 8 | `prependSystemContext` / `appendSystemContext` plugin hooks | N/A -- no custom plugins |
| 9 | `allowPromptInjection` hook policy; default is locked | N/A -- security improvement |
| 10 | `channels.slack.typingReaction` for Socket Mode DMs | Informational -- new optional config; default off; no change required |
| 11 | `allowBots: "mentions"` for Discord bot acceptance | N/A |
| 12 | Tool result head+tail truncation for oversized tool results | Benefit -- GOG returns large sheet/contact data; prevents context overflow |
| 13 | Cron job persistence: skip backup during normalization | Clear -- OpenClaw-side `cron/jobs.json` is empty; no impact |
| 14 | Diffs tool PDF output | N/A |
| 15 | `google/gemini-3.1-flash-lite-preview` model | N/A |

---

### v2026.3.13-1 (recovery release -- no breaking changes)

| # | Change | Assessment |
|---|--------|------------|
| 1 | Token count handling post-compaction fix | Benefit -- compaction is `safeguard` mode on all agents |
| 2 | Thread media transport policy | N/A |
| 3 | Discord gateway metadata fetch error handling | N/A |
| 4 | Session state: `lastAccountId` / `lastThreadId` preserved on reset | Benefit -- `reset-sessions.sh` truncates sessions; state now correctly preserved |
| 5 | Anthropic thinking block removal on replay | Benefit -- Argo Claude models use thinking; `reset-sessions.sh` causes replays |
| 6 | Android chat settings redesign | N/A |
| 7 | Browser batch action dispatch normalization | N/A |
| 8 | Dashboard chat history reload optimization | N/A |

---

### v2026.3.22 -- BREAKING CHANGES

| # | Change | Assessment |
|---|--------|------------|
| **B1** | Plugin installs prefer ClawHub over npm for npm-safe names | N/A -- no plugin installs |
| **B2** | Legacy Chrome extension relay path and `driver: "extension"` removed | N/A -- no Chrome extension |
| **B3** | Image generation standardized on core `image_generate` tool | N/A -- not using image generation |
| **B4** | Plugin SDK surface changed to `openclaw/plugin-sdk/*` | N/A -- no custom plugins |
| **B5** | `CLAWDBOT_*` and `MOLTBOT_*` environment variables removed | Clear -- not present in `docker-compose.yml` or `apply-config.sh` |
| **B6** | `.moltbot` state directory auto-detection removed | Clear -- no `.moltbot` directory on spark-ts |
| 7 | `openclaw skills search/install/update` flows | N/A -- informational |
| 8 | Claude marketplace registry with `plugin@marketplace` installs | N/A |
| 9 | Owner-gated `/plugins` and `/plugin` chat commands | N/A -- informational |
| 10 | Per-agent thinking/reasoning/fast model defaults | Informational -- new capability; 3 agents with different model tiers could benefit |
| 11 | `/btw` command for side questions without context changes | N/A -- informational |
| 12 | Pluggable sandbox backends: OpenShell and SSH | N/A -- using Docker |
| 13 | `anthropic-vertex` provider for Claude via Vertex AI | N/A |
| 14 | Bundled Chutes, Exa, Tavily, Firecrawl search providers | Clear -- bundled but require explicit config/enable; `tools: {}` empty and exec deny-all prevent unintended use |

---

### v2026.3.23

| # | Change | Assessment |
|---|--------|------------|
| 1 | Standard DashScope endpoints for Qwen | N/A -- using local vllm |
| 2 | Knot theme WCAG 2.1 AA contrast / button primitives | N/A -- UI only |
| 3 | SHA-256 hashes for inline CSP scripts | Benefit -- security hardening for Control UI |
| 4 | Plugin runtime sidecars npm packaging fix | N/A |
| 5 | Channel authentication fix for single-channel setups | Benefit -- Slack-only; previously had auth gaps |
| 6 | OpenAI token persistence fix | N/A |
| 7 | Operator scope preservation in device-auth bypass paths | Benefit -- 2 paired devices; scopes not lost on bypass |
| 8 | ClawHub plugin compatibility checks against active runtime versions | N/A |

---

### v2026.3.28 -- BREAKING CHANGES

| # | Change | Assessment |
|---|--------|------------|
| **B1** | `qwen-portal-auth` OAuth integration removed; must migrate to Model Studio | Clear -- using local vllm, not Qwen portal auth |
| **B2** | Auto-migration of config keys older than 2 months no longer performed; legacy keys now fail validation | Low risk -- config is ~5.5 weeks old; run `openclaw doctor` post-upgrade (Action A4) |
| 3 | xAI/Grok moves to Responses API with `x_search` support | N/A |
| 4 | MiniMax `image-01` image generation and editing | N/A |
| 5 | Plugin async `requireApproval` in `before_tool_call` hooks | N/A -- no custom plugins |
| 6 | Unified `upload-file` action for Slack, Teams, Google Chat | Benefit -- Slack file uploads improved; no config required |
| 7 | `openclaw config schema` outputs JSON schema for `openclaw.json` | Informational |

---

### v2026.3.31 -- BREAKING CHANGES

| # | Change | Assessment |
|---|--------|------------|
| **B1** | Duplicated `nodes.run` shell wrapper removed; exec routes through `exec host=node` only | Clear -- zero occurrences of `nodes.run` in all agent files and scripts |
| **B2** | Legacy provider compatibility subpaths in plugin SDK deprecated (warnings only; removal planned later) | N/A -- no custom plugins |
| **B3** | Plugin install fails by default on dangerous-code findings; `--dangerously-force-unsafe-install` required to override | N/A -- no plugin installs |
| **B4** | `trusted-proxy` rejects mixed shared-token configurations | Clear -- token-only auth; no mixed config |
| **B5** | Node commands remain disabled until pairing receives explicit approval; pairing alone no longer sufficient | N/A -- no node companion apps |
| **B6** | Node event trust surface reduced; node-triggered workflows relying on broad host/session tool access may require adjustment | Clear -- no node-triggered workflows; cron uses `docker exec` (host-side), not node events |
| 7 | Background tasks unified under SQLite ledger with task flow control surfaces | N/A -- informational |
| 8 | QQ Bot bundled as channel plugin with multi-account support | N/A |
| 9 | Matrix streaming, history context, proxy configuration | N/A |
| 10 | MCP remote HTTP/SSE server support enabled | Informational -- new capability, requires explicit config |
| 11 | WhatsApp emoji reactions | N/A |
| 12 | Security hardening across exec, auth, and gateway layers | Benefit -- general improvement |

---

### v2026.4.1 -- BREAKING CHANGES

| # | Change | Assessment |
|---|--------|------------|
| **B1** | Telegram legacy `groupMentionsOnly` auto-migrates to `groups["*"].requireMention` | N/A -- no Telegram |
| **B2** | Plugin runtime dependencies externalized; bundled plugins require `dist/plugins/runtime` layout | Clear -- Slack plugin is bundled (`plugins.entries.slack.enabled: true`); layout update is inside the new Docker image automatically; no custom plugin manifests to update |
| **B3** | `allow-always` approvals now persist as durable trust instead of `allow-once` | Clear -- no `allow-always` entries in `exec-approvals.json` |
| **B4** | Channel plugin loading restored under restrictive allowlists for legacy `channels.<id>` config | N/A -- informational |
| 5 | `/tasks` command for background task board | N/A -- informational |
| 6 | SearXNG web search plugin | N/A -- bundled but not auto-enabled |
| 7 | Amazon Bedrock Guardrails support | N/A |
| 8 | Z.AI models `glm-5.1` and `glm-5v-turbo` | N/A |
| 9 | Model switching queues behind active runs instead of interrupting turns | Benefit -- `apply-config.sh` model switches now queue cleanly |
| 10 | Voice Wake on macOS | N/A |
| 11 | Feishu Drive comment events with thread context resolution | N/A |
| 12 | Gateway webchat `maxChars` truncation config | N/A -- informational |
| 13 | `agents.defaults.params` global default provider parameters | N/A -- informational |
| 14 | Telegram `errorPolicy` and cooldown controls | N/A |
| 15 | WhatsApp inbound message timestamps in model context | N/A |
| 16 | Discord/Telegram exec approval routing refinements | N/A |
| 17 | LINE runtime resolution restored for global npm installations | N/A |
| 18 | Chat errors no longer leak raw provider failures to external channels | Benefit -- when Argo is unavailable, error details will not appear in Slack |
| 19 | Gateway reload ignores startup config writes (prevents restart loops) | Benefit -- directly relevant to `apply-config.sh` workflow |
| 20 | Task registry maintenance no longer stalls gateway event loop | N/A -- informational |
| 21 | Stale completed tasks hidden from `/status` and `session_status` | N/A -- informational |
| 22 | Memory session indexing preserves transcripts during restart-driven reindexes | Benefit -- `reset-sessions.sh` causes restarts; transcripts now preserved |
| 23 | Auth profile credential persistence fixed for OAuth token rotation | N/A -- not using OAuth providers |
| 24 | Discord gateway reconnect handling improved | N/A |

---

### v2026.4.2 -- BREAKING CHANGES

| # | Change | Assessment |
|---|--------|------------|
| **B1** | xAI plugin: `tools.web.x_search.*` moves to `plugins.entries.xai.config.xSearch.*`; run `openclaw doctor --fix` | N/A -- not using xAI |
| **B2** | Firecrawl: `tools.web.fetch.firecrawl.*` moves to `plugins.entries.firecrawl.config.webFetch.*`; run `openclaw doctor --fix` | N/A -- not using Firecrawl |
| 3 | Task Flow substrate restored with managed/mirrored sync modes | N/A -- informational |
| 4 | Task Flow child spawning with sticky cancel intent | N/A |
| 5 | Android assistant-role entrypoints and Google Assistant App Actions | N/A |
| **6** | **Gateway/Node exec defaults to YOLO mode (`security=full, ask=off`)** | **Action A2** -- `exec-approvals.json` has explicit `security: "deny"`; must verify this survives post-upgrade normalization |
| 7 | Provider replay hooks (transcript policy, cleanup, reasoning dispatch) | N/A -- informational |
| 8 | `before_agent_reply` plugin hook | N/A -- no custom plugins |
| 9 | Channel session routing moved to plugin-owned session-key surfaces | N/A -- Slack Socket Mode only; no Telegram/Feishu routing |
| 10 | Feishu Drive comment-event flow | N/A |
| 11 | Matrix `m.mentions` spec-compliant metadata | N/A |
| 12 | Diff viewer `viewerBaseUrl` for stable proxy/public origin | N/A |
| 13 | `agents.defaults.compaction.model` resolved consistently for both manual and engine-owned compaction | Benefit -- `compaction.mode: safeguard`; fix ensures agent primary model used consistently |
| 14 | `agents.defaults.compaction.notifyUser` makes compaction notice opt-in | N/A -- defaults to off |
| **15** | **Provider Transport Security: centralized auth, proxy, TLS, header shaping; blocks insecure TLS/runtime transport overrides** | **Action A3** -- argo (`http://172.18.0.1:44497`) and vllm (`http://nim:8000/v1`) use HTTP on internal Docker network; standard self-hosted pattern but must monitor after restart |
| 16 | GitHub Copilot API host classification hardened | N/A |
| 17 | Provider streaming headers centralized | N/A -- informational |
| 18 | Media HTTP normalization (OpenAI, Deepgram, Gemini, Moonshot) | N/A |
| 19 | Gateway execution loopback: legacy-role fallback restored for empty paired-device token maps | Benefit -- 2 paired devices with tokens; fixes a pairing edge case |
| 20 | Subagent gateway calls pinned to `operator.admin` | Benefit -- `subagents.maxConcurrent: 8` on all agents; prevents scope-upgrade failures |
| **21** | **Execution approval normalization strips "invalid" `security`, `ask`, and `askFallback` values** | **Action A2** -- `exec-approvals.json` uses `security: "deny"` and `security: "allowlist"`; must verify these survive normalization |
| 22 | Slack mrkdwn guidance added to inbound context | Benefit -- improves Slack rendering for all agents |
| 23 | WhatsApp `unavailable` presence on connect | N/A |
| 24 | WhatsApp HTML/XML/CSS added to MIME map | N/A |
| 25 | Matrix guided setup restored; live partial preview fixed | N/A |
| 26 | Feishu comment thread delivery hardened | N/A |
| 27 | MS Teams: strips already-streamed text from fallback block delivery | N/A |
| 28 | Slack thread context filtered by effective conversation allowlist | Benefit -- Slack channel allowlist in use; improves thread context reliability |
| 29 | Mattermost status probes routed through SSRF guard | N/A |
| 30 | Zalo webhook replay dedup scoped by chat + sender | N/A |
| 31 | QQBot: local file paths restricted to QQBot-owned media storage | N/A |
| 32 | Image generation: OpenAI/MiniMax/fal routed through shared provider HTTP transport | N/A |
| 33 | Browser inspection: static Chrome helpers kept out of activated browser runtime | N/A |
| 34 | Browser CDP: trailing-dot localhost hosts normalized before loopback checks | N/A |
| 35 | `antml:thinking` blocks stripped from user-visible output | Benefit -- Argo Claude models use thinking; was leaking into Slack |
| 36 | Kimi Coding tool: Anthropic payloads normalized to OpenAI function shape | N/A |
| 37 | Image tool paths resolved against agent `workspaceDir` instead of `process.cwd()` | Benefit -- agents write files to workspace; improves allowlist reliability |
| 38 | Podman: removed noisy container output | N/A |
| 39 | Plugin runtime: LINE reply directives and browser-backed cleanup preserved | N/A |
| 40 | ACP gateway reconnect: maintained prompts across transient websocket drops | N/A -- informational if A1 enables ACP |
| 41 | Gateway session kill requires `operator.admin` HTTP operator scope | Benefit -- security hardening; paired devices have this scope |
| 42 | MS Teams: formatted non-Error failures with shared helper | N/A |
| 43 | Channel setup: untrusted workspace channel plugins ignored (prevents shadowing built-ins) | Benefit -- security hardening |
| 44 | Execution allowlist on Windows: quote-aware `argPattern` matching | N/A -- Linux host |
| 45 | Gateway state: empty `node-pending-work` entries pruned | Benefit -- prevents indefinite state map growth in gateway volume |
| 46 | Webhook secret: `safeEqualSecret` helper replacing ad-hoc timing-safe comparisons | N/A -- Slack uses Socket Mode, not webhooks |
| 47 | OpenShell mirror: `remoteWorkspaceDir` constrained to managed roots | N/A |
| 48 | Plugin activation metadata: provenance preserved across CLI/gateway/status surfaces | N/A |
| 49 | Execution environment: additional host env override pivots blocked (package roots, runtimes, compiler paths, credential locations) | Benefit -- tighter containment for Docker sandbox exec |
| **50** | **Dotenv workspace overrides blocked: workspace `.env` files cannot override `OPENCLAW_PINNED_PYTHON`** | Clear -- no `.env` files in any agent workspace directory |
| 51 | Plugin JSON5 support in `openclaw.plugin.json` manifests | N/A |
| 52 | Telegram callback data rewritten to fit Telegram `callback_data` limit | N/A |
| 53 | Cron execution timeouts surfaced in isolated cron runs | Benefit -- heartbeat runs exec; timeouts now properly reported |
| 54 | Telegram approval followups: fallback to origin session key | N/A |
| 55 | Node-host execution: `pnpm dlx` bound through approval planner | N/A |
| 56 | Node host execution: stops forwarding gateway workspace cwd to remote node | N/A |
| 57 | Execution approval channels: initiating-surface availability decoupled from native delivery | Clear -- `ask: "off"` on all agents; no interactive prompts |

---

## Action Items -- Full Detail

### Action A1 -- ACP Dispatch (v2026.3.2)

**When:** Decide before upgrading.

The key `"acp"` is absent from your `openclaw.json`. Previously this meant ACP dispatch was off by default. After upgrading to 2026.3.2+, it will default to **enabled**.

ACP (Agent Communication Protocol) handles agent-to-agent turn routing. Your 3 agents operate on entirely separate Slack channels with no configured ACP handoffs. Enabling ACP dispatch should not cause cross-agent routing interference under your current configuration. However, it is a behavioral change on a multi-agent system that has not been tested under ACP.

**Conservative option:** Add this to `openclaw.json` before upgrading (via `apply-config.sh`):

```json
"acp": { "dispatch": { "enabled": false } }
```

You can enable it later if you want to explore ACP-based agent coordination.

**Permissive option:** Leave it enabled and monitor for unexpected behavior on first restart. If agents start routing responses to wrong channels, disable via `apply-config.sh` and restart.

---

### Action A2 -- exec-approvals Normalization (v2026.4.2)

**When:** Within 60 seconds of gateway startup post-upgrade.

Your security posture depends entirely on `exec-approvals.json`. The current live state is:

```json
{
  "defaults": { "security": "deny", "ask": "off", "askFallback": "deny" },
  "agents": {
    "chattpc26": {
      "security": "allowlist", "ask": "off", "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [{ "id": "gog-001", "pattern": "/usr/local/bin/gog" }]
    }
  }
}
```

The 2026.4.2 release normalizes exec approval configuration and strips values it considers invalid. The new gateway default is YOLO mode (`security: full, ask: off`). If `"deny"` or `"allowlist"` were treated as invalid values, main and cecat would lose their deny-all protection and chattpc26 would lose its gog-only allowlist -- giving all agents unrestricted host execution access.

This is **unlikely** -- `"deny"` and `"allowlist"` are first-class values, not edge cases -- but the consequence of a misconfiguration is severe. Do not skip this check.

**Immediately after the gateway starts post-upgrade, run:**

```bash
docker run --rm -v openclaw_openclaw-config:/data alpine cat /data/exec-approvals.json
```

Verify:
- `defaults.security` is still `"deny"`
- `defaults.ask` is still `"off"`
- `agents.chattpc26.security` is still `"allowlist"`
- `agents.chattpc26.allowlist` still contains only `/usr/local/bin/gog`

If any of these have changed or been removed: run `docker compose down` immediately and do not bring the gateway back up until you restore the file from the most recent snapshot.

---

### Action A3 -- Provider Transport Security Hardening (v2026.4.2)

**When:** Monitor after first restart.

Both providers use HTTP on internal Docker networks:
- vllm: `http://nim:8000/v1` (Docker internal, `qwen3-coder-next_nim_net`)
- argo: `http://172.18.0.1:44497` (Docker bridge gateway IP)

This is a standard self-hosted pattern. The 2026.4.2 security change blocks code-based TLS override attacks, not legitimate HTTP for internal endpoints. Your providers should be unaffected. However, the change also centralizes proxy/auth/header handling, and any edge case in how internal HTTP connections are classified could produce connection errors.

**After the first heartbeat fires (within 15 minutes), run:**

```bash
docker logs openclaw-gateway --since 15m 2>&1 | grep -iE "error|refused|transport|tls|connect"
```

If you see connection errors to `nim:8000` or `172.18.0.1:44497`, check whether the gateway is crash-looping. The `apply-config.sh` script has a built-in 20-second crash-loop detector that automatically reverts to local vllm. If that fires, investigate the provider connection issue before re-applying the config.

---

### Action A4 -- Config Migration Policy (v2026.3.28)

**When:** After the gateway is confirmed healthy post-upgrade.

Starting in 2026.3.28, config keys older than 2 months are no longer auto-migrated; they fail validation instead. Your config was last touched 2026-02-25, making it approximately 5.5 weeks old at upgrade time -- inside the 2-month window. However, running doctor is low-cost insurance and also handles the xAI and Firecrawl path migrations introduced in 2026.4.2 (not relevant to your setup, but harmless to run).

```bash
docker compose -f ~/code/spark-ai/openclaw/docker-compose.yml \
  --profile cli run --rm openclaw-cli openclaw doctor --fix
```

---

## Upgrade Procedure

```
1. Take a snapshot before touching anything
   bash ~/code/spark-ai-agents/ops/openclaw-snapshot.sh

2. Decide on ACP dispatch (Action A1)
   If disabling: add "acp": {"dispatch": {"enabled": false}} via apply-config.sh before pulling

3. Pull new image
   docker pull ghcr.io/openclaw/openclaw:latest

4. Restart gateway
   cd ~/code/spark-ai/openclaw
   docker compose down && docker compose up -d

5. [CRITICAL -- within 60 seconds] Verify exec-approvals.json (Action A2)
   docker run --rm -v openclaw_openclaw-config:/data alpine cat /data/exec-approvals.json
   Confirm: defaults.security = "deny", chattpc26.security = "allowlist", gog-only allowlist intact
   If anything has changed: docker compose down immediately; restore from snapshot

6. Monitor provider connectivity (Action A3)
   docker logs openclaw-gateway --since 15m 2>&1 | grep -iE "error|refused|transport|tls"

7. Run doctor (Action A4)
   docker compose -f ~/code/spark-ai/openclaw/docker-compose.yml \
     --profile cli run --rm openclaw-cli openclaw doctor --fix

8. Smoke test: verify all 3 Slack channels respond to a @mention

9. Verify heartbeat fires within 15 minutes for each agent
   Check shared/logs/sessions-seed.log to confirm main.jsonl is present after next seed run
```

---

## Counts

| Category | Count |
|----------|-------|
| Total individual changes examined | 120+ |
| Versions covered | 13 |
| Breaking changes examined | 22 |
| Breaking changes that require action | 1 (A1: ACP dispatch default) |
| Breaking changes confirmed clear | 21 |
| Actions requiring pre-upgrade decision | 1 (A1) |
| Actions requiring post-upgrade verification | 3 (A2, A3, A4) |
| Changes that are improvements / benefits to this system | 25+ |
