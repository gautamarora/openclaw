# Casper Claw — Technical Design Document

## 1. Overview

Casper Claw is a personal AI assistant that runs as a WhatsApp bot, built on the OpenClaw architecture using TypeScript throughout. It replaces OpenClaw's CLI-skill pattern with native TypeScript SDK integrations, and simplifies the multi-channel gateway into a single-channel WhatsApp runtime using Baileys.

### 1.1 Design Principles

- **TypeScript-native**: All integrations use official TypeScript/JS SDKs — no shelling out to CLI binaries
- **Single-channel**: WhatsApp via Baileys is the only channel; no channel plugin abstraction needed
- **Claw architecture**: Preserves the core Gateway, Heartbeat, and Memory systems from OpenClaw
- **Pi runtime**: Uses `@mariozechner/pi-agent-core`, `pi-ai`, and `pi-coding-agent` for the agentic loop
- **Minimal surface**: No skills system, no sandbox, no multi-agent routing — one agent, one channel, focused

### 1.2 What We Keep from OpenClaw

| OpenClaw Concept | Casper Claw Equivalent |
|---|---|
| Gateway (message routing + lifecycle) | Simplified gateway (Baileys connection + message dispatch) |
| Heartbeat (periodic background agent turns) | Heartbeat runner (interval timer + HEARTBEAT.md) |
| Memory (vector search + markdown files) | Memory system (SQLite + embeddings + markdown) |
| Pi runtime (agent loop + tools + streaming) | Same — `pi-coding-agent` + `pi-ai` |
| Session persistence (JSONL transcripts) | Same — `SessionManager` from `pi-coding-agent` |
| System prompt (identity + context injection) | Simplified system prompt builder |
| Tools (structured LLM tool calls) | TypeScript SDK tools (not CLI wrappers) |

### 1.3 What We Drop

- Multi-channel plugin architecture (only WhatsApp)
- Skills system (SKILL.md + CLI binaries)
- Multi-agent routing and bindings
- Sandbox/container isolation
- WebSocket control plane + Control UI
- OpenAI-compatible HTTP API endpoints
- Device pairing, mDNS discovery, Tailscale
- Cron system (heartbeat covers periodic needs for v1)
- Canvas, TTS, Nodes, Browser tools

---

## 2. Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Casper Claw                        │
│                                                      │
│  ┌──────────┐    ┌──────────┐    ┌───────────────┐  │
│  │ WhatsApp │    │ Gateway  │    │  Pi Runtime   │  │
│  │ (Baileys)│───>│ (Router) │───>│  (Agent Loop) │  │
│  │          │<───│          │<───│               │  │
│  └──────────┘    └──────────┘    └───────┬───────┘  │
│                       │                  │           │
│                  ┌────┴────┐       ┌─────┴─────┐    │
│                  │Heartbeat│       │   Tools    │    │
│                  │ Runner  │       │ (TS SDKs)  │    │
│                  └─────────┘       └─────┬─────┘    │
│                                          │           │
│                                    ┌─────┴─────┐    │
│                                    │  Memory    │    │
│                                    │  (SQLite)  │    │
│                                    └───────────┘    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### 2.2 Component Dependency Graph

```
casper-claw (main process)
├── @whiskeysockets/baileys          # WhatsApp connection
├── @mariozechner/pi-coding-agent    # Agent session, SessionManager
├── @mariozechner/pi-ai              # LLM streaming (streamSimple)
├── @mariozechner/pi-agent-core      # Agent types, events
├── better-sqlite3                   # Memory index storage
├── openai                           # Embeddings + image gen SDK
│
├── SDK integrations (per config):
│   ├── octokit                      # GitHub
│   ├── @notionhq/client             # Notion
│   ├── @slack/web-api               # Slack
│   ├── @google/generative-ai        # Gemini
│   └── ... (user-selected SDKs)
│
└── Node.js built-ins:
    ├── fs/path                      # Session files, memory files
    ├── crypto                       # Session IDs, hashing
    └── timers                       # Heartbeat scheduling
```

### 2.3 Directory Structure

```
casper-claw/
├── src/
│   ├── index.ts                     # Entry point — starts gateway
│   ├── config/
│   │   ├── types.ts                 # CasperConfig type definition
│   │   ├── loader.ts                # Load + validate casper.json
│   │   └── defaults.ts              # Default config values
│   ├── gateway/
│   │   ├── gateway.ts               # Main gateway lifecycle
│   │   ├── whatsapp.ts              # Baileys connection manager
│   │   ├── message-handler.ts       # Inbound message dispatch
│   │   └── session-key.ts           # Session key generation
│   ├── agent/
│   │   ├── runner.ts                # Wraps runEmbeddedPiAgent()
│   │   ├── system-prompt.ts         # System prompt builder
│   │   ├── tools.ts                 # Tool assembly + policy
│   │   └── identity.ts              # Agent name, prefix, emoji
│   ├── heartbeat/
│   │   ├── runner.ts                # Interval timer + execution
│   │   ├── visibility.ts            # Suppress/deliver logic
│   │   └── active-hours.ts          # Quiet hours check
│   ├── memory/
│   │   ├── manager.ts               # SQLite index manager
│   │   ├── search.ts                # Hybrid vector + BM25 search
│   │   ├── embeddings.ts            # Embedding provider abstraction
│   │   ├── chunker.ts               # Markdown → chunks
│   │   ├── sync.ts                  # File watcher + reindex
│   │   └── tools.ts                 # memory_search, memory_get tools
│   ├── tools/
│   │   ├── exec.ts                  # Shell execution tool
│   │   ├── fs.ts                    # Read, write, edit tools
│   │   ├── web.ts                   # web_search, web_fetch tools
│   │   ├── github.ts                # GitHub tool (octokit)
│   │   ├── notion.ts                # Notion tool
│   │   └── index.ts                 # Tool registry + assembly
│   └── sessions/
│       ├── manager.ts               # Session file read/write
│       └── history.ts               # History limiting + compaction
├── casper.json                      # Configuration file
├── workspace/
│   ├── AGENTS.md                    # Agent operating instructions
│   ├── SOUL.md                      # Personality + tone
│   ├── HEARTBEAT.md                 # Periodic check instructions
│   ├── MEMORY.md                    # Curated long-term memory
│   └── memory/
│       └── YYYY-MM-DD.md            # Daily memory logs
├── package.json
├── tsconfig.json
└── README.md
```

