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
| Memory (vector search + markdown files) | A single `MEMORY.md` file the agent reads/writes |
| Pi runtime (agent loop + tools + streaming) | Same — `pi-coding-agent` + `pi-ai` |
| Session persistence (JSONL transcripts) | `SessionManager` from `pi-coding-agent` — one JSONL file per sender |
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
│   │   └── message-handler.ts       # Inbound message dispatch
│   ├── agent/
│   │   ├── runner.ts                # Wraps runEmbeddedPiAgent()
│   │   ├── system-prompt.ts         # System prompt builder
│   │   ├── tools.ts                 # Tool assembly + policy
│   │   └── identity.ts              # Agent name, prefix, emoji
│   ├── heartbeat/
│   │   ├── runner.ts                # Interval timer + execution
│   │   ├── visibility.ts            # Suppress/deliver logic
│   │   └── active-hours.ts          # Quiet hours check
│   ├── tools/
│   │   ├── exec.ts                  # Shell execution tool
│   │   ├── fs.ts                    # Read, write, edit tools
│   │   ├── web.ts                   # web_search, web_fetch tools
│   │   ├── github.ts                # GitHub tool (octokit)
│   │   ├── notion.ts                # Notion tool
│   │   └── index.ts                 # Tool registry + assembly
├── casper.json                      # Configuration file
├── workspace/
│   ├── AGENTS.md                    # Agent operating instructions
│   ├── SOUL.md                      # Personality + tone
│   ├── HEARTBEAT.md                 # Periodic check instructions
│   └── MEMORY.md                    # Long-term memory (agent reads/writes this)
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
    dir: string;                         // "./sessions" — where JSONL files live
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
    "dir": "./sessions"
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
├─ 3. Initialize agent
│     ├─ Resolve model + auth
│     ├─ Build system prompt
│     └─ Assemble tools
├─ 4. Start heartbeat runner
│     ├─ Calculate next due time
│     └─ Schedule timer
├─ 5. RUNNING — process messages
│
└─ On SIGINT/SIGTERM:
    ├─ Stop heartbeat timer
    ├─ Close Baileys connection
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

  // 4. Resolve session file
  const senderId = msg.jid.split("@")[0];
  const sessionFile = path.join(ctx.config.session.dir, `${senderId}.jsonl`);

  // 5. Send ack reaction (eyes emoji)
  await ctx.wa.sendReaction(msg.jid, msg.messageId, "👀");

  // 6. Dispatch to agent
  const result = await ctx.agent.run({
    prompt: msg.text,
    sessionFile,
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

### 4.4 Concurrency Control

Only one agent run per sender at a time. If a message arrives while the agent is already processing for that sender, queue it.

```typescript
const active = new Set<string>();
const queue = new Map<string, InboundMessage[]>();

async function dispatch(msg: InboundMessage): Promise<void> {
  const sender = msg.jid;

  if (active.has(sender)) {
    const pending = queue.get(sender) ?? [];
    pending.push(msg);
    queue.set(sender, pending);
    return;
  }

  active.add(sender);
  try {
    await handleInboundMessage(msg, ctx);
    while (queue.get(sender)?.length) {
      await handleInboundMessage(queue.get(sender)!.shift()!, ctx);
    }
  } finally {
    active.delete(sender);
    queue.delete(sender);
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
  sessionFile: string;
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
      sessionManager: SessionManager.open(params.sessionFile),
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
    // ... dynamically list enabled integrations
  );

  // 3. Memory
  sections.push(
    `## Memory`,
    `MEMORY.md is your long-term memory. Read it at the start of conversations`,
    `to recall context. Write to it when you learn important facts, preferences,`,
    `or decisions that should persist across sessions.`,
  );

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

Delegated to `pi-coding-agent`'s built-in compaction. If compaction fails, the session is reset.

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
minimal:    read, write, edit
coding:     minimal + exec, web_search, web_fetch
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

    // 5. Run agent in the heartbeat session
    const sessionFile = path.join(this.config.session.dir, "_heartbeat.jsonl");

    const result = await this.agent.run({
      prompt,
      sessionFile,
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

Memory is a single markdown file: `workspace/MEMORY.md`.

The agent reads it (via the `read` tool) and writes to it (via the `write` or `edit` tool). There is no database, no embeddings, no indexing, no search tool. The file is small enough to fit in context — the agent just reads the whole thing when it needs to remember something.

### 8.1 How It Works

```
workspace/MEMORY.md
```

That's it. One file. The system prompt tells the agent:

> MEMORY.md is your long-term memory. Read it at the start of important conversations
> to recall context. Write to it when you learn important facts, preferences,
> or decisions that should persist across sessions.

The agent uses the `read` and `edit` tools it already has. No new tools, no new infrastructure.

### 8.2 Why This Is Enough

A personal WhatsApp bot's memory needs are small: the user's name, preferences, a few key decisions, ongoing projects. That's a few KB of markdown — easily fits in a single context window read. If it grows too large, the agent can reorganize it (move old items to an archive section, summarize, etc.) using the same `edit` tool.

OpenClaw needs vector search because it indexes thousands of files across multiple workspaces for multiple agents. Casper Claw has one user, one agent, one file.

### 8.3 Example MEMORY.md

```markdown
# Memory

## User
- Name: Alex
- Timezone: America/New_York
- Prefers concise responses

## Preferences
- GitHub: uses gautamarora org
- Coffee order: oat milk latte
- Preferred model for code: claude-sonnet

## Active Projects
- Casper Claw — WhatsApp bot project, TS-based
- Q1 planning — deadline March 15

## Key Decisions
- 2026-02-15: Decided to use Baileys instead of whatsapp-web.js
- 2026-02-16: Chose pi-coding-agent over building custom agent loop
```

### 8.4 Future: If the File Gets Too Big

If MEMORY.md grows beyond what fits comfortably in context (~20KB), the simplest upgrade is to split it into a few files (`MEMORY.md`, `MEMORY-archive.md`) and add a lightweight search. But that's a v2 problem. Start with the simplest thing.

---

## 9. Session Management

A session is the current conversation. `pi-coding-agent`'s `SessionManager` handles all the mechanics (JSONL persistence, message appending, compaction). We just tell it which file to use.

### 9.1 One JSONL File Per Sender

```
sessions/
├── 1234567890.jsonl         # Per-sender (JID without @s.whatsapp.net)
├── 9876543210.jsonl
└── _heartbeat.jsonl         # Heartbeat session
```

```typescript
function sessionFile(jid: string): string {
  const senderId = jid.split("@")[0];
  return path.join("./sessions", `${senderId}.jsonl`);
}
```

`SessionManager.open(filePath)` handles everything: appending messages, building the message array for LLM calls, persisting to disk, and reading it back on restart.

### 9.2 Compaction

When the context window fills up, `pi-coding-agent` automatically compacts the session — summarizes old turns into a shorter summary, keeps recent turns intact. We don't implement this ourselves.

### 9.3 Starting Fresh

The user can send `/new` to start a fresh conversation. The gateway renames the old file and the next message creates a new one.

```typescript
if (msg.text.trim() === "/new") {
  const file = sessionFile(msg.jid);
  if (fs.existsSync(file)) {
    fs.renameSync(file, file.replace(".jsonl", `.${Date.now()}.bak.jsonl`));
  }
  await wa.sendMessage(msg.jid, "Fresh conversation started.");
  return;
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

### Phase 2: Heartbeat + Memory (Week 2)

9. **Heartbeat runner** — Interval timer, HEARTBEAT.md, HEARTBEAT_OK suppression
10. **Active hours** — Quiet hours for heartbeat
11. **MEMORY.md** — System prompt instructions for reading/writing memory file
12. **Workspace files** — AGENTS.md, SOUL.md injection into system prompt

**Milestone**: Agent remembers things via MEMORY.md. Heartbeat checks in periodically.

### Phase 3: SDK Integrations (Week 3)

13. **Web tools** — web_search (Brave), web_fetch
14. **GitHub tool** — Octokit-based PR/issue/run management
15. **Additional integrations** — Notion, or others based on user needs

**Milestone**: Agent can search the web, manage GitHub, and gracefully handle long conversations.

### Phase 4: Polish (Week 4)

16. **Group chat support** — Mention-based triggers in WhatsApp groups
17. **Media handling** — Image/audio/document receipt and processing
18. **Typing delay** — Human-like response timing
19. **Logging + error recovery** — Structured logging, graceful failure handling

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
