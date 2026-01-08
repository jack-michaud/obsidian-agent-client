# Mobile Support Feasibility Assessment

This document analyzes the feasibility of adding mobile support to the Agent Client plugin via WebSocket transport for remote agents.

## Current State

The Agent Client plugin is currently **desktop-only** due to its reliance on Node.js-specific APIs for process spawning and terminal management.

### Desktop-Only Dependencies

| Component | File | Desktop-Only API | Purpose |
|-----------|------|------------------|---------|
| AcpAdapter | `adapters/acp/acp.adapter.ts` | `child_process.spawn` | Spawns agent process (Claude Code, Gemini CLI, etc.) |
| AcpAdapter | `adapters/acp/acp.adapter.ts` | `process.env` | Passes environment variables to agent |
| TerminalManager | `shared/terminal-manager.ts` | `child_process.spawn` | Creates terminal processes for agent tool calls |
| TerminalManager | `shared/terminal-manager.ts` | `process.env`, `process.cwd()` | Environment and working directory |
| TerminalManager | `shared/terminal-manager.ts` | `Buffer` | Byte length calculations for output truncation |
| ChatView | `components/chat/ChatView.tsx` | `process.cwd()` | Fallback for vault path |
| WSL Utils | `shared/wsl-utils.ts` | Windows-specific paths | WSL mode for Windows |

### Platform Checks

The codebase includes explicit platform checks:
- `ChatView.tsx:58` - Throws error if `!Platform.isDesktopApp`
- `terminal-manager.ts:35` - Throws error if `!Platform.isDesktopApp`

## WebSocket Support in Obsidian

### Available Network APIs

| API | Desktop | Mobile | Notes |
|-----|---------|--------|-------|
| `requestUrl()` | ✅ | ✅ | HTTP requests, bypasses CORS |
| `WebSocket` | ✅ | ✅ | Standard WebSocket API |
| `fetch()` | ✅ | ⚠️ | May be blocked by CORS |

### Evidence of WebSocket Support

1. **Obsidian Sync** uses WebSockets for real-time synchronization
2. **Community plugins** successfully use WebSocket connections (e.g., MCP server plugin on port 22360)
3. Standard browser `WebSocket` API is available in Obsidian's Chromium/WebView runtime

## ACP Protocol Transport

### Current Implementation

The plugin currently uses **stdio transport** (stdin/stdout):

```typescript
// acp.adapter.ts
const stream = acp.ndJsonStream(input, output);
this.connection = new acp.ClientSideConnection(() => this, stream);
```

Where `input` and `output` are created from the spawned process's stdin/stdout streams.

### ACP Transport Architecture

ACP is designed to be **transport-agnostic**:
- Uses JSON-RPC 2.0 for message format
- `ndJsonStream` accepts any `WritableStream`/`ReadableStream` pair
- Remote agents are mentioned in the ACP specification (work in progress)

### WebSocket Transport Feasibility

The `ndJsonStream` function can work with WebSocket-backed streams:

```typescript
// Proposed WebSocket transport
const ws = new WebSocket('wss://agent-server.example.com');

const input = new WritableStream<Uint8Array>({
  write(chunk) {
    ws.send(chunk);
  }
});

const output = new ReadableStream<Uint8Array>({
  start(controller) {
    ws.onmessage = (event) => {
      controller.enqueue(new Uint8Array(event.data));
    };
  }
});

const stream = acp.ndJsonStream(input, output);
```

## Implementation Plan

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Obsidian (Desktop/Mobile)                   │
├─────────────────────────────────────────────────────────────────┤
│  AcpAdapter (Extended)                                          │
│  ├── Local Mode (Desktop)    - Spawns process, stdio transport  │
│  │                             terminal: true                   │
│  └── Remote Mode (Any)       - WebSocket transport              │
│                                terminal: false                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ WebSocket (JSON-RPC messages only)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Remote Agent Server                          │
│  (Runs on desktop/server with Node.js)                          │
├─────────────────────────────────────────────────────────────────┤
│  - WebSocket server accepting connections                       │
│  - Spawns local agent processes (Claude Code, etc.)             │
│  - Proxies ACP protocol messages (no terminal execution)        │
└─────────────────────────────────────────────────────────────────┘
```

### Required Changes

#### 1. Extend AcpAdapter with Remote Support

Modify `adapters/acp/acp.adapter.ts` to support both local and remote connections:

```typescript
export class AcpAdapter implements IAgentClient, IAcpClient {
  private connection: acp.ClientSideConnection | null = null;
  private agentProcess: ChildProcess | null = null;  // Only used in local mode
  private ws: WebSocket | null = null;               // Only used in remote mode
  private isRemoteMode: boolean = false;