---

## 3. Configuration

### 3.1 Config File (`casper.json`)

A single JSON5 config file at the project root (or `~/.casper/casper.json`).

```typescript
type CasperConfig = {
  // Agent identity
  agent: {
    name: string;                        // "Casper"
    workspace: string;                   // "./workspace"
    model: string;                       // "anthropic/claude-sonnet-4-5-20250929"
    fallbackModel?: string;              // "anthropic/claude-haiku-4-5-20251001"
    thinkingLevel?: "off" | "low" | "high";
  };

  // WhatsApp connection
  whatsapp: {
    authDir: string;                     // "./auth" — Baileys auth state
    ownerJid?: string;                   // Owner's JID for privileged commands
    allowedJids?: string[];              // JID allowlist (empty = allow all)
    groupPolicy: "ignore" | "mention" | "all";  // How to handle group messages
    mentionPatterns?: string[];          // ["@casper", "@bot"]
    typingDelay?: {                      // Human-like typing simulation
      enabled: boolean;
      minMs: number;                     // 500
      maxMs: number;                     // 2000
      perCharMs: number;                 // 30
    };
  };

  // Session management
  session: {
    dir: string;                         // "./sessions"
    scope: "per-sender" | "global";      // One session per sender or shared
    reset?: {
      mode: "daily" | "idle";
      atHour?: number;                   // 0-23 for daily reset
      idleMinutes?: number;              // Minutes before idle reset
    };
    maxHistoryTurns?: number;            // Limit conversation turns (default: 50)
  };

  // Heartbeat
  heartbeat: {
    enabled: boolean;                    // true
    every: string;                       // "30m"
    target: "owner" | "none";            // Who receives heartbeat messages
    activeHours?: {
      start: string;                     // "08:00"
      end: string;                       // "22:00"
      timezone: string;                  // "America/New_York"
    };
    prompt?: string;                     // Custom heartbeat prompt
    ackMaxChars?: number;                // 300 — suppress short acks
  };

  // Memory
  memory: {
    enabled: boolean;                    // true
    embedding: {
      provider: "openai" | "local";      // Embedding provider
      model?: string;                    // "text-embedding-3-small"
      apiKey?: string;                   // Or use env OPENAI_API_KEY
    };
    store: {
      path: string;                      // "./memory.sqlite"
    };
    chunking: {
      tokens: number;                    // 400
      overlap: number;                   // 80
    };
    search: {
      maxResults: number;                // 6
      minScore: number;                  // 0.35
      hybrid: {
        enabled: boolean;                // true
        vectorWeight: number;            // 0.7
        textWeight: number;              // 0.3
      };
    };
    sync: {
      onSessionStart: boolean;           // true
      onSearch: boolean;                  // true
      watch: boolean;                    // true
      watchDebounceMs: number;           // 1500
    };
  };

  // Tools — which SDK integrations to enable
  tools: {
    profile: "minimal" | "coding" | "full";
    exec?: {
      enabled: boolean;                  // true for coding profile
      timeoutSec?: number;               // 120
    };
    web?: {
      enabled: boolean;
      searchProvider?: "brave" | "perplexity";
      searchApiKey?: string;
    };
    integrations?: {
      github?: {
        enabled: boolean;
        token: string;                   // Or env GITHUB_TOKEN
      };
      notion?: {
        enabled: boolean;
        apiKey: string;                  // Or env NOTION_API_KEY
      };
      // Extensible — add more SDK integrations here
    };
  };

  // Auth
  auth: {
    anthropicApiKey?: string;            // Or env ANTHROPIC_API_KEY
    openaiApiKey?: string;               // For embeddings; or env OPENAI_API_KEY
  };

  // Logging
  logging?: {
    level: "debug" | "info" | "warn" | "error";
    file?: string;                       // Log file path
  };
};
```

### 3.2 Example Configuration

```json5
{
  "agent": {
    "name": "Casper",
    "workspace": "./workspace",
    "model": "anthropic/claude-sonnet-4-5-20250929",
    "thinkingLevel": "low"
  },
  "whatsapp": {
    "authDir": "./auth",
    "ownerJid": "1234567890@s.whatsapp.net",
    "groupPolicy": "mention",
    "mentionPatterns": ["@casper"]
  },
  "session": {
    "dir": "./sessions",
    "scope": "per-sender",
    "reset": { "mode": "daily", "atHour": 4 },
    "maxHistoryTurns": 50
  },
  "heartbeat": {
    "enabled": true,
    "every": "30m",
    "target": "owner",
    "activeHours": {
      "start": "08:00",
      "end": "22:00",
      "timezone": "America/New_York"
    }
  },
  "memory": {
    "enabled": true,
    "embedding": {
      "provider": "openai",
      "model": "text-embedding-3-small"
    },
    "store": { "path": "./memory.sqlite" },
    "chunking": { "tokens": 400, "overlap": 80 },
    "search": {
      "maxResults": 6,
      "minScore": 0.35,
      "hybrid": { "enabled": true, "vectorWeight": 0.7, "textWeight": 0.3 }
    },
    "sync": {
      "onSessionStart": true,
      "onSearch": true,
      "watch": true,
      "watchDebounceMs": 1500
    }
  },
  "tools": {
    "profile": "coding",
    "exec": { "enabled": true, "timeoutSec": 120 },
    "web": { "enabled": true, "searchProvider": "brave" },
    "integrations": {
      "github": { "enabled": true }
    }
  },
  "logging": {
    "level": "info",
    "file": "./casper.log"
  }
}
```

---

## 4. Gateway

The Gateway is the main process. It owns the WhatsApp connection, dispatches inbound messages to the agent, and delivers responses back.

### 4.1 Lifecycle

