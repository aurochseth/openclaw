# OpenClaw Settings — Plain English Guide

> **Purpose**: Practical reference for every OpenClaw setting. What it does, why you'd touch it, the cost tradeoff, and whether to just turn it on.
> **Updated**: February 2026 (v2026.2.26)
> **Source**: TypeScript type definitions in [`src/config/`](file:///Users/jerrisontiong/aurochs-clean/src/config/)

---

## How Settings Work

All settings live in your `openclaw.json`. You can also set them at runtime via WebSocket:

```bash
# Via CLI
openclaw config set agents.defaults.thinkingDefault high
openclaw config get agents.defaults.thinkingDefault

# Via WebSocket
{ "method": "config.set", "params": { "key": "agents.defaults.thinkingDefault", "value": "high" } }
{ "method": "config.patch", "params": { "patch": { "agents.defaults.contextTokens": 200000 } } }
```

Every setting has a **dot-path** like `agents.defaults.heartbeat.every`. The path maps directly to a nested JSON key.

---

## Settings That Affect Intelligence (Costs More Tokens)

These make the agent **smarter** but increase your token bill.

### `agents.defaults.thinkingDefault`

**What it does**: Tells the LLM to reason internally before answering. The model generates hidden "thinking" tokens — you don't see them, but you're billed for them as output tokens (the expensive kind).

| Level     | What the model does                       | Token cost impact       |
| --------- | ----------------------------------------- | ----------------------- |
| `off`     | Answer immediately, no internal reasoning | Cheapest                |
| `minimal` | Brief internal thought                    | +10-20% output tokens   |
| `low`     | Think a bit harder                        | +30-50% output tokens   |
| `medium`  | Extended reasoning chain                  | +100-200% output tokens |
| `high`    | Maximum thinking budget ("ultrathink")    | +300-500% output tokens |
| `xhigh`   | Beyond max (GPT-5.2/Codex only)           | +500%+ output tokens    |

**Resolution order** (first one wins):

1. Inline directive on the message (`/t high solve this bug`)
2. Session override (`/think:medium` as a standalone message)
3. This config setting
4. Fallback: `low` for reasoning-capable models, `off` otherwise

**Should you turn it on?** ✅ Set to `low`. It's the sweet spot — marginal cost increase for noticeably better answers. Only go higher for specific tasks using inline `/t high`.

```json5
{ agents: { defaults: { thinkingDefault: "low" } } }
```

---

### `agents.defaults.memorySearch`

**What it does**: Before every response, the agent semantically searches your `MEMORY.md` and `memory/*.md` files. Matching snippets are injected into the conversation as extra context. The agent "remembers" things from past sessions.

**How it increases tokens**:

1. **Indexing** (one-time): Your memory files are converted to vector embeddings via an API call. This happens once per file change, not per message.
2. **Per message** (ongoing): Retrieved snippets are added to the input context. Typically **+500 to 2,000 input tokens per turn**, depending on how many results are injected.

| Sub-setting                  | What it does                                                                                 | Token/cost impact                                                   |
| ---------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `enabled`                    | Master toggle                                                                                | Off = no extra tokens                                               |
| `provider`                   | Embedding API (`openai`, `gemini`, `local`, `voyage`)                                        | `gemini` = free tier; `local` = free but slower                     |
| `sources`                    | What to index: `["memory"]` or `["memory", "sessions"]`                                      | Adding `sessions` = much more indexed content, more embedding calls |
| `extraPaths`                 | Additional directories/files to index                                                        | More paths = more embeddings + more retrieval tokens                |
| `fallback`                   | Fallback provider if primary fails (`"openai"`, `"gemini"`, `"local"`, `"voyage"`, `"none"`) | Resilience at no extra cost                                         |
| `query.maxResults`           | How many snippets to inject per turn                                                         | More results = more input tokens per message                        |
| `query.hybrid.enabled`       | Combine keyword search (BM25) with vector search                                             | Better recall, negligible extra cost                                |
| `query.hybrid.mmr`           | MMR re-ranking for result diversity                                                          | Avoids redundant snippets                                           |
| `query.hybrid.temporalDecay` | Boost recent memories over older ones                                                        | Better relevance                                                    |
| `store.cache.enabled`        | Cache embeddings in SQLite                                                                   | Saves re-embedding unchanged content                                |
| `sync.watch`                 | Watch files for changes and auto-reindex                                                     | More embedding calls if files change often                          |

**Should you turn it on?** ✅ **Yes** if your agent has accumulated knowledge it needs to recall across sessions.

**Recommended config**:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini", // free tier embedding API
        fallback: "local", // NEW: fallback if Gemini is down
        query: {
          maxResults: 5,
          hybrid: { enabled: true, mmr: { enabled: true, lambda: 0.7 } },
        },
        cache: { enabled: true },
      },
    },
  },
}
```

---

### `agents.defaults.contextTokens`

**What it does**: Caps the conversation context window. Without this, the LLM uses as much context as the model allows.

> [!IMPORTANT]
> **v2026.2.20 change**: The default context window is now **200,000 tokens** (`DEFAULT_CONTEXT_TOKENS = 200_000`). Models can declare up to **1M+ token** context windows via `contextWindow` in their provider config. This means you can set `contextTokens: 1000000` if your model supports it (e.g., Gemini 2.5 Pro with 1M context).

**Tradeoff**: Lower cap = cheaper per turn but the agent "forgets" earlier conversation faster (triggers compaction sooner). Higher cap = more expensive but longer conversational memory.

**Should you set it?** ✅ Yes. Set based on your model's actual capability and budget:

| Model             | Max Context | Recommended `contextTokens` |
| ----------------- | ----------- | --------------------------- |
| Gemini 2.5 Flash  | 1,048,576   | `200000` – `500000`         |
| Gemini 2.5 Pro    | 1,048,576   | `200000` – `1000000`        |
| Claude Sonnet 4.6 | 200,000     | `200000`                    |
| GPT-5.2           | 128,000     | `64000` – `128000`          |

```json5
{ agents: { defaults: { contextTokens: 200000 } } }
```

---

## Settings That Save Tokens

These make things **cheaper** without making the agent dumber.

### `agents.defaults.contextPruning`

**What it does**: Automatically removes old tool outputs (file contents, command results, web fetches) from the context after they expire.

| Sub-setting          | What it does                                  |
| -------------------- | --------------------------------------------- |
| `mode: "cache-ttl"`  | Enable pruning with a time-based expiry       |
| `ttl: "1h"`          | Remove tool outputs older than 1 hour         |
| `keepLastAssistants` | Always keep the last N assistant messages     |
| `softTrim.maxChars`  | NEW: Max chars for soft-trimmed results       |
| `hardClear.enabled`  | NEW: Enable hard clearing of very old results |
| `tools.allow/deny`   | NEW: Per-tool pruning control                 |

**Token impact**: Can **save 5,000-20,000+ tokens per turn** in long sessions.

**Should you turn it on?** ✅ **Absolutely yes.** Free performance.

```json5
{
  agents: {
    defaults: {
      contextPruning: { mode: "cache-ttl", ttl: "1h" },
    },
  },
}
```

---

### `agents.defaults.compaction`

**What it does**: When the conversation fills the context window, the agent summarizes older messages to make room.

| Sub-setting          | What it does                                                       |
| -------------------- | ------------------------------------------------------------------ |
| `mode: "safeguard"`  | Aggressive compaction that prevents context overflow (recommended) |
| `mode: "default"`    | Standard compaction                                                |
| `reserveTokensFloor` | Minimum free space to keep after compaction                        |
| `maxHistoryShare`    | Max % of context for history (default 0.5 = 50%)                   |
| `keepRecentTokens`   | NEW: Budget for cut-point selection                                |

**Should you change it?** The default (`safeguard`) is already good.

---

### `agents.defaults.compaction.memoryFlush`

**What it does**: Before compacting, the agent writes important information to `MEMORY.md` so nothing is lost.

**Should you turn it on?** ✅ Already on by default. Leave it on.

---

## Settings That Cost Zero Tokens (Display/Routing/Safety)

### `agents.defaults.verboseDefault`

| Level  | What you see                               |
| ------ | ------------------------------------------ |
| `off`  | Just the final reply                       |
| `on`   | Tool names + paths/commands as they happen |
| `full` | Tool names + their outputs too             |

**Should you turn it on?** ✅ Yes, set to `on`. Free transparency.

---

### `agents.defaults.elevatedDefault`

| Level  | What it means                                   |
| ------ | ----------------------------------------------- |
| `off`  | Agent can never run elevated commands           |
| `ask`  | Agent asks you before running elevated commands |
| `on`   | Agent can run elevated commands without asking  |
| `full` | Full elevated access (dangerous)                |

**Should you turn it on?** ⚠️ Leave at `off` or `ask` unless fully automated.

---

### `agents.defaults.blockStreamingDefault` / `humanDelay` / `typingMode`

Display-only settings. Zero token impact. See previous version for details.

---

## ⭐ Multi-Agent / Hive Mind Settings (NEW in v2026.2.20)

> [!IMPORTANT]
> These settings are critical for the Aurochs hive mind architecture (main agent + worker sub-agents).

### `agents.defaults.subagents` — Sub-agent spawning

**What it does**: Controls spawned sub-agents (via `sessions_spawn`). Sub-agents are independent agent sessions the main agent creates for parallel work.

| Sub-setting           | Default          | What it does                                         |
| --------------------- | ---------------- | ---------------------------------------------------- |
| `maxConcurrent`       | **8** ← ⬆️ was 1 | Max parallel sub-agents (global lane)                |
| `maxSpawnDepth`       | 1                | How deep can sub-agents nest (1 = no sub-sub-agents) |
| `maxChildrenPerAgent` | 5                | Max active children per parent session               |
| `archiveAfterMinutes` | 60               | Auto-archive idle sub-agents                         |
| `model`               | Same as parent   | Default model for sub-agents (can be cheaper!)       |
| `thinking`            | Same as parent   | Default thinking level for sub-agents                |

> [!TIP]
> **Hive mind pattern**: Set `maxConcurrent: 8` (now the default!) and use a cheaper model for workers:
>
> ```json5
> {
>   agents: {
>     defaults: {
>       subagents: {
>         maxConcurrent: 8,
>         maxSpawnDepth: 2, // Allow sub-sub-agents
>         maxChildrenPerAgent: 5,
>         model: "google/gemini-3-flash", // Workers use cheaper model
>         thinking: "low",
>       },
>     },
>   },
> }
> ```

**Token impact**: Each sub-agent is a full LLM session. Running 8 concurrent sub-agents = 8× the token cost. Use cheaper models for workers.

---

### `agents.defaults.maxConcurrent` — Main agent concurrency

**Default**: **4** (NEW — was 1)

Controls how many concurrent agent runs can happen across all conversations. Set higher for multi-tenant setups.

---

### `tools.agentToAgent` — Direct agent-to-agent calling ⭐

**What it does**: Enables agents to **directly message other agents** by ID. This is the key setting for the hive mind architecture where a main agent delegates to specialized worker agents.

```json5
{
  tools: {
    agentToAgent: {
      enabled: true, // Enable cross-agent messaging
      allow: ["worker-*"], // Allow messaging to agents matching pattern
    },
  },
}
```

| Sub-setting | Default | What it does                               |
| ----------- | ------- | ------------------------------------------ |
| `enabled`   | `false` | Master toggle for agent-to-agent messaging |
| `allow`     | `[]`    | Allowlist of target agent IDs or patterns  |

**Token impact**: Each agent-to-agent message is a full turn in the target agent's session.

**For hive mind**: ✅ Enable this and set `allow` to match your worker agent IDs:

```json5
{
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["main", "worker-research", "worker-coding", "worker-analysis"],
    },
  },
}
```

---

### `tools.sessions.visibility` — Cross-session access control

**What it does**: Controls which sessions tools can interact with.

| Level   | What the agent can see                                                |
| ------- | --------------------------------------------------------------------- |
| `self`  | Only its own session                                                  |
| `tree`  | Own session + sub-agents it spawned (default)                         |
| `agent` | Any session belonging to its agent ID                                 |
| `all`   | Any session in the system (cross-agent still requires `agentToAgent`) |

**For hive mind**: Set to `tree` (default) so the main agent can see its spawned workers but workers can't see each other.

---

### `session.agentToAgent` — Ping-pong limits

Controls back-and-forth between agents:

| Sub-setting        | Default | What it does                                       |
| ------------------ | ------- | -------------------------------------------------- |
| `maxPingPongTurns` | 5       | Max back-and-forth turns (0-5) before forcing stop |

---

## ⭐ Security Hardening (NEW in v2026.2.20, expanded v2026.2.26)

### `tools.exec` — Command execution security

| Sub-setting                | What it does                                      |
| -------------------------- | ------------------------------------------------- |
| `security: "deny"`         | Can't run anything                                |
| `security: "allowlist"`    | Only pre-approved commands                        |
| `security: "full"`         | Run anything (dangerous!)                         |
| `ask: "on-miss"`           | Ask before running unknown commands               |
| `ask: "always"`            | Always ask before running any command             |
| `safeBins`                 | Stdin-only binaries exempt from allowlist         |
| `applyPatch.enabled`       | Enable apply_patch for OpenAI models              |
| `applyPatch.workspaceOnly` | Restrict patches to workspace dir (default: true) |
| `applyPatch.allowModels`   | Per-model allowlist for apply_patch               |

> [!IMPORTANT]
> **v2026.2.26 security**: Exec approvals now bind to exact argv identity — trailing-space executable path swaps no longer reuse mismatched approvals. Symlink `cwd` paths are rejected and path-like executables are canonicalized before spawn.

### `tools.elevated` — Elevated mode gateway

| Sub-setting | What it does                                           |
| ----------- | ------------------------------------------------------ |
| `enabled`   | Master toggle (default: true)                          |
| `allowFrom` | Per-channel allowlists for who can trigger `/elevated` |

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        telegram: [12345678], // Only this Telegram user
        discord: ["987654321"], // Only this Discord user
      },
    },
  },
}
```

