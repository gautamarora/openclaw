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
| Memory (vector search + markdown files) | Markdown files + lazy embedding index |
| Pi runtime (agent loop + tools + streaming) | Same — `pi-coding-agent` + `pi-ai` |
| Session persistence (JSONL transcripts) | Delegate to `SessionManager` from `pi-coding-agent` |
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
│                                    │ (md files) │    │
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
├── better-sqlite3                   # Memory embedding index
├── openai                           # Embeddings SDK
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
│   │   ├── indexer.ts               # Chunk, embed, store, search (single file)
│   │   └── tools.ts                 # memory_search, memory_get tool defs
│   ├── tools/
│   │   ├── exec.ts                  # Shell execution tool
│   │   ├── fs.ts                    # Read, write, edit tools
│   │   ├── web.ts                   # web_search, web_fetch tools
│   │   ├── github.ts                # GitHub tool (octokit)
│   │   ├── notion.ts                # Notion tool
│   │   └── index.ts                 # Tool registry + assembly
│   └── sessions/
│       └── sessions.ts              # Session file path resolution + reset logic
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
    dir: string;                         // "./workspace/memory" — where .md files live
    embeddingModel?: string;             // "text-embedding-3-small" (uses OPENAI_API_KEY from env)
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
    "dir": "./workspace/memory"
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

Delegated to `pi-coding-agent`'s built-in compaction. The gateway adds a pre-compaction memory flush hook (see Section 9.5) so memories are preserved before old turns are summarized. If compaction fails, the session is reset.

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
// memory_search: Vector search over memory markdown files
{ name: "memory_search", params: { query: string } }

// memory_get: Read a specific memory file (or line range within it)
{ name: "memory_get", params: { path: string, from?: number, lines?: number } }
```

See Section 8 for details. Uses lazy-sync embedding index over workspace markdown files.

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

### 8.1 Design Philosophy

OpenClaw's memory system is a full embedded search engine: SQLite, FTS5 virtual tables, vector embeddings, hybrid BM25+cosine scoring, embedding caches, file watchers with chokidar, and atomic database swaps for reindexing. That's appropriate for a multi-agent system indexing thousands of files across multiple workspaces.

For a single personal WhatsApp bot, we simplify radically. The agent already has a `read` tool and a `write` tool. Memory is just **markdown files the agent reads and writes**. The only addition is a `memory_search` tool that does vector search over those files so the agent can find things without reading every file.

### 8.2 What's on Disk

```
workspace/
├── MEMORY.md              # Curated long-term facts (agent maintains this)
└── memory/
    ├── 2026-02-15.md      # Daily log
    ├── 2026-02-16.md
    ├── 2026-02-17.md
    └── .index.sqlite      # Embedding index (auto-generated, gitignored)
```

**MEMORY.md** is the primary knowledge base. The agent is instructed (via AGENTS.md) to keep it organized: user preferences, key decisions, recurring topics, important facts.

**memory/YYYY-MM-DD.md** files are daily append-only logs. The agent writes here during conversation when it learns something worth remembering, and during pre-compaction flushes.

**.index.sqlite** is a derived artifact. Delete it and it rebuilds from the .md files on next search.

### 8.3 How It Works

```
Agent calls memory_search("user's birthday")
  │
  ├─ 1. Scan memory dir for .md files
  │     Compare file mtimes against last indexed mtimes
  │     Re-chunk + re-embed only changed files
  │
  ├─ 2. Embed the query via OpenAI
  │     text-embedding-3-small → 1536-dim vector
  │
  ├─ 3. Cosine similarity against all stored chunks
  │     Sort by score, return top 5
  │
  └─ Returns: [{ file, lines, score, snippet }]
```

No hybrid search, no BM25, no FTS5. Just vector similarity. For a personal memory corpus (likely <100 files, <1000 chunks), brute-force cosine over all chunks is fast enough — OpenClaw itself falls back to this when the sqlite-vec extension isn't available.

### 8.4 SQLite Schema

Minimal. Two tables.

```sql
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  hash TEXT NOT NULL,           -- SHA-256 of file content
  mtime INTEGER NOT NULL
);