  async initialize(config: AgentConfig): Promise<InitializeResult> {
    // Determine mode based on config
    this.isRemoteMode = !!config.remoteUrl;

    let stream: ReturnType<typeof acp.ndJsonStream>;

    if (this.isRemoteMode) {
      // Remote mode: connect via WebSocket
      stream = await this.initializeRemoteConnection(config.remoteUrl);
    } else {
      // Local mode: spawn process (existing logic)
      stream = await this.initializeLocalProcess(config);
    }

    this.connection = new acp.ClientSideConnection(() => this, stream);

    // Initialize with appropriate capabilities
    // Remote mode: client handles filesystem (agent can't access mobile vault)
    // Local mode: agent handles filesystem directly, no terminal capability needed
    const initResult = await this.connection.initialize({
      protocolVersion: acp.PROTOCOL_VERSION,
      clientCapabilities: {
        fs: {
          readTextFile: this.isRemoteMode,   // Client handles file reads in remote mode
          writeTextFile: this.isRemoteMode,  // Client handles file writes in remote mode
        },
        terminal: !this.isRemoteMode,  // No terminal in remote mode
      },
    });

    // Rest of initialization...
  }

  // ... initializeRemoteConnection, initializeLocalProcess ...

  // ========================================================================
  // File System Operations (used in remote mode)
  // ========================================================================

  async readTextFile(params: acp.ReadTextFileRequest): Promise<acp.ReadTextFileResponse> {
    // Use Obsidian Vault API to read files
    // https://docs.obsidian.md/Reference/TypeScript+API/Vault/read
    const file = this.plugin.app.vault.getAbstractFileByPath(params.path);
    if (!file || !(file instanceof TFile)) {
      throw new Error(`File not found: ${params.path}`);
    }
    const content = await this.plugin.app.vault.read(file);
    return { content };
  }

  async writeTextFile(params: acp.WriteTextFileRequest): Promise<acp.WriteTextFileResponse> {
    // Use Obsidian Vault API to write files
    // https://docs.obsidian.md/Reference/TypeScript+API/Vault/modify
    const file = this.plugin.app.vault.getAbstractFileByPath(params.path);
    if (file && file instanceof TFile) {
      await this.plugin.app.vault.modify(file, params.content);
    } else {
      // Create new file if it doesn't exist
      await this.plugin.app.vault.create(params.path, params.content);
    }
    return {};
  }
}
```

**Key difference between modes:**

| Mode | Filesystem | Terminal | Why |
|------|------------|----------|-----|
| Local (Desktop) | Agent handles directly | ✅ Client spawns processes | Agent runs locally with full access |
| Remote (Mobile) | ✅ Client via Vault API | ❌ Not available | Agent can't access mobile filesystem |

#### 2. Remote Agent Server (Already Exists)

An ACP Remote Server with WebSocket support has already been developed by the plugin author. This server:
- Accepts WebSocket connections from mobile/remote clients
- Spawns and manages local agent processes
- Proxies ACP protocol messages between WebSocket and stdio
- Handles authentication and security

The remote server will be published separately and can be self-hosted by users who want mobile support.

#### 3. Settings Updates

```typescript
interface AgentClientPluginSettings {
  // Existing settings...

  // New remote agent settings
  remoteAgent: {
    enabled: boolean;
    url: string;           // WebSocket URL (wss://...)
    authToken?: string;    // Optional auth token
  };
}
```

#### 4. Platform Detection & Mode Selection

```typescript
// In ChatView or useAgentSession
const shouldUseRemote = !Platform.isDesktopApp || settings.remoteAgent.enabled;

const config: AgentConfig = {
  ...baseConfig,
  remoteUrl: shouldUseRemote ? settings.remoteAgent.url : undefined,
};
```

#### 5. Update Platform Checks

Remove or modify platform checks to allow mobile when remote is configured:

```typescript
// ChatView.tsx - Update platform check
if (!Platform.isDesktopApp && !settings.remoteAgent.enabled) {
  // Show "Configure remote agent" UI instead of throwing error
}

// terminal-manager.ts - Already gated by terminal capability
// No changes needed - won't be called when terminal: false
```

### Phased Implementation

#### Phase 1: AcpAdapter Remote Support
- [ ] Add `initializeRemoteConnection()` method to AcpAdapter
- [ ] Add mode detection based on `config.remoteUrl`
- [ ] Set capabilities based on mode:
  - Remote: `fs: { readTextFile: true, writeTextFile: true }, terminal: false`
  - Local: `fs: { readTextFile: false, writeTextFile: false }, terminal: true`
- [ ] Implement `readTextFile()` using Obsidian Vault API (`vault.read()`)
- [ ] Implement `writeTextFile()` using Obsidian Vault API (`vault.modify()`, `vault.create()`)
- [ ] Handle WebSocket lifecycle (connect, disconnect, reconnect)

#### Phase 2: Settings & UI
- [ ] Add remote agent settings to `AgentClientPluginSettings`
- [ ] Add settings UI for remote connection configuration
- [ ] Update platform checks in ChatView to allow mobile with remote config
- [ ] Add connection status indicator

#### Phase 3: Remote Server ✅ (Complete)
- [x] Create standalone Node.js WebSocket server
- [x] Implement agent process management
- [x] Proxy ACP protocol messages
- [x] Add authentication/security

#### Phase 4: Polish & Documentation
- [ ] Reconnection handling with backoff
- [ ] Error messages and troubleshooting UI
- [ ] Documentation for self-hosting the remote server
- [ ] Mobile-specific UX improvements

> **Note:** Terminal operations are NOT proxied through the remote server. Instead, the client declares `terminal: false` in its capabilities during ACP initialization. The agent will adapt its behavior accordingly (e.g., providing instructions instead of executing). This is the proper ACP approach - capability negotiation, not remote execution.

## Capability Negotiation

### The ACP Way

ACP includes built-in capability negotiation during initialization. The client declares what it supports, and the agent adapts its behavior accordingly.

### Capability Matrix by Mode

```typescript
// Local mode (Desktop): Agent handles files, client handles terminal
clientCapabilities: {
  fs: { readTextFile: false, writeTextFile: false },  // Agent has direct access
  terminal: true,                                      // Client spawns processes
}