### `approvals.exec` — Exec approval forwarding

**What it does**: When the agent needs approval to run a command, forward the request to specified chat channels (e.g., send approval request to your Telegram).

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "both", // "session" | "targets" | "both"
      agentFilter: ["main"], // Only forward for main agent
      targets: [
        {
          channel: "telegram",
          to: "12345678",
        },
      ],
    },
  },
}
```

### `tools.loopDetection` — Circuit breaker

**What it does**: Detects and stops stuck tool-call loops (agent calling the same tool repeatedly with no progress).

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
      warningThreshold: 10, // Warn after 10 repeats
      criticalThreshold: 20, // Block after 20 repeats
      globalCircuitBreakerThreshold: 30, // Hard stop after 30
      detectors: {
        genericRepeat: true, // Detect identical calls
        knownPollNoProgress: true, // Known polling patterns
        pingPong: true, // Alternating patterns
      },
    },
  },
}
```

### `tools.fs.workspaceOnly` — Filesystem restriction

Restrict all file read/write/edit tools to the agent workspace directory:

```json5
{ tools: { fs: { workspaceOnly: true } } }
```

> [!IMPORTANT]
> **v2026.2.26**: Hardlink aliases are now rejected in `workspaceOnly` and `applyPatch.workspaceOnly` boundary checks — in-workspace hardlinks to out-of-workspace files are blocked. Symlink-parent escapes in browser temp/download paths are also caught.

