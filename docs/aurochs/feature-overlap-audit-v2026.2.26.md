# Aurochs × OpenClaw — Feature Overlap Audit (v2026.2.26)

> **Purpose**: Identify what Aurochs custom-built that OpenClaw now does natively after the v2026.2.20–2.26 updates
> **Date**: February 27, 2026
> **Sources**: `aurochs-deploy/api/domains/`, `aurochs-clean/src/config/`, `agent-zero-port-plan.md`, `openclaw-vs-aurochs-comparison.md`

---

## Executive Summary

OpenClaw v2026.2.20–2.26 added several capabilities that **overlap with or supersede** features Aurochs custom-built. The three Agent Zero features planned for porting are also affected:

| Verdict                                          | Count | Action                                   |
| ------------------------------------------------ | ----- | ---------------------------------------- |
| 🔴 **REDUNDANT** — OpenClaw now does it natively | 4     | Retire Aurochs code, use native          |
| 🟡 **ENHANCED** — Aurochs adds governance on top | 6     | Keep Aurochs layer, rewire to use native |
| 🟢 **UNIQUE** — OpenClaw doesn't have this       | 5     | Keep as-is, core Aurochs value           |

---

## 🔴 REDUNDANT — Retire These

### 1. Knowledge Steward → OpenClaw Memory Search

| Aspect    | Aurochs `knowledge-steward.js` | OpenClaw Native (v2026.2.20)                                        |
| --------- | ------------------------------ | ------------------------------------------------------------------- |
| Storage   | SQLite FTS5                    | SQLite hybrid (vector + FTS5)                                       |
| Search    | FTS5 + LIKE fallback           | Hybrid BM25 + vector + **MMR** + **temporal decay**                 |
| Embedding | ❌ None                        | ✅ 4 providers (OpenAI, Gemini, Voyage, local) + **fallback chain** |
| Chunking  | Character-based (~1000 chars)  | Token-based (400 tokens, 80 overlap)                                |

**Action**: Delete `knowledge-steward.js` (426 LOC). Place enterprise documents in `tenants/{tenant}/memory/`. Use OpenClaw's `memorySearch` with new features:

- `fallback: "local"` — resilience if Gemini is down
- `query.hybrid.mmr.enabled: true` — deduplicate results
- `query.hybrid.temporalDecay.enabled: true` — boost recent knowledge