```
startGateway(config)
│
├─ 1. Load config (casper.json)
├─ 2. Initialize Baileys connection
│     ├─ Load auth state from config.whatsapp.authDir
│     ├─ Connect to WhatsApp (QR code on first run)
│     └─ Register message listener
├─ 3. Initialize memory system
│     ├─ Open SQLite database
│     ├─ Run initial sync (index workspace/MEMORY.md + memory/*.md)
│     └─ Start file watcher
├─ 4. Initialize agent
│     ├─ Resolve model + auth
│     ├─ Build system prompt
│     └─ Assemble tools
├─ 5. Start heartbeat runner
│     ├─ Calculate next due time
│     └─ Schedule timer
├─ 6. RUNNING — process messages
│
└─ On SIGINT/SIGTERM:
    ├─ Stop heartbeat timer
    ├─ Close Baileys connection
    ├─ Close SQLite database
    └─ Exit
```

### 4.2 WhatsApp Connection (`src/gateway/whatsapp.ts`)

Wraps `@whiskeysockets/baileys` with reconnection and auth state management.

```typescript
import makeWASocket, {
  useMultiFileAuthState,
  DisconnectReason,
  WAMessage,
} from "@whiskeysockets/baileys";

type WhatsAppConnection = {
  sock: WASocket;
  sendMessage: (jid: string, text: string) => Promise<void>;
  sendReaction: (jid: string, messageId: string, emoji: string) => Promise<void>;
  isConnected: () => boolean;
  close: () => Promise<void>;
};

async function createWhatsAppConnection(config: CasperConfig): Promise<WhatsAppConnection> {
  const { state, saveCreds } = await useMultiFileAuthState(config.whatsapp.authDir);

  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: true,       // QR code for first-time pairing
  });

  // Persist auth state on credential updates
  sock.ev.on("creds.update", saveCreds);

  // Handle connection state changes
  sock.ev.on("connection.update", ({ connection, lastDisconnect }) => {
    if (connection === "close") {
      const reason = (lastDisconnect?.error as any)?.output?.statusCode;
      if (reason !== DisconnectReason.loggedOut) {
        // Reconnect with exponential backoff
        reconnect();
      }
    }
  });

  return {
    sock,
    sendMessage: async (jid, text) => {
      await sock.sendMessage(jid, { text });
    },
    sendReaction: async (jid, messageId, emoji) => {
      await sock.sendMessage(jid, {
        react: { text: emoji, key: { id: messageId, remoteJid: jid } },
      });
    },
    isConnected: () => sock.user !== undefined,
    close: async () => { sock.end(undefined); },
  };
}
```

### 4.3 Message Handler (`src/gateway/message-handler.ts`)

Receives inbound WhatsApp messages, filters them, and dispatches to the agent.

```typescript
type InboundMessage = {
  jid: string;                     // Sender JID
  senderName: string;              // Push name
  text: string;                    // Message body
  messageId: string;               // WhatsApp message ID
  isGroup: boolean;                // Group or DM
  quotedText?: string;             // Quoted/replied message
  mediaType?: "image" | "audio" | "document";
  mediaBuffer?: Buffer;            // Downloaded media bytes
  timestamp: number;
};

async function handleInboundMessage(
  msg: InboundMessage,
  ctx: {
    config: CasperConfig;
    wa: WhatsAppConnection;
    agent: AgentRunner;
    sessionManager: SessionManager;
  },
): Promise<void> {
  // 1. Filter: ignore own messages, status broadcasts
  if (msg.jid === ctx.wa.sock.user?.id) return;
  if (msg.jid === "status@broadcast") return;

  // 2. Filter: check allowlist
  if (ctx.config.whatsapp.allowedJids?.length) {
    if (!ctx.config.whatsapp.allowedJids.includes(msg.jid)) return;
  }

  // 3. Filter: group policy
  if (msg.isGroup) {
    const policy = ctx.config.whatsapp.groupPolicy;
    if (policy === "ignore") return;
    if (policy === "mention") {
      const patterns = ctx.config.whatsapp.mentionPatterns ?? [];
      const mentioned = patterns.some((p) => msg.text.toLowerCase().includes(p.toLowerCase()));
      if (!mentioned) return;
    }
  }

  // 4. Resolve session key
  const sessionKey = buildSessionKey(msg.jid, ctx.config);

  // 5. Send ack reaction (eyes emoji)
  await ctx.wa.sendReaction(msg.jid, msg.messageId, "👀");

  // 6. Dispatch to agent
  const result = await ctx.agent.run({
    prompt: msg.text,
    sessionKey,
    senderName: msg.senderName,
    senderJid: msg.jid,
    media: msg.mediaBuffer ? { type: msg.mediaType!, data: msg.mediaBuffer } : undefined,
  });

  // 7. Apply typing delay
  if (ctx.config.whatsapp.typingDelay?.enabled) {
    await simulateTypingDelay(result.text, ctx.config.whatsapp.typingDelay);
  }

  // 8. Send response
  await ctx.wa.sendMessage(msg.jid, result.text);

  // 9. Clear ack reaction
  await ctx.wa.sendReaction(msg.jid, msg.messageId, "");
}
```

### 4.4 Session Key Generation (`src/gateway/session-key.ts`)

```typescript
function buildSessionKey(jid: string, config: CasperConfig): string {
  if (config.session.scope === "global") {
    return "casper:main";
  }
  // per-sender: one session per WhatsApp JID
  const normalized = jid.split("@")[0]; // strip @s.whatsapp.net
  return `casper:wa:${normalized}`;
}
```

### 4.5 Concurrency Control

Only one agent run per session key at a time. If a message arrives while the agent is already running for that session, queue it.

```typescript
type MessageQueue = Map<string, Array<InboundMessage>>;
type ActiveRuns = Set<string>;

// In gateway.ts:
const queue: MessageQueue = new Map();
const active: ActiveRuns = new Set();

async function dispatch(msg: InboundMessage, sessionKey: string): Promise<void> {
  if (active.has(sessionKey)) {
    // Queue the message — process after current run completes
    const pending = queue.get(sessionKey) ?? [];
    pending.push(msg);
    queue.set(sessionKey, pending);
    return;
  }

  active.add(sessionKey);
  try {
    await handleInboundMessage(msg, ctx);

    // Process queued messages for this session
    while (queue.get(sessionKey)?.length) {
      const next = queue.get(sessionKey)!.shift()!;
      await handleInboundMessage(next, ctx);
    }
  } finally {
    active.delete(sessionKey);
    queue.delete(sessionKey);
  }
}
```