### Tool Profiles — Preset permission bundles (NEW)

Instead of manually listing allowed tools, use a profile:

| Profile     | What's included                             |
| ----------- | ------------------------------------------- |
| `minimal`   | Basic tools only (read, list)               |
| `coding`    | Full coding tools (read, write, edit, exec) |
| `messaging` | Messaging tools (send, channel ops)         |
| `full`      | Everything enabled                          |

```json5
{ tools: { profile: "coding", deny: ["sandbox"] } }
```

### `logging.redactSensitive` — Token redaction (NEW)

| Setting   | What it does                                        |
| --------- | --------------------------------------------------- |
| `"off"`   | No redaction                                        |
| `"tools"` | Redact sensitive tokens in tool summaries (default) |

### Diagnostics / OTEL (NEW)

OpenTelemetry integration for observability:

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      traces: true,
      metrics: true,
      logs: true,
    },
  },
}
```

---

## ⭐ Toggle Features (NEW capabilities to enable/disable)

These are features you can now **toggle on/off** that weren't available before:

| Feature                  | Config Path                                                       | Default | What it does                       |
| ------------------------ | ----------------------------------------------------------------- | ------- | ---------------------------------- |
| Agent-to-Agent messaging | `tools.agentToAgent.enabled`                                      | `false` | Direct cross-agent communication   |
| Tool loop detection      | `tools.loopDetection.enabled`                                     | `false` | Circuit breaker for stuck tools    |
| Exec approval forwarding | `approvals.exec.enabled`                                          | `false` | Forward approvals to chat channels |
| Apply patch tool         | `tools.exec.applyPatch.enabled`                                   | `false` | Patch-based file editing           |
| Media understanding      | `tools.media.image.enabled`                                       | `false` | Image understanding via LLM        |
| Audio understanding      | `tools.media.audio.enabled`                                       | `false` | Audio transcription/understanding  |
| Video understanding      | `tools.media.video.enabled`                                       | `false` | Video understanding                |
| Link understanding       | `tools.links.enabled`                                             | `false` | Auto-process links in messages     |
| Web search               | `tools.web.search.enabled`                                        | auto    | Requires API key                   |
| Web fetch                | `tools.web.fetch.enabled`                                         | `true`  | URL content fetching               |
| Broadcast messaging      | `tools.message.broadcast.enabled`                                 | `true`  | One-to-many message delivery       |
| Cross-context sends      | `tools.message.crossContext.allowAcrossProviders`                 | `false` | Send across channel providers      |
| Memory search            | `agents.defaults.memorySearch.enabled`                            | `true`  | Vector memory search               |
| Session memory indexing  | `agents.defaults.memorySearch.experimental.sessionMemory`         | `false` | Index session transcripts          |
| Hybrid search MMR        | `agents.defaults.memorySearch.query.hybrid.mmr.enabled`           | `false` | MMR re-ranking                     |
| Temporal decay           | `agents.defaults.memorySearch.query.hybrid.temporalDecay.enabled` | `false` | Boost recent memories              |
| Context pruning          | `agents.defaults.contextPruning.mode`                             | `"off"` | Auto-remove stale tool outputs     |
| Cron jobs                | `cron.enabled`                                                    | varies  | Scheduled tasks                    |
| Heartbeat                | `agents.defaults.heartbeat.every`                                 | `"30m"` | Periodic agent wakeup              |
| Diagnostics/OTEL         | `diagnostics.enabled`                                             | `false` | Observability tracing              |

---

## Settings That Affect Autonomy

### `agents.defaults.heartbeat`

| Sub-setting                 | What it does                                               | Default                      |
| --------------------------- | ---------------------------------------------------------- | ---------------------------- |
| `every`                     | How often to run                                           | `30m`                        |
| `activeHours.start` / `end` | Only run during these hours                                | All day                      |
| `model`                     | Use a different (cheaper) model for heartbeats             | Same as main                 |
| `prompt`                    | Custom heartbeat prompt                                    | _"Read HEARTBEAT.md..."_     |
| `target`                    | Where to deliver results                                   | `last` (last active channel) |
| `to`                        | NEW: Specific delivery target (E.164, chat ID, :topic:NNN) | —                            |
| `accountId`                 | NEW: Account ID for multi-account channels                 | —                            |
| `includeReasoning`          | Show reasoning chain in heartbeat replies                  | `false`                      |
| `suppressToolErrorWarnings` | NEW: Suppress tool error warnings                          | `false`                      |

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        model: "google/gemini-2.5-flash-lite",
        activeHours: { start: "09:00", end: "22:00" },
      },
    },
  },
}
```