CREATE TABLE chunks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  path TEXT NOT NULL,
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  text TEXT NOT NULL,
  embedding BLOB NOT NULL,     -- Float32Array as raw bytes (not JSON)
  FOREIGN KEY (path) REFERENCES files(path) ON DELETE CASCADE
);
```

That's it. No FTS5, no embedding cache table, no metadata table. When a file changes (hash differs), delete its old chunks and reinsert. When a file is deleted, `ON DELETE CASCADE` cleans up.

Embeddings are stored as raw `Float32Array` buffers (6KB per chunk at 1536 dims) instead of JSON strings (~12KB per chunk). For a personal corpus this doesn't matter much, but it's simpler to work with in code.

### 8.5 Chunking

Same approach as OpenClaw but hardcoded instead of configurable:

```typescript
const CHUNK_CHARS = 1600;     // ~400 tokens
const OVERLAP_CHARS = 320;    // ~80 tokens

function chunkFile(path: string, content: string): Chunk[] {
  const lines = content.split("\n");
  const chunks: Chunk[] = [];
  let start = 0;
  let charCount = 0;

  for (let i = 0; i < lines.length; i++) {
    charCount += lines[i].length + 1;
    if (charCount >= CHUNK_CHARS || i === lines.length - 1) {
      chunks.push({
        path,
        startLine: start + 1,
        endLine: i + 1,
        text: lines.slice(start, i + 1).join("\n"),
      });
      // Walk back for overlap
      let overlapChars = 0;
      let overlapStart = i;
      while (overlapStart > start && overlapChars < OVERLAP_CHARS) {
        overlapChars += lines[overlapStart].length + 1;
        overlapStart--;
      }
      start = overlapStart + 1;
      charCount = 0;
    }
  }
  return chunks;
}
```

### 8.6 Memory Tools

Two tools exposed to the agent:

```typescript
// memory_search: Find relevant memories
{
  name: "memory_search",
  description: "Search long-term memory for relevant information.",
  parameters: {
    type: "object",
    properties: {
      query: { type: "string", description: "What to search for" },
    },
    required: ["query"],
  },
  execute: async (id, { query }) => {
    await indexer.syncIfNeeded();                    // Re-index changed files
    const queryVec = await embed(query);             // Embed query
    const results = indexer.search(queryVec, 5);     // Top 5 by cosine sim
    return { content: [{ type: "text", text: formatResults(results) }] };
  },
}

// memory_get: Read a specific memory file (or range of lines)
{
  name: "memory_get",
  description: "Read a memory file. Use after memory_search to see full context.",
  parameters: {
    type: "object",
    properties: {
      path: { type: "string", description: "Relative path within memory dir" },
      from: { type: "number", description: "Start line (1-indexed)" },
      lines: { type: "number", description: "Number of lines to read" },
    },
    required: ["path"],
  },
  execute: async (id, { path: filePath, from, lines }) => {
    const content = readMemoryFile(filePath, from, lines);
    return { content: [{ type: "text", text: content }] };
  },
}
```

### 8.7 Sync Strategy

**Lazy sync on search** — no file watchers, no chokidar dependency. When `memory_search` is called:

1. List all `.md` files in the memory dir
2. For each file, compare `fs.statSync(file).mtimeMs` against stored mtime
3. If changed: re-read, re-chunk, re-embed, update in SQLite
4. If new file: chunk, embed, insert
5. If file deleted: delete from SQLite (cascade removes chunks)

This is fast because the corpus is small and embedding calls are batched. OpenAI's embedding API accepts arrays of texts, so we send all new/changed chunks in one call.

### 8.8 Memory Lifecycle

```
1. Startup:
   - Nothing. Index is built lazily on first memory_search.

2. During conversation:
   - Agent learns something → writes to memory/YYYY-MM-DD.md via write tool
   - Agent needs to recall → calls memory_search → lazy sync → vector search

3. Pre-compaction flush:
   - Session approaching context limit
   - Gateway injects a silent prompt: "Store important memories from this conversation"
   - Agent writes to memory/YYYY-MM-DD.md
   - pi-coding-agent compacts the session (summarizes old turns)
   - Memories survive in files, searchable via memory_search later