---

## 5. Agent Runtime

### 5.1 Agent Runner (`src/agent/runner.ts`)

Wraps the pi-coding-agent runtime. This is the core agent loop.

```typescript
import { createAgentSession } from "@mariozechner/pi-coding-agent";
import { streamSimple } from "@mariozechner/pi-ai";

type AgentRunParams = {
  prompt: string;
  sessionKey: string;
  senderName: string;
  senderJid: string;
  media?: { type: string; data: Buffer };
};

type AgentRunResult = {
  text: string;
  usage: { input: number; output: number };
  durationMs: number;
  toolCalls: string[];          // Names of tools invoked
};

class AgentRunner {
  private config: CasperConfig;
  private tools: AnyAgentTool[];
  private systemPrompt: string;

  constructor(config: CasperConfig, tools: AnyAgentTool[]) {
    this.config = config;
    this.tools = tools;
    this.systemPrompt = buildSystemPrompt(config);
  }

  async run(params: AgentRunParams): Promise<AgentRunResult> {
    const startedAt = Date.now();

    // 1. Create or load session
    const { session } = await createAgentSession({
      cwd: this.config.agent.workspace,
      model: this.config.agent.model,
      customTools: this.tools,
      sessionManager: getSessionManager(params.sessionKey, this.config),
    });

    // 2. Set system prompt
    session.agent.systemPrompt = this.systemPrompt;
    session.agent.streamFn = streamSimple;

    // 3. Subscribe to events (collect results)
    const collected = { texts: [] as string[], toolCalls: [] as string[] };
    session.subscribe((event) => {
      if (event.type === "message_update" && event.data?.type === "text_delta") {
        collected.texts.push(event.data.delta);
      }
      if (event.type === "tool_execution_end") {
        collected.toolCalls.push(event.data.toolName);
      }
    });

    // 4. Run agent loop
    await session.prompt(params.prompt);

    // 5. Collect final text
    const text = collected.texts.join("");
    const usage = session.lastUsage ?? { input: 0, output: 0 };

    return {
      text,
      usage,
      durationMs: Date.now() - startedAt,
      toolCalls: collected.toolCalls,
    };
  }
}
```

### 5.2 System Prompt (`src/agent/system-prompt.ts`)

Simplified from OpenClaw's full system prompt builder. No skills section, no sandbox section, no multi-agent section.

```typescript
function buildSystemPrompt(config: CasperConfig): string {
  const sections: string[] = [];

  // 1. Identity
  sections.push(
    `You are ${config.agent.name}, a personal AI assistant running on WhatsApp.`,
    `You communicate via WhatsApp messages. Keep responses concise and mobile-friendly.`,
  );

  // 2. Tools
  sections.push(
    `## Tools`,
    `You have tools available. Use them when needed to help the user.`,
    `- File tools (read, write, edit): Access files in the workspace`,
    `- Exec tool: Run shell commands`,
    `- Memory tools (memory_search, memory_get): Search and retrieve long-term memories`,
    // ... dynamically list enabled integrations
  );

  // 3. Memory
  if (config.memory.enabled) {
    sections.push(
      `## Memory`,
      `You have long-term memory. Use memory_search to recall past information.`,
      `Write important facts, preferences, and decisions to memory/ files.`,
      `- MEMORY.md: Curated long-term facts`,
      `- memory/YYYY-MM-DD.md: Daily logs (append-only)`,
    );
  }

  // 4. Workspace files (inject AGENTS.md, SOUL.md, etc.)
  const workspaceFiles = ["AGENTS.md", "SOUL.md", "USER.md"];
  for (const file of workspaceFiles) {
    const content = readFileIfExists(path.join(config.agent.workspace, file));
    if (content) {
      sections.push(`## ${file}`, content);
    }
  }

  // 5. Runtime info
  sections.push(
    `## Runtime`,
    `Agent: ${config.agent.name}`,
    `Model: ${config.agent.model}`,
    `Date: ${new Date().toISOString()}`,
    `Platform: ${process.platform} (${process.arch})`,
  );

  return sections.join("\n\n");
}
```

### 5.3 Context Management

Follow OpenClaw's approach: context window guard + auto-compaction + pre-compaction memory flush.

```
On each agent run:
  1. Check context window guard
     - Warn below 8000 tokens remaining
     - Hard-fail below 1000 tokens
  2. If context overflow during LLM call:
     a. Trigger pre-compaction memory flush
        - Silent agentic turn: "Store durable memories now"
        - Agent writes to memory/YYYY-MM-DD.md
     b. Compact session history (summarize old turns)
     c. Retry the prompt
  3. If compaction still overflows:
     - Truncate oversized tool results in history
     - Retry once more
  4. If still overflows:
     - Return error to user: "Session too long, resetting."
     - Reset session
```

---

## 6. Tools

### 6.1 Tool Architecture

Tools are TypeScript objects conforming to the `AnyAgentTool` interface from `pi-agent-core`. Unlike OpenClaw's skill system (which teaches the LLM to use CLIs via the exec tool), Casper Claw tools are structured, typed, and call SDKs directly.

```typescript
type AnyAgentTool = {
  name: string;
  label?: string;
  description: string;
  parameters: object;          // JSON Schema
  execute: (
    toolCallId: string,
    params: unknown,
    signal?: AbortSignal,
  ) => Promise<{ content: Array<{ type: "text"; text: string }> }>;
};
```

### 6.2 Tool Profiles

Three profiles control which tools are available:

```
minimal:    memory_search, memory_get
coding:     minimal + read, write, edit, exec, web_search, web_fetch
full:       coding + all enabled integrations
```

### 6.3 Core Tools (Always Available)

#### File Tools (`src/tools/fs.ts`)

```typescript
// read: Read file contents
{ name: "read", params: { path: string, offset?: number, limit?: number } }

// write: Write file contents
{ name: "write", params: { path: string, content: string } }