---

### `agents.defaults.sandbox`

| Sub-setting              | What it does                                      |
| ------------------------ | ------------------------------------------------- |
| `mode: "off"`            | No sandboxing (everything runs on host)           |
| `mode: "non-main"`       | Sub-agents are sandboxed, main agent runs on host |
| `mode: "all"`            | Everything runs in containers                     |
| `workspaceAccess`        | `"none"` / `"ro"` / `"rw"`                        |
| `scope`                  | `"session"` / `"agent"` / `"shared"`              |
| `sessionToolsVisibility` | NEW: `"spawned"` (default) or `"all"`             |
| `docker.image`           | Custom Docker image                               |
| `docker.memory`          | Container memory limit                            |
| `docker.cpus`            | CPU limit                                         |
| `docker.network`         | Network mode                                      |
| `browser.enabled`        | Sandbox browser (CDP/VNC)                         |
| `prune`                  | NEW: Auto-prune sandbox containers                |

---

## Settings for Tools

### `tools.exec`

| Sub-setting                | What it does                                           |
| -------------------------- | ------------------------------------------------------ |
| `host`                     | Where commands run (`sandbox`, `gateway`, `node`)      |
| `security`                 | `"deny"` / `"allowlist"` / `"full"`                    |
| `ask`                      | `"off"` / `"on-miss"` / `"always"`                     |
| `node`                     | NEW: Default node binding for `exec.host=node`         |
| `safeBins`                 | NEW: Stdin-only binaries exempt from allowlist         |
| `backgroundMs`             | Auto-background after this many ms                     |
| `timeoutSec`               | Kill the process after this many seconds               |
| `notifyOnExit`             | Alert when background commands finish                  |
| `notifyOnExitEmptySuccess` | NEW: Also notify on empty success exits                |
| `approvalRunningNoticeMs`  | NEW: Notice when approved exec runs long (default 10s) |
| `applyPatch`               | NEW: apply_patch for OpenAI models                     |