4. Index corruption / stale:
   - Delete .index.sqlite → rebuilds on next search
```

---

## 9. Session Management

### 9.1 Design Philosophy

OpenClaw's session system handles multi-agent, multi-channel, multi-account routing with complex key generation (`agent:<id>:<channel>:<kind>:<peerId>:thread:<threadId>`), write locking with 10s timeouts, atomic file renames, 45s in-memory caches, session branching, topic splitting, and maintenance policies (prune after 30 days, cap 500 entries, rotate at 10MB).

Casper Claw has one agent, one channel. We delegate to `pi-coding-agent`'s `SessionManager` which already handles JSONL persistence. We just need to tell it which file to use.

### 9.2 Session Storage

pi-coding-agent's `SessionManager` stores sessions as JSONL files. We just pick the file path:

```
sessions/
├── 1234567890.jsonl         # Per-sender (JID without @s.whatsapp.net)
├── 9876543210.jsonl
└── _main.jsonl              # Heartbeat / system session
```

Each file is managed entirely by `SessionManager.open(filePath)`. It handles:
- Appending messages (user, assistant, tool_use, tool_result)
- Building the message array for LLM calls
- Compaction (summarizing old turns when context gets full)
- Reading back the session on process restart

We don't wrap, cache, or lock these files — `SessionManager` does it.

### 9.3 Session Key → File Path

```typescript
function sessionFilePath(jid: string, config: CasperConfig): string {
  if (config.session.scope === "global") {
    return path.join(config.session.dir, "_main.jsonl");
  }
  const senderId = jid.split("@")[0];   // "1234567890"
  return path.join(config.session.dir, `${senderId}.jsonl`);
}
```

### 9.4 Session Reset

Two reset modes, checked before each agent run:

```typescript
function shouldReset(sessionFile: string, config: CasperConfig["session"]): boolean {
  if (!config.reset) return false;

  const stat = fs.statSync(sessionFile, { throwIfNoEntry: false });
  if (!stat) return false;  // No session file yet

  const lastModified = stat.mtimeMs;
  const now = Date.now();

  if (config.reset.mode === "idle") {
    const idleMs = (config.reset.idleMinutes ?? 60) * 60_000;
    return now - lastModified > idleMs;
  }

  if (config.reset.mode === "daily") {
    const resetHour = config.reset.atHour ?? 4;
    const lastDate = new Date(lastModified);
    const today = new Date();
    today.setHours(resetHour, 0, 0, 0);
    // Reset if the session file was last written before today's reset hour
    return lastDate < today && new Date() >= today;
  }

  return false;
}

// Reset = rename old file, start fresh
function resetSession(sessionFile: string): void {
  const backup = sessionFile.replace(".jsonl", `.${Date.now()}.jsonl`);
  fs.renameSync(sessionFile, backup);
}
```

Also supports `/new` and `/reset` as text triggers — if the incoming message matches, reset the session before running the agent.

### 9.5 Compaction

Handled entirely by `pi-coding-agent`. When the context window fills up:

1. **Pre-compaction memory flush**: Before compacting, the gateway runs a silent agent turn with the prompt "Store important memories from this conversation to memory files." The agent writes to `memory/YYYY-MM-DD.md`. This ensures long-term knowledge is preserved before old turns are summarized away.

2. **Compaction**: `SessionManager.compact()` summarizes old conversation turns into a shorter summary, freeing context space. The session continues with the summary + recent turns.

3. **If still too long**: The gateway resets the session and tells the user the conversation was too long.

We don't implement our own compaction logic. We call `session.compact()` and let the library handle it.

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

9. **Memory indexer** — SQLite schema, chunking, OpenAI embeddings, cosine search (single file)
10. **Memory tools** — memory_search, memory_get tool definitions
11. **Heartbeat runner** — Interval timer, HEARTBEAT.md, HEARTBEAT_OK suppression
12. **Active hours** — Quiet hours for heartbeat
13. **Pre-compaction flush** — Memory flush hook before session compaction

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