// edit: Find-and-replace in file
{ name: "edit", params: { path: string, old_string: string, new_string: string } }
```

These wrap Node.js `fs` operations. Paths are resolved relative to the workspace and validated to prevent directory traversal.

#### Exec Tool (`src/tools/exec.ts`)

```typescript
// exec: Run a shell command
{ name: "exec", params: { command: string, timeout?: number } }
```

Spawns a child process with configurable timeout. Captures stdout + stderr. Working directory is the workspace.

#### Memory Tools (`src/memory/tools.ts`)

```typescript
// memory_search: Semantic + keyword search over memory files
{ name: "memory_search", params: { query: string, maxResults?: number } }

// memory_get: Read a specific memory file
{ name: "memory_get", params: { path: string, from?: number, lines?: number } }
```

Delegates to the memory manager (Section 8).

#### Web Tools (`src/tools/web.ts`)

```typescript
// web_search: Search the web
{ name: "web_search", params: { query: string, maxResults?: number } }

// web_fetch: Fetch and parse a URL
{ name: "web_fetch", params: { url: string, prompt?: string } }
```

`web_search` calls Brave Search API or Perplexity. `web_fetch` fetches a URL, converts HTML to markdown, and optionally summarizes.

### 6.4 SDK Integration Tools

These replace OpenClaw's CLI-based skills with direct SDK calls. Each integration is optional and enabled via config.

#### GitHub Tool (`src/tools/github.ts`)

```typescript
import { Octokit } from "octokit";

// Single tool with action-based dispatch (like OpenClaw's browser tool pattern)
{
  name: "github",
  description: "Interact with GitHub repositories, issues, PRs, and actions.",
  params: {
    action: "list_prs" | "get_pr" | "create_issue" | "list_issues" |
            "get_issue" | "list_runs" | "get_run" | "search_code" |
            "list_repos" | "get_file" | "create_pr",
    owner?: string,
    repo?: string,
    // ... action-specific params
  },
  execute: async (id, params) => {
    const octokit = new Octokit({ auth: config.tools.integrations.github.token });
    switch (params.action) {
      case "list_prs":
        const prs = await octokit.rest.pulls.list({
          owner: params.owner,
          repo: params.repo,
          state: params.state ?? "open",
        });
        return jsonResult(prs.data);
      // ... other actions
    }
  },
}
```

#### Notion Tool (`src/tools/notion.ts`)

```typescript
import { Client } from "@notionhq/client";

{
  name: "notion",
  description: "Create, search, and manage Notion pages and databases.",
  params: {
    action: "search" | "get_page" | "create_page" | "update_page" |
            "query_database" | "append_blocks",
    // ... action-specific params
  },
  execute: async (id, params) => {
    const notion = new Client({ auth: config.tools.integrations.notion.apiKey });
    switch (params.action) {
      case "search":
        const results = await notion.search({ query: params.query });
        return jsonResult(results);
      // ... other actions
    }
  },
}
```

### 6.5 Adding New Integrations

New SDK tools follow a consistent pattern:

```typescript
// src/tools/my-integration.ts
import { SomeSDK } from "some-sdk";

export function createMyIntegrationTool(config: IntegrationConfig): AnyAgentTool {
  const client = new SomeSDK({ auth: config.apiKey });

  return {
    name: "my_integration",
    description: "Do things with MyService.",
    parameters: {
      type: "object",
      properties: {
        action: { type: "string", enum: ["list", "get", "create"] },
        // ... action-specific params
      },
      required: ["action"],
    },
    execute: async (toolCallId, params) => {
      switch (params.action) {
        case "list": return jsonResult(await client.list());
        case "get": return jsonResult(await client.get(params.id));
        case "create": return jsonResult(await client.create(params.data));
        default: return errorResult(`Unknown action: ${params.action}`);
      }
    },
  };
}
```

Register in `src/tools/index.ts`:

```typescript
export function assembleTools(config: CasperConfig): AnyAgentTool[] {
  const tools: AnyAgentTool[] = [
    // Core tools (always available based on profile)
    ...createFsTools(config),
    createExecTool(config),
    ...createMemoryTools(config),
    ...createWebTools(config),
  ];

  // SDK integrations (conditionally enabled)
  if (config.tools.integrations?.github?.enabled) {
    tools.push(createGitHubTool(config.tools.integrations.github));
  }
  if (config.tools.integrations?.notion?.enabled) {
    tools.push(createNotionTool(config.tools.integrations.notion));
  }
  // ... more integrations

  // Apply profile filter
  return applyToolProfile(tools, config.tools.profile);
}
```

---

## 7. Heartbeat

### 7.1 What It Does

The heartbeat is a periodic agent turn that runs in the main session. It reads `HEARTBEAT.md` from the workspace and follows its instructions. If nothing needs attention, the agent responds with `HEARTBEAT_OK` and the message is suppressed.

### 7.2 Heartbeat Runner (`src/heartbeat/runner.ts`)

```typescript
class HeartbeatRunner {
  private config: CasperConfig;
  private agent: AgentRunner;
  private wa: WhatsAppConnection;
  private timer: NodeJS.Timeout | null = null;
  private lastHeartbeatText: string | null = null;
  private lastHeartbeatSentAt: number = 0;

  start(): void {
    this.scheduleNext();
  }

  stop(): void {
    if (this.timer) clearTimeout(this.timer);
  }

  private scheduleNext(): void {
    const intervalMs = parseDuration(this.config.heartbeat.every);
    this.timer = setTimeout(() => this.tick(), intervalMs);
  }

  private async tick(): Promise<void> {
    try {
      const result = await this.runHeartbeatOnce();
      if (result.status !== "skipped") {
        // Log result
      }
    } finally {
      this.scheduleNext();
    }
  }