---

### `tools.web.search`

| Sub-setting            | What it does                              |
| ---------------------- | ----------------------------------------- |
| `provider`             | `"brave"`, `"perplexity"`, or `"grok"`    |
| `search.maxResults`    | How many results (1-10)                   |
| `grok.apiKey`          | xAI API key for Grok search               |
| `grok.model`           | Grok model (default: `"grok-4-1-fast"`)   |
| `grok.inlineCitations` | Markdown citation links in results        |
| `perplexity.apiKey`    | Perplexity/OpenRouter API key             |
| `perplexity.baseUrl`   | Custom API base URL                       |
| `perplexity.model`     | Model (default: `"perplexity/sonar-pro"`) |

---

### `tools.media` — Media Understanding (NEW)

**What it does**: Process images, audio, and video attachments using LLMs.

```json5
{
  tools: {
    media: {
      concurrency: 2,
      image: {
        enabled: true,
        models: [{ provider: "google", model: "gemini-2.5-flash" }],
        maxBytes: 10485760,
      },
      audio: {
        enabled: true,
        models: [{ provider: "google", model: "gemini-2.5-flash" }],
      },
    },
  },
}
```

---

### `tools.sessions.visibility`

| Level   | What the agent can see                        |
| ------- | --------------------------------------------- |
| `self`  | Only its own session                          |
| `tree`  | Own session + sub-agents it spawned (default) |
| `agent` | Any session belonging to its agent ID         |
| `all`   | Any session in the system                     |

