# Agent Architecture Summary: OpenClaw Analysis & Secure Design

> **Purpose**: Reference document for building a secure AI agent using Claude Agent SDK + custom MCP server.
> **Source**: Analysis of OpenClaw codebase (485K lines) and security research.
> **Date**: 2026-01-31

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [OpenClaw Capabilities Worth Replicating](#openclaw-capabilities-worth-replicating)
3. [OpenClaw Security Risks to Eliminate](#openclaw-security-risks-to-eliminate)
4. [Recommended Architecture](#recommended-architecture)
5. [MCP Server Design](#mcp-server-design)
6. [Memory System Design](#memory-system-design)
7. [Scheduling System Design](#scheduling-system-design)
8. [Security Model](#security-model)
9. [Implementation Roadmap](#implementation-roadmap)
10. [Code References from OpenClaw](#code-references-from-openclaw)

---

## Executive Summary

### What OpenClaw Is
OpenClaw (formerly Clawdbot/Moltbot) is a 485,000-line TypeScript codebase providing:
- Multi-channel AI messaging gateway (WhatsApp, Telegram, Discord, Slack, Signal, iMessage, 20+ channels)
- Persistent memory with vector search
- Session management and context compaction
- Autonomous scheduling (cron, heartbeat, webhooks)
- Tool execution with optional sandboxing

### Why Not Use It
- **Attack surface**: 485K lines, 106 dependencies
- **Known CVEs**: CVE-2025-49596 (CVSS 9.4), CVE-2025-6514 (CVSS 9.6), CVE-2025-52882 (CVSS 8.8)
- **Shell execution by default**: One prompt injection away from full system compromise
- **Exposed instances in the wild**: Researchers found hundreds via Shodan with full credential exposure

### The Alternative
Build a minimal secure agent using:
- **Claude Agent SDK**: Handles agentic loop, tool execution, context compaction
- **Custom MCP Server**: Exposes only the tools you need, nothing more
- **~500-1000 lines** vs 485,000

---

## OpenClaw Capabilities Worth Replicating

### 1. Memory System (Priority: High)

**What OpenClaw Does**:
- `MEMORY.md` - Curated long-term memory (decisions, preferences, facts)
- `memory/YYYY-MM-DD.md` - Daily logs (append-only running notes)
- SQLite + vector embeddings for semantic search
- Hybrid search: BM25 (exact match) + vector similarity (semantic)
- Pre-compaction memory flush (saves important context before summarizing)

**Key Parameters**:
- Chunk size: ~400 tokens with ~80 token overlap
- Hybrid weights: 70% vector, 30% BM25
- Embedding cache to avoid re-embedding unchanged content

**Files to Study**:
- `src/memory/internal.ts` - Chunking algorithm (`chunkMarkdown()`)
- `src/memory/manager-search.ts` - Hybrid search implementation
- `src/memory/manager.ts` - Index management, file watching

### 2. Context Compaction (Priority: High)

**What OpenClaw Does**:
- Default context window: 200K tokens
- When approaching limit: summarize older messages
- Keep recent ~40% of context intact
- Reserve ~20K tokens for response
- Trigger memory flush before compaction

**Files to Study**:
- `src/agents/compaction.ts` - Compaction logic
- `src/agents/context-window-guard.ts` - Window management

### 3. Session Management (Priority: Medium)

**What OpenClaw Does**:
- Session isolation by key: `agent:<agentId>:<channel>:dm:<peerId>`
- Daily reset (default 4 AM local time)
- Idle timeout reset (optional)
- Per-user isolation in group chats
- Persistent transcripts as JSONL files

**Session Scopes**:
- `main` - Single shared session
- `per-peer` - One session per person
- `per-channel-peer` - One session per person per channel
- `per-account-channel-peer` - Full isolation

**Files to Study**:
- `src/config/sessions/session-key.ts` - Key generation
- `src/config/sessions/types.ts` - Session schema

### 4. Scheduling (Priority: High)

**What OpenClaw Does**:

**Cron Jobs**:
- Recurring or one-shot scheduled tasks
- Isolated or main-session execution
- Delivery to messaging channels
- Model/thinking overrides per job
- Auto-delete after one-shot runs

**Heartbeat**:
- Periodic check-in (default: 30 minutes)
- Reads `HEARTBEAT.md` for instructions
- Active hours restriction (e.g., 8am-10pm)
- Suppresses "nothing to report" responses

**Webhooks**:
- `POST /hooks/wake` - Enqueue system event
- `POST /hooks/agent` - Trigger isolated agent run
- Token authentication required
- Custom mappings for external services (Gmail, GitHub)

**Files to Study**:
- `docs/automation/webhook.md` - Webhook design
- `docs/automation/cron-jobs.md` - Cron design
- `docs/gateway/heartbeat.md` - Heartbeat design

### 5. Tool Execution Model (Priority: High)

**What OpenClaw Does**:
- Tool allowlists/denylists
- Per-command approval prompts
- Sandboxed execution (Docker containers)
- Elevated mode escape hatch
- Tool groups: `group:runtime`, `group:fs`, `group:web`, etc.

**What to Replicate**:
- Allowlist-only model (no denylists - too easy to miss something)
- Input validation in every tool
- Audit logging of all tool calls
- Rate limiting

---

## OpenClaw Security Risks to Eliminate

### Critical Risks

| Risk | OpenClaw Exposure | Your Solution |
|------|-------------------|---------------|
| **Shell execution** | `exec` tool runs arbitrary bash | **No shell tools at all** |
| **Arbitrary file access** | `read`/`write`/`edit` tools access any file | **Only expose specific paths via MCP** |
| **Prompt injection → RCE** | LLM can be tricked into running commands | **No dangerous tools to invoke** |
| **Exposed admin panels** | Gateway often deployed without auth | **Auth required, Tailscale/VPN only** |
| **Credential theft** | Creds stored in predictable paths | **Env vars only, no file storage** |
| **Memory poisoning** | Attacker writes malicious instructions | **Append-only memory, review before execution** |

### Known CVEs to Avoid

| CVE | Description | Your Mitigation |
|-----|-------------|-----------------|
| CVE-2025-49596 | Unauthenticated admin access | Auth required on all endpoints |
| CVE-2025-6514 | Command injection via mcp-remote | No shell execution |
| CVE-2025-52882 | Arbitrary file access | Allowlisted paths only |

### Attack Vectors from Research

1. **Prompt injection via email** (Snyk): Malicious email tricks agent into forwarding data
2. **Memory poisoning** (Palo Alto): Fragmented payloads assembled over time
3. **Supply chain via skills** (SOC Prime): Poisoned packages on ClawdHub
4. **Exposed Shodan instances** (Bitdefender): API keys, tokens, full access found

---

## Recommended Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     YOUR SYSTEM                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Claude Agent SDK                          │  │
│  │  • Agentic loop (battle-tested)                       │  │
│  │  • Context compaction (built-in)                      │  │
│  │  • Tool permissions (canUseTool callback)             │  │
│  │  • MCP client                                         │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │                                   │
│                          │ MCP Protocol (stdio)              │
│                          ▼                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Your MCP Server (~300-500 lines)          │  │
│  │                                                        │  │
│  │  Memory Tools:                                         │  │
│  │  • memory_read(file) - Read MEMORY.md or daily log    │  │
│  │  • memory_append(content) - Append to daily log       │  │
│  │  • memory_search(query, limit) - Hybrid vector search │  │
│  │                                                        │  │
│  │  Scheduling Tools:                                     │  │
│  │  • cron_add(id, cron, prompt, deliver_to)             │  │
│  │  • cron_list()                                         │  │
│  │  • cron_remove(id)                                     │  │
│  │                                                        │  │
│  │  Your Automations:                                     │  │
│  │  • automation_<name>() - Pre-baked workflows          │  │
│  │                                                        │  │
│  │  NO: shell, arbitrary file, network fetch              │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │                                   │
│                          ▼                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              SQLite Database                           │  │
│  │  • memory_chunks (content, embedding, file, lines)    │  │
│  │  • memory_fts (FTS5 full-text index)                  │  │
│  │  • cron_jobs (id, cron, prompt, deliver_to)           │  │
│  │  • audit_log (timestamp, tool, params, result)        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Filesystem (workspace/)                   │  │
│  │  • MEMORY.md - Curated long-term memory               │  │
│  │  • memory/YYYY-MM-DD.md - Daily logs                  │  │
│  │  • HEARTBEAT.md - Heartbeat instructions (optional)   │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Runners (separate processes)              │  │
│  │  • cron-runner.ts - Triggers agent on schedule        │  │
│  │  • webhook-server.ts - HTTP endpoints for triggers    │  │
│  │  • heartbeat in cron-runner (interval-based)          │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## MCP Server Design

### Tool Inventory

```typescript
// Memory Tools
tool("memory_read", { file: string }) → string
tool("memory_append", { content: string }) → { ok: true, file: string }
tool("memory_search", { query: string, limit?: number }) → SearchResult[]
tool("memory_curate", { content: string }) → { ok: true } // Write to MEMORY.md

// Scheduling Tools
tool("cron_add", { id: string, cron: string, prompt: string, deliver_to?: string }) → Job
tool("cron_list", {}) → Job[]
tool("cron_remove", { id: string }) → { ok: true }
tool("cron_run_now", { id: string }) → { ok: true, queued: true }

// Your Pre-baked Automations
tool("automation_daily_summary", {}) → string
tool("automation_<your_workflow>", { ...params }) → Result
```

### Security Constraints

```typescript
// In every tool handler:

// 1. Validate inputs
if (!isValidPath(file)) throw new Error("Invalid path");
if (content.length > MAX_CONTENT_SIZE) throw new Error("Content too large");

// 2. Restrict paths
const ALLOWED_PATHS = ["MEMORY.md", /^memory\/\d{4}-\d{2}-\d{2}\.md$/];
if (!ALLOWED_PATHS.some(p => typeof p === "string" ? p === file : p.test(file))) {
  throw new Error("Path not allowed");
}

// 3. Rate limit
if (rateLimiter.isExceeded("memory_append", 10, "1h")) {
  throw new Error("Rate limit exceeded");
}

// 4. Audit log
await db.run(`INSERT INTO audit_log VALUES (?, ?, ?, ?)`,
  [Date.now(), "memory_append", JSON.stringify(params), "success"]);
```

### Explicit Non-Goals (DO NOT BUILD)

```typescript
// NEVER expose these tools:
tool("shell_exec", ...) // NO
tool("file_read", { path: any }) // NO - only specific memory paths
tool("file_write", { path: any }) // NO
tool("http_fetch", { url: any }) // NO - only if absolutely needed, allowlist domains
tool("browser_*", ...) // NO
tool("process_*", ...) // NO
```

---

## Memory System Design

### File Structure

```
workspace/
├── MEMORY.md                    # Curated long-term memory (agent writes important facts)
├── HEARTBEAT.md                 # Optional heartbeat instructions
└── memory/
    ├── 2026-01-29.md           # Daily log
    ├── 2026-01-30.md           # Daily log
    └── 2026-01-31.md           # Daily log (today)
```

### MEMORY.md Format

```markdown
# User Preferences
- Prefers concise responses
- Timezone: America/Los_Angeles
- Primary language: English

# Key Facts
- Project: Building secure AI agent
- Stack: Claude Agent SDK + MCP server
- Infrastructure: Hetzner VPS behind Tailscale

# Decisions
- 2026-01-31: Chose to build custom MCP server instead of using OpenClaw
- 2026-01-31: Will use sqlite-vec for vector storage
```

### Daily Log Format

```markdown
# 2026-01-31

## Session Notes
- Discussed OpenClaw architecture
- Analyzed security risks
- Designed MCP server structure

## Tasks Completed
- Created architecture summary document

## Pending
- Build MCP server
- Implement memory tools
```

### SQLite Schema

```sql
-- Memory chunks for vector search
CREATE TABLE memory_chunks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file TEXT NOT NULL,              -- "MEMORY.md" or "memory/2026-01-31.md"
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  content TEXT NOT NULL,
  content_hash TEXT NOT NULL,      -- SHA-256 for deduplication
  embedding BLOB,                  -- Float32 array
  created_at INTEGER DEFAULT (unixepoch())
);

-- Full-text search index
CREATE VIRTUAL TABLE memory_fts USING fts5(
  content,
  content='memory_chunks',
  content_rowid='id'
);

-- Triggers to keep FTS in sync
CREATE TRIGGER memory_fts_insert AFTER INSERT ON memory_chunks BEGIN
  INSERT INTO memory_fts(rowid, content) VALUES (new.id, new.content);
END;

-- Embedding cache (avoid re-embedding unchanged content)
CREATE TABLE embedding_cache (
  content_hash TEXT PRIMARY KEY,
  embedding BLOB NOT NULL,
  created_at INTEGER DEFAULT (unixepoch())
);
```

### Chunking Algorithm (from OpenClaw)

```typescript
function chunkMarkdown(content: string, options = { tokens: 400, overlap: 80 }): Chunk[] {
  const lines = content.split("\n");
  const maxChars = options.tokens * 4; // ~4 chars per token
  const overlapChars = options.overlap * 4;

  const chunks: Chunk[] = [];
  let current: Line[] = [];
  let currentChars = 0;

  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    const lineSize = line.length + 1;

    if (currentChars + lineSize > maxChars && current.length > 0) {
      // Flush current chunk
      chunks.push({
        startLine: current[0].lineNo,
        endLine: current[current.length - 1].lineNo,
        text: current.map(l => l.text).join("\n"),
        hash: sha256(text)
      });

      // Keep overlap
      let overlapAcc = 0;
      const kept: Line[] = [];
      for (let j = current.length - 1; j >= 0 && overlapAcc < overlapChars; j--) {
        kept.unshift(current[j]);
        overlapAcc += current[j].text.length + 1;
      }
      current = kept;
      currentChars = kept.reduce((sum, l) => sum + l.text.length + 1, 0);
    }

    current.push({ lineNo: i + 1, text: line });
    currentChars += lineSize;
  }

  // Flush final chunk
  if (current.length > 0) {
    chunks.push({ /* ... */ });
  }

  return chunks;
}
```

### Hybrid Search (from OpenClaw)

```typescript
async function hybridSearch(query: string, limit: number = 8): Promise<SearchResult[]> {
  const queryEmbedding = await embed(query);

  // Vector search (semantic similarity)
  const vectorResults = await db.all(`
    SELECT id, file, start_line, end_line, content,
           vec_distance_cosine(embedding, ?) as distance
    FROM memory_chunks
    WHERE embedding IS NOT NULL
    ORDER BY distance ASC
    LIMIT ?
  `, [queryEmbedding, limit * 4]); // Over-fetch for reranking

  // BM25 search (exact tokens)
  const bm25Results = await db.all(`
    SELECT rowid as id, rank as score
    FROM memory_fts
    WHERE memory_fts MATCH ?
    ORDER BY rank
    LIMIT ?
  `, [query, limit * 4]);

  // Combine with weights (OpenClaw default: 70% vector, 30% BM25)
  const combined = mergeResults(vectorResults, bm25Results, {
    vectorWeight: 0.7,
    textWeight: 0.3
  });

  return combined.slice(0, limit);
}
```

---

## Scheduling System Design

### Cron Jobs Table

```sql
CREATE TABLE cron_jobs (
  id TEXT PRIMARY KEY,
  cron_expression TEXT NOT NULL,     -- "0 7 * * *" or null for one-shot
  run_at INTEGER,                    -- Unix timestamp for one-shot
  prompt TEXT NOT NULL,
  deliver_to TEXT,                   -- "telegram:123456" or "email:foo@bar.com"
  model_override TEXT,               -- Optional model override
  delete_after_run BOOLEAN DEFAULT FALSE,
  last_run_at INTEGER,
  created_at INTEGER DEFAULT (unixepoch())
);
```

### Cron Runner

```typescript
// cron-runner.ts
import cron from "node-cron";
import { runAgent } from "./agent.js";
import { deliver } from "./delivery.js";

async function loadAndScheduleJobs() {
  const jobs = db.prepare("SELECT * FROM cron_jobs WHERE cron_expression IS NOT NULL").all();

  for (const job of jobs) {
    cron.schedule(job.cron_expression, async () => {
      console.log(`[CRON] Running job: ${job.id}`);

      const result = await runAgent(job.prompt, {
        model: job.model_override
      });

      if (job.deliver_to) {
        await deliver(job.deliver_to, result);
      }

      db.run("UPDATE cron_jobs SET last_run_at = ? WHERE id = ?", [Date.now(), job.id]);

      if (job.delete_after_run) {
        db.run("DELETE FROM cron_jobs WHERE id = ?", [job.id]);
      }
    });
  }
}

// Heartbeat (every 30 minutes)
setInterval(async () => {
  const heartbeatMd = await fs.readFile("workspace/HEARTBEAT.md", "utf-8").catch(() => null);
  if (!heartbeatMd) return;

  const result = await runAgent(
    `Check HEARTBEAT.md and follow instructions. If nothing needs attention, reply exactly: HEARTBEAT_OK\n\n${heartbeatMd}`
  );

  if (result && result.trim() !== "HEARTBEAT_OK") {
    await deliver(config.heartbeat.deliverTo, result);
  }
}, 30 * 60 * 1000);
```

### Webhook Server

```typescript
// webhook-server.ts
import Fastify from "fastify";
import { runAgent } from "./agent.js";

const app = Fastify();
const WEBHOOK_TOKEN = process.env.WEBHOOK_TOKEN;

// Auth middleware
app.addHook("preHandler", async (req, reply) => {
  const token = req.headers.authorization?.replace("Bearer ", "")
             || req.headers["x-webhook-token"];
  if (token !== WEBHOOK_TOKEN) {
    reply.code(401).send({ error: "Unauthorized" });
  }
});

// Wake endpoint - add event to main session
app.post("/hooks/wake", async (req) => {
  const { text } = req.body as { text: string };
  // Queue for next agent run
  await db.run("INSERT INTO pending_events VALUES (?, ?)", [Date.now(), text]);
  return { ok: true };
});

// Agent endpoint - run isolated agent
app.post("/hooks/agent", async (req) => {
  const { prompt, deliver_to } = req.body as { prompt: string; deliver_to?: string };

  // Run in background
  setImmediate(async () => {
    const result = await runAgent(prompt);
    if (deliver_to) {
      await deliver(deliver_to, result);
    }
  });

  return { ok: true, status: "queued" };
});

app.listen({ port: 3000, host: "127.0.0.1" }); // Localhost only!
```

---

## Security Model

### Defense Layers

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Tool Design (Your MCP Server)                      │
│  • Only expose tools you explicitly need                    │
│  • Validate all inputs                                      │
│  • No shell, no arbitrary file access, no unrestricted net  │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: Rate Limiting                                      │
│  • memory_append: 10/hour                                   │
│  • memory_curate: 3/hour                                    │
│  • delivery (telegram/email): 5/hour                        │
│  • cron_add: 5/day                                          │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: Audit Logging                                      │
│  • Every tool call logged with timestamp, params, result    │
│  • Review logs regularly for anomalies                      │
│  • Alert on unusual patterns                                │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: Network Isolation                                  │
│  • Run on dedicated VPS (nothing else on it)                │
│  • Tailscale for access (no public ports)                   │
│  • Firewall: allow only required API endpoints              │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│  Layer 5: Operational Security                               │
│  • Credentials in env vars only (not files)                 │
│  • Weekly VM snapshots                                      │
│  • If compromised: nuke and restore                         │
└─────────────────────────────────────────────────────────────┘
```

### Prompt Injection Mitigation

```typescript
// In your MCP server, treat LLM outputs as untrusted:

// BAD - trusts LLM output
tool("file_write", async ({ path, content }) => {
  await fs.writeFile(path, content); // LLM controls path!
});

// GOOD - validates and restricts
tool("memory_append", async ({ content }) => {
  // Fixed path, LLM only controls content
  const today = `memory/${new Date().toISOString().slice(0, 10)}.md`;

  // Validate content
  if (content.length > 10000) throw new Error("Content too large");
  if (content.includes("```bash") && content.includes("rm ")) {
    // Log suspicious content but still allow (it's just text)
    console.warn("Suspicious content pattern detected");
  }

  // Append only, can't overwrite
  await fs.appendFile(today, `\n${content}`);
  return { ok: true, file: today };
});
```

---

## Implementation Roadmap

### Phase 1: Core MCP Server (Week 1)

- [ ] Set up TypeScript + MCP SDK project
- [ ] Implement SQLite schema (memory_chunks, cron_jobs, audit_log)
- [ ] Implement `memory_read` tool
- [ ] Implement `memory_append` tool
- [ ] Implement `memory_curate` tool (write to MEMORY.md)
- [ ] Add input validation to all tools
- [ ] Add audit logging

### Phase 2: Vector Search (Week 1-2)

- [ ] Choose embedding approach:
  - Local: `@xenova/transformers` with `all-MiniLM-L6-v2` (~80MB)
  - Remote: OpenAI `text-embedding-3-small` (costs money, better quality)
- [ ] Implement chunking algorithm
- [ ] Implement embedding generation + caching
- [ ] Implement hybrid search (vector + BM25)
- [ ] Implement `memory_search` tool

### Phase 3: Agent Runner (Week 2)

- [ ] Set up Claude Agent SDK integration
- [ ] Create `runAgent()` wrapper function
- [ ] Test with Claude Code directly first (zero code)
- [ ] Add custom system prompt support

### Phase 4: Scheduling (Week 2-3)

- [ ] Implement cron jobs table
- [ ] Implement `cron_add`, `cron_list`, `cron_remove` tools
- [ ] Build cron-runner.ts (node-cron based)
- [ ] Add heartbeat interval
- [ ] Build delivery system (start with one channel: Telegram or email)

### Phase 5: Webhooks (Week 3)

- [ ] Build webhook-server.ts (Fastify)
- [ ] Implement `/hooks/wake` endpoint
- [ ] Implement `/hooks/agent` endpoint
- [ ] Add authentication
- [ ] Test with external service (e.g., GitHub webhook)

### Phase 6: Hardening (Week 3-4)

- [ ] Add rate limiting to all tools
- [ ] Review and tighten input validation
- [ ] Set up Tailscale for access
- [ ] Configure firewall rules
- [ ] Set up VM snapshots
- [ ] Write operational runbook

---

## Code References from OpenClaw

### Files Worth Studying

| Feature | OpenClaw Path | Key Function/Pattern |
|---------|---------------|---------------------|
| Markdown chunking | `src/memory/internal.ts` | `chunkMarkdown()` |
| Hybrid search | `src/memory/manager-search.ts` | Vector + BM25 merge |
| File hashing | `src/memory/internal.ts` | `hashText()` |
| Session keys | `src/config/sessions/session-key.ts` | Key generation patterns |
| Compaction | `src/agents/compaction.ts` | When/how to summarize |
| Memory flush | `src/agents/pi-embedded-subscribe.ts` | Pre-compaction save |
| Webhook design | `docs/automation/webhook.md` | Endpoint patterns |
| Cron design | `docs/automation/cron-jobs.md` | Job management |
| Heartbeat | `docs/gateway/heartbeat.md` | Periodic check-in |

### Key Design Decisions from OpenClaw (Adopt These)

1. **Append-only daily logs** - Can't corrupt past data
2. **Content hashing for deduplication** - Don't re-embed unchanged content
3. **Hybrid search** - BM25 catches exact matches vectors miss
4. **Session key namespacing** - Clear isolation model
5. **Audit logging** - Know what happened when
6. **Heartbeat suppression** - Don't spam "nothing to report"

### Key Design Decisions from OpenClaw (Reject These)

1. **Shell execution tools** - Never expose
2. **Arbitrary file read/write** - Only specific paths
3. **Browser automation** - Attack surface too large
4. **Complex sandboxing** - VM isolation is simpler
5. **20+ channel integrations** - Start with 1, add as needed

---

## Quick Reference

### Estimated Lines of Code

| Component | Lines |
|-----------|-------|
| MCP Server (tools + validation) | ~300-400 |
| SQLite schema + queries | ~100 |
| Agent runner wrapper | ~50-100 |
| Cron runner | ~100 |
| Webhook server | ~100 |
| Delivery (1 channel) | ~50 |
| **Total** | **~700-900** |

### Dependencies (Minimal Set)

```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "@anthropic-ai/claude-code-sdk": "^1.0.0",
    "better-sqlite3": "^11.0.0",
    "node-cron": "^3.0.0",
    "fastify": "^5.0.0",
    "@xenova/transformers": "^3.0.0"
  }
}
```

### Environment Variables

```bash
# Required
ANTHROPIC_API_KEY=sk-ant-...
WEBHOOK_TOKEN=your-secret-token

# Optional
OPENAI_API_KEY=sk-...           # If using OpenAI embeddings
TELEGRAM_BOT_TOKEN=123:ABC...   # If delivering to Telegram
```

---

## Final Notes

This architecture gives you everything valuable from OpenClaw (memory, scheduling, semantic search) without the security risks (shell execution, 485K lines of unknown code, known CVEs).

The key insight: **Claude Agent SDK handles the hard parts** (agentic loop, context management, tool orchestration). Your MCP server just needs to be a thin, secure layer exposing exactly the capabilities you want.

Build small. Add incrementally. Trust nothing from the LLM.