  private async runHeartbeatOnce(): Promise<HeartbeatResult> {
    // 1. Check enabled
    if (!this.config.heartbeat.enabled) {
      return { status: "skipped", reason: "disabled" };
    }

    // 2. Check active hours
    if (this.config.heartbeat.activeHours) {
      if (!isWithinActiveHours(this.config.heartbeat.activeHours)) {
        return { status: "skipped", reason: "quiet-hours" };
      }
    }

    // 3. Read HEARTBEAT.md
    const heartbeatPath = path.join(this.config.agent.workspace, "HEARTBEAT.md");
    const heartbeatContent = readFileIfExists(heartbeatPath);
    if (!heartbeatContent?.trim()) {
      return { status: "skipped", reason: "empty-heartbeat-file" };
    }

    // 4. Build prompt
    const prompt = this.config.heartbeat.prompt
      ?? "Read HEARTBEAT.md and follow it. If nothing needs attention, reply HEARTBEAT_OK.";

    // 5. Run agent in the owner's session
    const sessionKey = this.config.heartbeat.target === "owner"
      ? buildSessionKey(this.config.whatsapp.ownerJid!, this.config)
      : "casper:main";

    const result = await this.agent.run({
      prompt,
      sessionKey,
      senderName: "system",
      senderJid: "heartbeat",
    });

    // 6. Check for HEARTBEAT_OK token
    const stripped = stripHeartbeatToken(result.text);
    if (stripped.isAck) {
      // Check if remaining text is short enough to suppress
      if (stripped.remainder.length <= (this.config.heartbeat.ackMaxChars ?? 300)) {
        return { status: "ok-empty" };
      }
      // Deliver the remainder
      result.text = stripped.remainder;
    }

    // 7. Duplicate suppression (don't nag with same message)
    if (result.text === this.lastHeartbeatText &&
        Date.now() - this.lastHeartbeatSentAt < 24 * 60 * 60 * 1000) {
      return { status: "skipped", reason: "duplicate" };
    }

    // 8. Deliver to owner
    if (this.config.heartbeat.target === "owner" && this.config.whatsapp.ownerJid) {
      await this.wa.sendMessage(this.config.whatsapp.ownerJid, result.text);
      this.lastHeartbeatText = result.text;
      this.lastHeartbeatSentAt = Date.now();
    }

    return { status: "ok-sent" };
  }
}
```

### 7.3 HEARTBEAT_OK Token

When the agent determines nothing needs attention, it responds with `HEARTBEAT_OK`. The runner strips this token and suppresses the message.

```typescript
function stripHeartbeatToken(text: string): { isAck: boolean; remainder: string } {
  const token = "HEARTBEAT_OK";
  const trimmed = text.trim();

  // Token at start
  if (trimmed.startsWith(token)) {
    return { isAck: true, remainder: trimmed.slice(token.length).trim() };
  }
  // Token at end
  if (trimmed.endsWith(token)) {
    return { isAck: true, remainder: trimmed.slice(0, -token.length).trim() };
  }
  // Handle markdown wrapping: **HEARTBEAT_OK**
  const mdToken = `**${token}**`;
  if (trimmed.startsWith(mdToken) || trimmed.endsWith(mdToken)) {
    const cleaned = trimmed.replace(mdToken, "").trim();
    return { isAck: true, remainder: cleaned };
  }

  return { isAck: false, remainder: trimmed };
}
```

### 7.4 Example HEARTBEAT.md

```markdown
# Heartbeat Checklist

When this heartbeat runs, check the following:

1. Check if there are any new GitHub notifications:
   - Use the github tool to check recent notifications
   - Only report if there's something urgent (PR reviews requested, CI failures)

2. If it's morning (before 10am), give a brief daily summary:
   - Weather for today
   - Any calendar items (if integrated)

3. If nothing needs attention, respond with HEARTBEAT_OK
```

---

## 8. Memory

### 8.1 Architecture

The memory system provides persistent, searchable knowledge that survives session resets and context compaction.

```
Workspace files (source of truth)
├── MEMORY.md               # Curated long-term facts
└── memory/
    ├── 2026-02-15.md        # Daily log
    ├── 2026-02-16.md
    └── 2026-02-17.md
         │
         ▼
    ┌─────────────┐
    │  Chunker     │  Split markdown into ~400-token chunks
    └──────┬──────┘
           ▼
    ┌─────────────┐
    │ Embeddings   │  OpenAI text-embedding-3-small (or local)
    └──────┬──────┘
           ▼
    ┌─────────────┐
    │   SQLite     │  Store chunks + embeddings + FTS5 index
    │   Database   │
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │ Hybrid       │  Vector cosine similarity (70%) +
    │ Search       │  BM25 keyword match (30%)
    └─────────────┘
```

### 8.2 Database Schema

```sql
-- Metadata table
CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT
);

-- Tracked files
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  hash TEXT NOT NULL,
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);

-- Memory chunks with embeddings
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,
  text TEXT NOT NULL,
  embedding TEXT NOT NULL,       -- JSON array of floats
  updated_at INTEGER NOT NULL,
  FOREIGN KEY (path) REFERENCES files(path)
);

-- BM25 full-text search
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  content=chunks,
  content_rowid=rowid
);

-- Embedding cache (avoid re-embedding unchanged text)
CREATE TABLE embedding_cache (
  hash TEXT PRIMARY KEY,
  model TEXT NOT NULL,
  embedding TEXT NOT NULL,
  dims INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);
```

### 8.3 Chunking (`src/memory/chunker.ts`)

Split markdown files into overlapping chunks of ~400 tokens:

```typescript
type MemoryChunk = {
  path: string;
  startLine: number;       // 1-indexed
  endLine: number;
  text: string;
  hash: string;            // SHA-256 of text
};

function chunkMarkdown(
  path: string,
  content: string,
  opts: { tokens: number; overlap: number },
): MemoryChunk[] {
  const lines = content.split("\n");
  const chunks: MemoryChunk[] = [];
  let start = 0;

  while (start < lines.length) {
    // Accumulate lines until we hit target token count
    let end = start;
    let tokenCount = 0;
    while (end < lines.length && tokenCount < opts.tokens) {
      tokenCount += estimateTokens(lines[end]);
      end++;
    }

    chunks.push({
      path,
      startLine: start + 1,
      endLine: end,
      text: lines.slice(start, end).join("\n"),
      hash: sha256(lines.slice(start, end).join("\n")),
    });

    // Advance with overlap
    const overlapLines = findOverlapStart(lines, end, opts.overlap);
    start = Math.max(start + 1, end - overlapLines);
  }

  return chunks;
}
```

### 8.4 Embedding Provider (`src/memory/embeddings.ts`)

```typescript
interface EmbeddingProvider {
  embed(texts: string[]): Promise<number[][]>;
  dimensions: number;
  model: string;
}