---

## Settings for Workspace & System Prompt

### Bootstrap Limits

> [!IMPORTANT]
> **v2026.2.20 change**: `bootstrapTotalMaxChars` default is now **150,000** (was 24,000). This means your SOUL.md and workspace files can be much larger.

| Setting                  | Default                     | What it does               |
| ------------------------ | --------------------------- | -------------------------- |
| `bootstrapMaxChars`      | 20,000                      | Max chars per file         |
| `bootstrapTotalMaxChars` | **150,000** ← ⬆️ was 24,000 | Max total across all files |

### Time & Envelope Settings

| Setting             | Default | What it does                           |
| ------------------- | ------- | -------------------------------------- |
| `userTimezone`      | Host TZ | IANA timezone shown in system prompt   |
| `timeFormat`        | `auto`  | 12-hour or 24-hour format              |
| `envelopeTimezone`  | `utc`   | Timestamp timezone in message wrappers |
| `envelopeTimestamp` | `on`    | Show absolute timestamps               |
| `envelopeElapsed`   | `on`    | Show elapsed time since last message   |

---

## Session Management

### `session`

| Sub-setting                     | What it does                                    | Default      |
| ------------------------------- | ----------------------------------------------- | ------------ |
| `scope`                         | `per-sender` or `global`                        | `per-sender` |
| `dmScope`                       | How DM sessions are tracked                     | `main`       |
| `identityLinks`                 | NEW: Map platform identities to canonical peers | —            |
| `reset.mode`                    | `daily` or `idle`                               | —            |
| `resetByType`                   | NEW: Different reset rules for DM/group/thread  | —            |
| `resetByChannel`                | NEW: Per-channel reset overrides                | —            |
| `maintenance.pruneAfter`        | Delete old session data                         | `30d`        |
| `maintenance.maxEntries`        | Cap total session entries                       | 500          |
| `maintenance.rotateBytes`       | NEW: Rotate sessions.json at size limit         | `10mb`       |
| `agentToAgent.maxPingPongTurns` | Max agent-to-agent back-and-forth               | 5            |

---

## Channel Settings (Telegram, Discord, Slack, etc.)

### Common per-channel settings

| Setting                  | What it does                                                                       |
| ------------------------ | ---------------------------------------------------------------------------------- |
| `dmPolicy`               | Who can DM the agent: `pairing`, `allowlist`, `open`, `disabled`                   |
| `allowFrom`              | List of allowed user IDs                                                           |
| `groupPolicy`            | How the agent behaves in groups: `open`, `disabled`, `allowlist`                   |
| `groupAllowFrom`         | Separate allowlist for groups (falls back to `allowFrom` by default)               |
| `configWrites`           | Allow the channel to modify config (default: true)                                 |
| `streaming`              | Live typing preview (replaces `streamMode`): `off`, `partial`, `block`, `progress` |
| `defaultTo`              | Default delivery target (e.g., a chat ID for heartbeat output)                     |
| `webhookPort`            | Local webhook listener port (Telegram, default: 8787)                              |
| `network.dnsResultOrder` | DNS resolution order: `ipv4first` or `verbatim`                                    |