// Remote mode (Mobile/Any): Client handles files, no terminal
clientCapabilities: {
  fs: { readTextFile: true, writeTextFile: true },    // Client uses Vault API
  terminal: false,                                     // Not available remotely
}
```

### Why This Split?

| Capability | Local Mode | Remote Mode | Reason |
|------------|------------|-------------|--------|
| `fs.readTextFile` | ❌ Agent handles | ✅ Client handles | Agent can't access mobile vault |
| `fs.writeTextFile` | ❌ Agent handles | ✅ Client handles | Agent can't access mobile vault |
| `terminal` | ✅ Client spawns | ❌ Not available | Security - no remote code execution |

### File System Implementation (Remote Mode)

When the agent requests file operations, the client uses Obsidian's Vault API:

- **Read**: `vault.read(file)` - [Vault.read docs](https://docs.obsidian.md/Reference/TypeScript+API/Vault/read)
- **Write**: `vault.modify(file, content)` - [Vault.modify docs](https://docs.obsidian.md/Reference/TypeScript+API/Vault/modify)
- **Create**: `vault.create(path, content)` - [Vault.create docs](https://docs.obsidian.md/Reference/TypeScript+API/Vault/create)

This allows the remote agent to work with files in the user's vault even though it runs on a different machine.

### Why NOT Proxy Terminal Commands

Proxying terminal operations through the remote server would be problematic:
- **Security risk** - Remote code execution vulnerabilities
- **Context mismatch** - Server's filesystem ≠ user's intent
- **Complexity** - Would need to handle stdout/stderr streaming, exit codes, timeouts
- **Not the ACP way** - Capability negotiation exists for this reason

The proper approach is to let the agent adapt to the client's capabilities.

## Challenges and Considerations

### Security

1. **Authentication** - Remote server needs auth tokens
2. **Encryption** - Must use WSS (WebSocket Secure)
3. **Network exposure** - Server exposes agent capabilities

### Performance

1. **Latency** - Network round-trip for each message
2. **Large payloads** - Image/file transfers may be slower

### User Experience

1. **Setup complexity** - Users must run remote server
2. **Network dependency** - Requires internet/local network
3. **Error handling** - Connection drops, timeouts

### Deployment Options for Remote Server

#### Option A: Local Network (Recommended)
- Run remote server on desktop/home server
- Mobile connects on same local network
- Simpler security (no internet exposure)
- Lower latency

#### Option B: Cloud-Hosted
- Run remote server in cloud (user's own VPS)
- Accessible from anywhere
- Requires proper security configuration (WSS, auth tokens)

#### Option C: Future - Agent-Specific Mobile SDKs
- Some agents may eventually offer native mobile APIs
- Would require agent-specific adapters
- Not currently available for Claude Code/Gemini

## Conclusion

### Feasibility: ✅ YES

Mobile support is **technically feasible** through WebSocket transport, and the most complex component (the remote server) has already been developed:

1. **Obsidian supports WebSockets** on mobile
2. **ACP is transport-agnostic** and can work over WebSocket
3. **The plugin architecture** already abstracts the transport via `IAgentClient`

### Effort Estimate

| Component | Complexity | Status |
|-----------|------------|--------|
| AcpAdapter remote mode | Medium | To do - add WebSocket transport |
| Filesystem via Vault API | Low | To do - implement `readTextFile`/`writeTextFile` |
| Remote Server | High | ✅ Complete - already developed |
| Settings UI | Low | To do - add remote connection options |
| Platform checks | Low | To do - update ChatView to allow mobile |
| Testing | Medium | To do - desktop + mobile scenarios |

### Recommendations

1. **Extend AcpAdapter** - Add remote mode to existing adapter rather than creating a new class
2. **Document self-hosting** - Clear instructions for users to run the remote server
3. **Maintain desktop-first** - Keep local process support as the default for desktop users
4. **Graceful fallback** - Show "Configure remote agent" UI on mobile if not configured