function createEmbeddingProvider(config: CasperConfig["memory"]["embedding"]): EmbeddingProvider {
  switch (config.provider) {
    case "openai":
      return createOpenAIEmbeddings(config);
    case "local":
      return createLocalEmbeddings(config);
    default:
      throw new Error(`Unknown embedding provider: ${config.provider}`);
  }
}

// OpenAI implementation
function createOpenAIEmbeddings(config): EmbeddingProvider {
  const client = new OpenAI({ apiKey: config.apiKey ?? process.env.OPENAI_API_KEY });
  const model = config.model ?? "text-embedding-3-small";

  return {
    model,
    dimensions: 1536,
    embed: async (texts) => {
      const response = await client.embeddings.create({
        model,
        input: texts,
      });
      return response.data.map((d) => d.embedding);
    },
  };
}
```

### 8.5 Hybrid Search (`src/memory/search.ts`)

```typescript
type SearchResult = {
  path: string;
  startLine: number;
  endLine: number;
  score: number;           // 0-1 blended score
  snippet: string;         // Truncated to ~700 chars
};

async function hybridSearch(
  db: Database,
  embedder: EmbeddingProvider,
  query: string,
  opts: CasperConfig["memory"]["search"],
): Promise<SearchResult[]> {
  const maxResults = opts.maxResults;
  const candidateCount = maxResults * (opts.hybrid.candidateMultiplier ?? 4);

  // 1. Vector search
  const queryEmbedding = (await embedder.embed([query]))[0];
  const vectorResults = vectorSearch(db, queryEmbedding, candidateCount);

  // 2. BM25 keyword search
  const textResults = bm25Search(db, query, candidateCount);

  // 3. Merge and score
  const merged = new Map<string, { vectorScore: number; textScore: number }>();

  for (const r of vectorResults) {
    merged.set(r.id, { vectorScore: r.score, textScore: 0 });
  }
  for (const r of textResults) {
    const existing = merged.get(r.id) ?? { vectorScore: 0, textScore: 0 };
    existing.textScore = r.score;
    merged.set(r.id, existing);
  }

  // 4. Weighted blend
  const scored = Array.from(merged.entries()).map(([id, scores]) => ({
    id,
    score: opts.hybrid.vectorWeight * scores.vectorScore +
           opts.hybrid.textWeight * scores.textScore,
  }));

  // 5. Filter and sort
  return scored
    .filter((r) => r.score >= opts.minScore)
    .sort((a, b) => b.score - a.score)
    .slice(0, maxResults)
    .map((r) => lookupChunk(db, r.id, r.score));
}

function vectorSearch(db: Database, embedding: number[], limit: number) {
  // Cosine similarity against all stored embeddings
  const chunks = db.prepare("SELECT id, embedding FROM chunks").all();
  return chunks
    .map((chunk) => ({
      id: chunk.id,
      score: cosineSimilarity(embedding, JSON.parse(chunk.embedding)),
    }))
    .sort((a, b) => b.score - a.score)
    .slice(0, limit);
}

function bm25Search(db: Database, query: string, limit: number) {
  // SQLite FTS5 BM25 ranking
  const rows = db.prepare(`
    SELECT rowid, rank FROM chunks_fts
    WHERE chunks_fts MATCH ?
    ORDER BY rank
    LIMIT ?
  `).all(query, limit);

  // Normalize BM25 scores to 0-1 range
  const maxRank = Math.abs(rows[0]?.rank ?? 1);
  return rows.map((r) => ({
    id: lookupIdByRowid(db, r.rowid),
    score: Math.abs(r.rank) / maxRank,
  }));
}
```

### 8.6 File Sync (`src/memory/sync.ts`)

Watches memory files for changes and incrementally reindexes.

```typescript
class MemorySync {
  private watcher: FSWatcher | null = null;
  private dirty = false;
  private debounceTimer: NodeJS.Timeout | null = null;

  start(workspaceDir: string, config: CasperConfig["memory"]): void {
    if (!config.sync.watch) return;

    const watchPaths = [
      path.join(workspaceDir, "MEMORY.md"),
      path.join(workspaceDir, "memory"),
    ];

    this.watcher = chokidar.watch(watchPaths, {
      ignoreInitial: true,
      awaitWriteFinish: {
        stabilityThreshold: config.sync.watchDebounceMs,
      },
    });

    this.watcher.on("change", () => this.markDirty());
    this.watcher.on("add", () => this.markDirty());
    this.watcher.on("unlink", () => this.markDirty());
  }

  private markDirty(): void {
    this.dirty = true;
  }