> Already recommended in [openclaw-vs-aurochs-comparison.md](file:///Users/jerrisontiong/aurochs-deploy/docs/archive/openclaw-vs-aurochs-comparison.md). Overlap 2.

---

### 2. A2A Protocol (Agent Zero Feature 3) → OpenClaw `tools.agentToAgent`

| Aspect      | Agent Zero Plan (`a2a-queue.js`) | OpenClaw Native (v2026.2.20)               |
| ----------- | -------------------------------- | ------------------------------------------ |
| Transport   | Custom HTTP queue via SQLite     | Native WebSocket `sessions_send`           |
| Discovery   | Manual tenant-to-tenant ACL      | `tools.agentToAgent.allow: ["pattern"]`    |
| Depth       | Request/response only            | Full session interaction + spawn           |
| Concurrency | Custom queue                     | Native `maxConcurrent: 8`                  |
| Ping-pong   | Not planned                      | `session.agentToAgent.maxPingPongTurns: 5` |

**Action**: **Do NOT build** the planned `api/domains/a2a/` module (660 LOC estimated). Use OpenClaw's native:

```json5
{
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["main", "worker-*"], // Hive mind pattern
    },
  },
  session: {
    agentToAgent: { maxPingPongTurns: 5 }, // Safety limit
  },
}
```

> [!WARNING]
> **Governance gap**: OpenClaw's `agentToAgent` has no RPN gating. If you need governance-aware A2A (RPN check before cross-agent messages), keep a **thin governance gate** (~50 LOC) as an MSC hook that intercepts `sessions_send` events. Don't build the full queue/ACL infrastructure.

---

### 3. Tool Loop Detection → OpenClaw `tools.loopDetection`

| Aspect    | Aurochs `ops/` rate-limiter loop guard | OpenClaw Native (v2026.2.20)                                    |
| --------- | -------------------------------------- | --------------------------------------------------------------- |
| Detection | Basic repeat counter                   | 3 detectors: `genericRepeat`, `knownPollNoProgress`, `pingPong` |
| Action    | Block after N repeats                  | Warning → critical → **circuit breaker** (30 calls)             |
| Scope     | Per-tenant rate limiting               | Per-session tool call history                                   |
| Config    | Custom                                 | `tools.loopDetection.enabled: true`                             |

**Action**: Retire the custom loop guard in the rate limiter. Use OpenClaw's native loop detection which is more sophisticated.

---

### 4. Tool Profiles → OpenClaw `tools.profile`

| Aspect    | Aurochs custom tool allow/deny    | OpenClaw Native (v2026.2.20)                                   |
| --------- | --------------------------------- | -------------------------------------------------------------- |
| Profiles  | Manual per-skill allow/deny lists | `minimal` / `coding` / `messaging` / `full` presets            |
| Per-model | Not supported                     | `tools.byProvider: { "google/gemini": { profile: "coding" } }` |
| Merge     | List concatenation                | Profile + `alsoAllow` additive merging                         |

**Action**: Replace custom tool gating with OpenClaw's native profiles. Keep governance-specific tool restrictions through `deny` lists.

---

## 🟡 ENHANCED — Keep Aurochs Layer, Rewire

### 5. LLM Routing (Aurochs `model-router.js`) + OpenClaw Provider Routing

| Aspect                | Aurochs (14 files in `api/domains/llm/`)       | OpenClaw Native              |
| --------------------- | ---------------------------------------------- | ---------------------------- |
| Model selection       | 4-tier router (elite/background/general/embed) | Alias-based, model catalog   |
| Circuit breaker       | ✅ Per-provider with cooldown                  | ❌ No native circuit breaker |
| Budget enforcement    | ✅ Token wallet reserve/settle                 | ❌ No native budgeting       |
| PII scrubbing         | ✅ Regex scrub/restore pipeline                | ❌ No native PII             |
| Key management        | ✅ Portkey gateway (routing + spend limits)    | Env vars only                |
| Portkey gateway       | ✅ Proxy through Portkey                       | ❌ Not integrated            |
| Stakes classification | ✅ HIGH/LOW preflight                          | ❌ No native stakes          |
| Policy engine         | ✅ 8 deterministic evaluators                  | ❌ No native policy          |

**Action**: **Keep the full Aurochs LLM domain**. It wraps OpenClaw's LLM calls with governance that OpenClaw will never add natively (budget, PII, Portkey routing, stakes). The circuit breaker is unique value.

But **rewire** the context tokens setting:

```json5
// In openclaw.json — leverage new 200k default
{ agents: { defaults: { contextTokens: 200000 } } }
```

---

### 6. Sub-agent Architecture → OpenClaw Hive Mind

| Aspect        | Aurochs MSC multi-agent                     | OpenClaw Native (v2026.2.20)           |
| ------------- | ------------------------------------------- | -------------------------------------- |
| Spawning      | MSC plugin dispatches to separate processes | Native `sessions_spawn`                |
| Concurrency   | Custom queue                                | `subagents.maxConcurrent: 8`           |
| Nesting       | Not supported                               | `maxSpawnDepth: 2` (sub-sub-agents)    |
| Worker models | Manual per-skill                            | `subagents.model: "cheaper-model"`     |
| Visibility    | Full access                                 | `sessions.visibility: "tree"` (scoped) |
| Archiving     | Manual cleanup                              | `archiveAfterMinutes: 60`              |

**Action**: **Rewire the hive mind** to use OpenClaw's native sub-agent spawning. Keep the MSC governance layer but delegate execution to OpenClaw's `sessions_spawn`:

- Main agent: uses primary model
- Workers: use `subagents.model` (cheaper)
- Workers can spawn sub-workers: `maxSpawnDepth: 2`
- Auto-cleanup: `archiveAfterMinutes: 60`

---

### 7. Memory Consolidation (Agent Zero Feature 1) — Still Needed, Partially Covered

| Aspect                | Agent Zero Plan (`memory-consolidation.js`) | OpenClaw Native (v2026.2.20)                             |
| --------------------- | ------------------------------------------- | -------------------------------------------------------- |
| Duplicate detection   | Embedding similarity + shingle Jaccard      | ✅ Hybrid search + **MMR** (dedup results, not memories) |
| LLM merge decisions   | ✅ LLM judges merge/replace/update/skip     | ❌ No LLM consolidation                                  |
| Auto-trigger on write | ✅ After every ingest                       | ❌ No auto-consolidation                                 |
| Keyword extraction    | ✅ LLM extracts tags                        | ❌ No keyword tagging                                    |

**Action**: **Still build this** (~200 LOC) BUT leverage OpenClaw's improved memory search for the similarity step instead of custom embedding calls:

- Use OpenClaw's `memorySearch` with `provider: "gemini"` + `fallback: "local"` for finding similar memories
- Keep the LLM decision step (Aurochs-unique value)
- Delegate the actual merge to existing `memory-refactor.js`

---

### 8. Exec Approval Forwarding → OpenClaw `approvals.exec`

| Aspect     | Aurochs notification system            | OpenClaw Native (v2026.2.20)             |
| ---------- | -------------------------------------- | ---------------------------------------- |
| Forwarding | Custom notifier → email/webhook outbox | Native forward to Telegram/Discord/Slack |
| Filtering  | SLA timers, DARE escalation            | `agentFilter`, `sessionFilter`           |
| Targets    | HMAC-signed webhooks                   | Channel + `to` + `accountId`             |

**Action**: Use OpenClaw's native `approvals.exec` for simple forwarding. Keep Aurochs' notification system for **governance escalations** (DARE, SLA breaches, NC-CAPA alerts) which are not covered by OpenClaw.

---

### 9. Security Guards → OpenClaw `tools.elevated` + `fs.workspaceOnly`

| Aspect            | Aurochs (15 guard scripts in `tools/guards/`) | OpenClaw Native (v2026.2.20)                |
| ----------------- | --------------------------------------------- | ------------------------------------------- |
| File access       | Custom scope guard                            | `tools.fs.workspaceOnly: true`              |
| Elevated mode     | Custom per-domain                             | `tools.elevated.allowFrom` per channel      |
| Apply patch guard | Not supported                                 | `tools.exec.applyPatch.workspaceOnly: true` |
| Import guards     | ✅ Domain boundary enforcement                | ❌ Not in OpenClaw                          |
| Commit guards     | ✅ Message format enforcement                 | ❌ Not in OpenClaw                          |
| Privacy guard     | ✅ Privacy surface analysis                   | ❌ Not in OpenClaw                          |

**Action**: Use OpenClaw's native `fs.workspaceOnly` and `elevated.allowFrom`. **Keep** the Aurochs-specific guards (domain boundary, commit message, privacy surface, cycle guard) — these are code quality tools, not agent security.

> [!NOTE]
> **v2026.2.26 update**: OpenClaw's `fs.workspaceOnly` now also rejects hardlink aliases. This closes a gap that existed in v2026.2.20.

---

### 10. Channel Allowlist Enforcement → OpenClaw Native Validation (v2026.2.26) 🆕

| Aspect                  | Aurochs channel security    | OpenClaw Native (v2026.2.26)                                                |
| ----------------------- | --------------------------- | --------------------------------------------------------------------------- |
| DM allowlist validation | Custom guard checks         | **Native**: `dmPolicy="allowlist"` rejects empty `allowFrom` at config load |
| Group isolation         | Custom DM≠group check       | **Native**: Group allowlists isolated from DM pairing store                 |
| Reaction/event gating   | Not implemented             | **Native**: All event types require sender authorization                    |
| Name matching           | Disabled by policy          | **Native**: Requires explicit `allowNameMatching: true`                     |
| Streaming config        | Custom `streamMode` mapping | **Native**: Auto-migration from `streamMode` to `streaming`                 |

**Action**: **Rewire Aurochs channel security** to rely on OpenClaw's native validation. Remove any custom empty-allowlist guards — OpenClaw now enforces this at startup. Keep the MSC-level governance (DARE escalation, audit logging) on top.

---

## 🟢 UNIQUE — OpenClaw Doesn't Have This

| #   | Feature                                       | Where in Aurochs                           | Why OpenClaw Won't Add It                |
| --- | --------------------------------------------- | ------------------------------------------ | ---------------------------------------- |
| 1   | **MSC Database** (35+ tables, ISO compliance) | `api/msc-db.js` + `platform/msc.db`        | Enterprise governance, not agent runtime |
| 2   | **Budget Enforcement** (token wallet)         | `api/domains/billing/wallet.js`            | Cost control is business logic           |
| 3   | **PII Scrubbing**                             | `api/pii-scrubber.js`                      | Compliance is business logic             |
| 4   | **PDCA / DARE / NC-CAPA Governance**          | `api/domains/governance/`                  | ISO framework is business domain         |
| 5   | **Multi-Tenancy**                             | Docker containers + `api/domains/tenancy/` | OpenClaw is designed single-tenant       |

> [!NOTE]
> **v2026.2.26 update**: OpenClaw's new `SecretRef` system (`env`/`file`/`exec` sources) provides structured secret resolution. Aurochs manages API keys via **Portkey gateway** (routing + spend limits per tenant), not per-tenant encrypted storage. `SecretRef` can load Portkey-provisioned keys into OpenClaw via `{ source: "env", id: "PORTKEY_API_KEY" }`.

---

## MCP Bridge (Agent Zero Feature 2) — Still Needed

| Aspect          | Agent Zero Plan               | OpenClaw Native (v2026.2.20)  |
| --------------- | ----------------------------- | ----------------------------- |
| MCP Client      | ✅ Full stdio/SSE/HTTP        | ❌ Browser tool dispatch only |
| Tool discovery  | ✅ Registry + MSC mapping     | ❌ No MCP tool discovery      |
| Governance gate | ✅ RPN check before execution | ❌ No governance              |

**Action**: **Still build this**. OpenClaw still lacks proper MCP server support. The MCP Bridge plan is still valid and provides unique value (governance-gated MCP tools).

---

## Action Summary

### Retire (Save ~1,100 LOC + avoid ~660 LOC new code)

| Item                              | LOC Saved       | Replacement                             |
| --------------------------------- | --------------- | --------------------------------------- |
| `knowledge-steward.js`            | 426 LOC deleted | OpenClaw `memorySearch` (native)        |
| A2A Protocol (planned, not built) | 660 LOC avoided | OpenClaw `tools.agentToAgent` (native)  |
| Custom loop guard                 | ~50 LOC deleted | OpenClaw `tools.loopDetection` (native) |
| Custom tool profiles              | ~30 LOC deleted | OpenClaw `tools.profile` (native)       |

### Rewire (Keep code but change integration points)

| Item                | Change                                                                 |
| ------------------- | ---------------------------------------------------------------------- |
| Sub-agent spawning  | Use native `sessions_spawn` instead of custom MSC dispatch             |
| Exec approvals      | Use native `approvals.exec` for simple cases, keep DARE for governance |
| Filesystem security | Use native `fs.workspaceOnly`, keep Aurochs domain guards              |
| Context tokens      | Set `contextTokens: 200000` to leverage new defaults                   |
| Bootstrap           | Set `bootstrapTotalMaxChars: 150000` for larger SOUL.md                |

### Still Build (Agent Zero port)

| Item                            | Why                                                       |
| ------------------------------- | --------------------------------------------------------- |
| Memory Consolidation (~200 LOC) | LLM-driven dedup is unique (use native similarity search) |
| MCP Bridge (~930 LOC)           | OpenClaw still lacks proper MCP support                   |

### Keep As-Is (Aurochs unique value)

| Item                           | Why                                                         |
| ------------------------------ | ----------------------------------------------------------- |
| Full LLM domain (14 files)     | Budget, PII, Portkey routing, circuit breaker — none native |
| Full MSC database (35+ tables) | ISO governance — will never be in OpenClaw                  |
| Full governance domain         | DARE, PDCA, NC-CAPA — enterprise differentiation            |
| PII scrubber                   | Compliance requirement — never in upstream                  |
| Multi-tenancy                  | Docker isolation — OpenClaw is single-tenant by design      |

---

## Config Update Required

Update `openclaw.json` to leverage v2026.2.26 features:

```diff
 {
   agents: { defaults: {
-    contextTokens: 64000,
+    contextTokens: 200000,
+    bootstrapTotalMaxChars: 150000,
     subagents: {
-      maxConcurrent: 1,
+      maxConcurrent: 8,
+      maxSpawnDepth: 2,
+      model: "google/gemini-3-flash",
     }
   } },
+  tools: {
+    agentToAgent: { enabled: true, allow: ["main", "worker-*"] },
+    loopDetection: { enabled: true },
+    fs: { workspaceOnly: true },
+    profile: "coding",
+  },
+  approvals: {
+    exec: { enabled: true, mode: "session" }
+  },
+  channels: {
+    telegram: {
-      streamMode: "partial",
+      streaming: "partial",
+      dmPolicy: "allowlist",
+      allowFrom: [12345678],  // REQUIRED in v2026.2.26
+    }
+  },
 }
```

> [!CAUTION]
> **v2026.2.26 action required**: If you have `dmPolicy: "allowlist"` in any channel config, ensure `allowFrom` is non-empty. OpenClaw will fail to start otherwise.

> [!TIP]
> **Portkey + SecretRef**: Load Portkey-provisioned API keys via `SecretRef` using `{ source: "env", id: "PORTKEY_API_KEY" }` for clean integration.