> [!CAUTION]
> **v2026.2.26 breaking change**: `dmPolicy="allowlist"` now **requires** at least one entry in `allowFrom`. Empty `allowFrom` with `allowlist` policy is a **config validation error** at startup. This affects all channels (Telegram, Discord, Slack, Signal, etc.).

> [!WARNING]
> **v2026.2.26 security**: Group allowlists are now **isolated from DM pairing store**. DM pairing approvals no longer inherit into group authorization. Use explicit `groupAllowFrom` for group access. Reaction, pin, member, and message-subtype events now require sender authorization across all channels.

> [!IMPORTANT]
> **Streaming migration**: `streamMode` is deprecated. OpenClaw auto-migrates it to `streaming` at config load. Update your config to use `streaming` directly.

---

## Recommended Starting Config (v2026.2.26)

A sensible starting point that leverages the latest features:

```json5
{
  agents: {
    defaults: {
      // Model selection
      model: { primary: "google/gemini-3-flash-preview" },

      // Thinking: low is the sweet spot
      thinkingDefault: "low",

      // Show tool activity (free)
      verboseDefault: "on",

      // Context: take advantage of large windows
      contextTokens: 200000,

      // Remove stale tool outputs (saves tokens)
      contextPruning: { mode: "cache-ttl", ttl: "1h" },

      // Memory (costs ~1k tokens/turn but agent remembers things)
      memorySearch: {
        enabled: true,
        provider: "gemini",
        fallback: "local",
        query: {
          maxResults: 5,
          hybrid: { enabled: true, mmr: { enabled: true } },
        },
      },

      // Heartbeat with cheap model
      heartbeat: {
        every: "30m",
        model: "google/gemini-2.5-flash-lite",
        activeHours: { start: "09:00", end: "22:00" },
      },

      // Sub-agents for hive mind
      subagents: {
        maxConcurrent: 8,
        maxSpawnDepth: 2,
        model: "google/gemini-3-flash",
        thinking: "low",
      },

      // Workspace
      workspace: "~/.openclaw/workspace",
      bootstrapMaxChars: 20000,
      bootstrapTotalMaxChars: 150000,
    },
  },

  // Session management
  session: {
    scope: "per-sender",
    reset: { mode: "idle", idleMinutes: 120 },
    maintenance: { pruneAfter: "30d", maxEntries: 500 },
    agentToAgent: { maxPingPongTurns: 5 },
  },

  // Tool security & features
  tools: {
    profile: "coding",
    exec: { security: "allowlist", ask: "on-miss" },
    fs: { workspaceOnly: true },
    sessions: { visibility: "tree" },
    agentToAgent: { enabled: true },
    loopDetection: { enabled: true },
    elevated: { enabled: true },
  },

  // Exec approval forwarding
  approvals: {
    exec: { enabled: true, mode: "session" },
  },

  // Channels: ensure allowFrom is set if using allowlist
  channels: {
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: [12345678], // REQUIRED in v2026.2.26
      streaming: "partial", // replaces streamMode
    },
  },
}
```

---

## Quick Reference: Token Cost Matrix

| Setting                        | Direction                    | Magnitude            | Worth it?                    |
| ------------------------------ | ---------------------------- | -------------------- | ---------------------------- |
| `thinkingDefault: low`         | 📈 More output tokens        | +30-50%              | ✅ Yes — better answers      |
| `thinkingDefault: high`        | 📈 More output tokens        | +300-500%            | ⚠️ Only for complex tasks    |
| `memorySearch.enabled`         | 📈 More input tokens         | +500-2k/turn         | ✅ Yes — agent remembers     |
| `contextTokens: 200000`        | 📈 Allows larger context     | Up to 200k/turn      | ✅ Yes — leverages model     |
| `contextPruning`               | 📉 Fewer input tokens        | -5k-20k/turn         | ✅ Absolutely — free savings |
| `compaction.memoryFlush`       | 📈 One extra turn on compact | +1 turn occasionally | ✅ Yes — prevents data loss  |
| `heartbeat (every 30m)`        | 📈 ~32 turns/day             | +32 turns/day        | ⚠️ Only if needed            |
| `heartbeat.model: flash-lite`  | 📉 Cheap heartbeat turns     | -90% vs main model   | ✅ Yes if using heartbeats   |
| `subagents (×8)`               | 📈 8× token cost             | Major increase       | ⚠️ Use cheaper worker models |
| `agentToAgent`                 | 📈 Extra agent turns         | Variable             | ✅ Essential for hive mind   |
| `loopDetection`                | —                            | Zero                 | ✅ Free safety net           |
| `verboseDefault: on`           | —                            | Zero                 | ✅ Free transparency         |
| `sandbox`                      | —                            | Zero                 | ✅ Security, no token cost   |
| `bootstrapTotalMaxChars: 150k` | 📈 More system prompt        | +per turn            | Larger SOUL.md ok now        |
| `diagnostics/otel`             | —                            | Zero                 | ✅ Observability             |