  async syncIfDirty(manager: MemoryManager): Promise<void> {
    if (!this.dirty) return;
    this.dirty = false;
    await manager.reindex();
  }
}
```

### 8.7 Memory Lifecycle

```
1. Startup:
   - Open/create SQLite database
   - Initial sync: scan MEMORY.md + memory/*.md
   - Chunk files → generate embeddings → store in SQLite
   - Start file watcher

2. Agent writes memory:
   - Agent calls write tool → memory/2026-02-17.md
   - File watcher detects change → marks dirty
   - Next memory_search call → syncIfDirty() → reindex changed file

3. Agent searches memory:
   - Agent calls memory_search("user's birthday")
   - Sync if dirty
   - Generate query embedding
   - Dual search: vector + BM25
   - Merge results → return top N snippets

4. Pre-compaction flush:
   - Session approaching context limit
   - Silent agent turn: "Store important memories"
   - Agent writes to memory/YYYY-MM-DD.md
   - Session compacted (old turns summarized)
   - Memories preserved in files → searchable later
```

---

## 9. Session Management

### 9.1 Session Storage

Sessions are stored as JSONL files (one JSON object per line):

```
sessions/
├── casper_wa_1234567890.jsonl     # Per-sender session
├── casper_wa_9876543210.jsonl
└── casper_main.jsonl              # Global/heartbeat session
```

Each file contains:

```jsonl
{"type":"session","id":"casper:wa:1234567890","cwd":"./workspace"}
{"type":"message","message":{"role":"user","content":"Hello"}}
{"type":"message","message":{"role":"assistant","content":"Hi! How can I help?"}}
{"type":"message","message":{"role":"user","content":"What's the weather?"}}
{"type":"message","message":{"role":"assistant","content":"Let me check...","tool_calls":[...]}}
{"type":"tool_result","tool_call_id":"...","content":"72°F, sunny"}
{"type":"message","message":{"role":"assistant","content":"It's 72°F and sunny today."}}
```

### 9.2 Session Reset

Sessions reset based on config:

```typescript
function shouldResetSession(
  sessionKey: string,
  lastMessageAt: number,
  config: CasperConfig["session"],
): boolean {
  if (!config.reset) return false;

  if (config.reset.mode === "daily") {
    const lastDate = new Date(lastMessageAt);
    const now = new Date();
    const resetHour = config.reset.atHour ?? 0;
    // Reset if we've crossed the reset hour boundary
    return crossedResetBoundary(lastDate, now, resetHour);
  }

  if (config.reset.mode === "idle") {
    const idleMs = (config.reset.idleMinutes ?? 60) * 60 * 1000;
    return Date.now() - lastMessageAt > idleMs;
  }

  return false;
}
```

### 9.3 History Limiting

Before feeding history to the LLM, limit to `maxHistoryTurns`:

```typescript
function limitHistory(messages: Message[], maxTurns: number): Message[] {
  // Keep the most recent N user-assistant turn pairs
  const turns: [Message, Message][] = [];
  for (let i = messages.length - 1; i >= 0; i--) {
    if (messages[i].role === "user") {
      const assistant = messages[i + 1]; // Next message should be assistant
      if (assistant?.role === "assistant") {
        turns.unshift([messages[i], assistant]);
      }
    }
  }
  return turns.slice(-maxTurns).flat();
}
```

---

## 10. Error Handling & Resilience

### 10.1 WhatsApp Reconnection

```typescript
// Exponential backoff for WhatsApp reconnects
const RECONNECT_DELAYS = [2000, 4000, 8000, 16000, 30000]; // ms

async function reconnect(attempt = 0): Promise<void> {
  const delay = RECONNECT_DELAYS[Math.min(attempt, RECONNECT_DELAYS.length - 1)];
  await sleep(delay);
  try {
    await createWhatsAppConnection(config);
  } catch (err) {
    log.error(`Reconnect attempt ${attempt + 1} failed:`, err);
    await reconnect(attempt + 1);
  }
}
```

### 10.2 LLM Error Handling

Follow OpenClaw's approach from `runEmbeddedPiAgent()`:

| Error | Recovery |
|---|---|
| Rate limit (429) | Wait `retry-after` header, retry |
| Auth failure (401/403) | Log error, fail the run, notify user |
| Context overflow | Compact → truncate tool results → reset session |
| Network timeout | Retry once with same prompt |
| Malformed response | Log, retry once |

### 10.3 Tool Error Handling

Tool errors are returned to the LLM as text content, not thrown:

```typescript
try {
  const result = await tool.execute(id, params, signal);
  return result;
} catch (err) {
  return {
    content: [{ type: "text", text: `Error: ${err.message}` }],
  };
}
```

The LLM sees the error and can decide how to proceed (retry, try alternative, inform user).

---

## 11. Implementation Plan

### Phase 1: Core Runtime (Week 1)

1. **Project scaffolding** — TypeScript project, package.json, tsconfig
2. **Config loader** — Parse casper.json, validate, apply defaults
3. **Baileys connection** — Connect, auth state, reconnection
4. **Message handler** — Receive messages, filter, extract text
5. **Agent runner** — Integrate pi-coding-agent, build system prompt
6. **Core tools** — read, write, edit, exec
7. **Response delivery** — Send agent response back to WhatsApp
8. **Session persistence** — JSONL session files, per-sender sessions

**Milestone**: Send a message on WhatsApp, get an AI response with file/exec tool access.

### Phase 2: Memory + Heartbeat (Week 2)

9. **Memory SQLite** — Schema creation, chunk storage
10. **Chunker** — Markdown → overlapping chunks
11. **Embeddings** — OpenAI embedding integration
12. **Hybrid search** — Vector + BM25 search
13. **Memory tools** — memory_search, memory_get for the agent
14. **File watcher** — Incremental reindex on file changes
15. **Heartbeat runner** — Interval timer, HEARTBEAT.md, HEARTBEAT_OK suppression
16. **Active hours** — Quiet hours for heartbeat

**Milestone**: Agent remembers things across sessions. Heartbeat checks in periodically.

### Phase 3: SDK Integrations (Week 3)

17. **Web tools** — web_search (Brave), web_fetch
18. **GitHub tool** — Octokit-based PR/issue/run management
19. **Additional integrations** — Notion, or others based on user needs
20. **Context management** — Pre-compaction memory flush, session compaction

**Milestone**: Agent can search the web, manage GitHub, and gracefully handle long conversations.

### Phase 4: Polish (Week 4)

21. **Group chat support** — Mention-based triggers in WhatsApp groups
22. **Media handling** — Image/audio/document receipt and processing
23. **Typing delay** — Human-like response timing
24. **Logging** — Structured logging to file
25. **Error recovery** — Graceful handling of all failure modes
26. **Session reset** — Daily and idle-based session reset

**Milestone**: Production-ready personal assistant.

---

## 12. Key Differences from OpenClaw

| Aspect | OpenClaw | Casper Claw |
|---|---|---|
| Channels | 6+ (WhatsApp, Telegram, Discord, Slack, Signal, etc.) | WhatsApp only (Baileys) |
| Agents | Multi-agent with bindings and routing | Single agent |
| Skills | 51 SKILL.md files wrapping CLI binaries | None — replaced by SDK tools |
| Tools | CLI-based via exec + SDK-based core tools | All SDK-based (TypeScript native) |
| Gateway | HTTP + WebSocket server with control plane | Simple process with Baileys connection |
| Config | 15+ config type files, deeply nested | Single flat config type |
| Sandbox | Docker container isolation | None (v1) |
| Cron | Full cron scheduler with isolated sessions | None (heartbeat covers periodic needs) |
| Control UI | Browser-based dashboard | None (WhatsApp is the UI) |
| Installation | npm global + CLI commands | Clone + npm install + npm start |
| Codebase | ~200+ source files, 50k+ lines | ~25 source files, ~3k lines |