---

## Version Change Summary

### v2026.2.20 Changes

| What Changed                      | Old                        | New                      | Impact                                                                                                    |
| --------------------------------- | -------------------------- | ------------------------ | --------------------------------------------------------------------------------------------------------- |
| `DEFAULT_CONTEXT_TOKENS`          | —                          | 200,000                  | Models can use large context windows                                                                      |
| `bootstrapTotalMaxChars`          | 24,000                     | **150,000**              | 6× larger workspace files in system prompt                                                                |
| `DEFAULT_AGENT_MAX_CONCURRENT`    | 1                          | **4**                    | 4× more concurrent agent runs                                                                             |
| `DEFAULT_SUBAGENT_MAX_CONCURRENT` | 1                          | **8**                    | 8× more concurrent sub-agents                                                                             |
| `tools.agentToAgent`              | N/A                        | **NEW**                  | Direct cross-agent messaging                                                                              |
| `tools.loopDetection`             | N/A                        | **NEW**                  | Circuit breaker for stuck tools                                                                           |
| `approvals.exec`                  | N/A                        | **NEW**                  | Forward approvals to chat channels                                                                        |
| `tools.exec.applyPatch`           | N/A                        | **NEW**                  | Patch-based editing                                                                                       |
| `tools.media`                     | N/A                        | **NEW**                  | Image/audio/video understanding                                                                           |
| `tools.web.search.grok`           | N/A                        | **NEW**                  | Grok-powered web search                                                                                   |
| `diagnostics.otel`                | N/A                        | **NEW**                  | OpenTelemetry integration                                                                                 |
| Config filename                   | `openclaw.default.json` ok | `openclaw.json` required | Breaking change                                                                                           |
| `gateway.bind`                    | Raw IPs                    | Named modes only         | Breaking change                                                                                           |
| Deprecated keys removed           | Present                    | **Deleted**              | `systemPrompt`, `controlUi.requireAuth`, `rateLimit`, `skills.paths`, `sandbox` (old), `channels.*.token` |

### v2026.2.26 Changes

| What Changed                               | Old                        | New                                                    | Impact                                             |
| ------------------------------------------ | -------------------------- | ------------------------------------------------------ | -------------------------------------------------- |
| `dmPolicy="allowlist"` + empty `allowFrom` | Silently accepted          | **Config validation error**                            | ⚠️ Breaking: must set `allowFrom`                  |
| `streamMode`                               | Primary key                | **Deprecated** → auto-migrated to `streaming`          | Update config to use `streaming`                   |
| `streaming`                                | N/A                        | `off` / `partial` / `block` / `progress`               | New `progress` mode added                          |
| Group allowlists                           | Inherited DM pairing store | **Isolated** — explicit `groupAllowFrom` only          | ⚠️ Breaking for groups relying on DM pair store    |
| Allowlist name matching                    | Enabled by default         | **Requires `allowNameMatching: true`**                 | Tighter security                                   |
| `defaultTo` (Telegram)                     | N/A                        | **NEW**                                                | Default delivery target per account                |
| `webhookPort` (Telegram)                   | Hardcoded 8787             | **Configurable**                                       | Port flexibility                                   |
| `network.dnsResultOrder`                   | N/A                        | `ipv4first` / `verbatim`                               | DNS resolution control                             |
| Secret references                          | Env vars only (`"${VAR}"`) | **`SecretRef`**: `env` / `file` / `exec` sources       | Structured secrets (Vault, file, env)              |
| `openai-codex-responses`                   | N/A                        | **NEW** model API                                      | Codex model support                                |
| Exec approval binding                      | Loose                      | **Exact argv identity match**                          | Symlink/whitespace swaps blocked                   |
| `fs.workspaceOnly`                         | Symlink check only         | **+hardlink rejection**                                | In-workspace hardlinks to out-of-workspace blocked |
| Reaction/event authorization               | Not gated                  | **Sender auth required** across all channels           | Reactions, pins, member events all gated           |
| Gemini model IDs                           | Exact match                | **Forward-compat fallback** for `gemini-3.1-*-preview` | Graceful future model handling                     |
