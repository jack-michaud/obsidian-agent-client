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
│  IAgentClient Interface                                         │
│  ├── AcpAdapter (Desktop)        - Local process via stdio      │
│  └── WebSocketAcpAdapter (NEW)   - Remote agent via WebSocket   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ WebSocket
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Remote Agent Server                          │
│  (Runs on desktop/server with Node.js)                          │
├─────────────────────────────────────────────────────────────────┤
│  - WebSocket server accepting connections                       │
│  - Spawns local agent processes (Claude Code, etc.)             │
│  - Proxies ACP protocol messages                                │
│  - Handles terminal operations                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Required Changes

#### 1. New WebSocket Adapter

Create `adapters/acp/websocket-acp.adapter.ts`:

```typescript
export class WebSocketAcpAdapter implements IAgentClient {
  private ws: WebSocket | null = null;
  private connection: acp.ClientSideConnection | null = null;

  async initialize(config: AgentConfig): Promise<InitializeResult> {
    // Connect to remote agent server via WebSocket
    this.ws = new WebSocket(config.remoteUrl);

    // Create streams from WebSocket
    const { input, output } = this.createStreamsFromWebSocket(this.ws);
    const stream = acp.ndJsonStream(input, output);

    // Rest of initialization...
  }
}
```

#### 2. Remote Agent Server

A separate Node.js server that:
- Accepts WebSocket connections from mobile clients
- Spawns and manages local agent processes
- Proxies ACP protocol messages between WebSocket and stdio
- Handles authentication and security

#### 3. Settings Updates

```typescript
interface AgentClientPluginSettings {
  // Existing settings...

  // New remote agent settings
  remoteAgentEnabled: boolean;
  remoteAgentUrl: string;
  remoteAgentAuthToken?: string;
}
```

#### 4. Platform-Aware Adapter Selection

```typescript
// In ChatView or plugin initialization
const adapter = Platform.isDesktopApp
  ? new AcpAdapter(plugin)
  : new WebSocketAcpAdapter(plugin);
```

### Phased Implementation

#### Phase 1: Infrastructure
- [ ] Create `WebSocketAcpAdapter` class implementing `IAgentClient`
- [ ] Update settings to support remote agent configuration
- [ ] Add platform detection for adapter selection

#### Phase 2: Remote Server
- [ ] Create standalone Node.js WebSocket server
- [ ] Implement agent process management
- [ ] Add authentication/security

#### Phase 3: Terminal Support
- [ ] Proxy terminal operations through WebSocket
- [ ] Handle terminal output streaming
- [ ] Implement timeout/cleanup

#### Phase 4: Polish
- [ ] Connection status UI
- [ ] Reconnection handling
- [ ] Error messages and troubleshooting

## Challenges and Considerations

### Security

1. **Authentication** - Remote server needs auth tokens
2. **Encryption** - Must use WSS (WebSocket Secure)
3. **Network exposure** - Server exposes agent capabilities

### Performance

1. **Latency** - Network round-trip for each message
2. **Terminal streaming** - May feel less responsive
3. **Large payloads** - Image/file transfers

### User Experience

1. **Setup complexity** - Users must run remote server
2. **Network dependency** - Requires internet/local network
3. **Error handling** - Connection drops, timeouts

### Alternative Approaches

#### Option A: Companion Desktop App
- Desktop app runs agent server
- Mobile connects to desktop on same network
- Simpler security (local network only)

#### Option B: Cloud-Hosted Agent Service
- Run agent server in cloud (user's own server)
- Better for remote access
- Higher setup complexity

#### Option C: Agent-Specific Mobile SDKs
- Some agents may offer native mobile APIs
- Would require agent-specific adapters
- Not currently available for Claude Code/Gemini

## Conclusion

### Feasibility: ✅ YES (with significant effort)

Mobile support is **technically feasible** through WebSocket transport:

1. **Obsidian supports WebSockets** on mobile
2. **ACP is transport-agnostic** and can work over WebSocket
3. **The plugin architecture** already abstracts the transport via `IAgentClient`

### Effort Estimate

| Component | Complexity | Notes |
|-----------|------------|-------|
| WebSocketAcpAdapter | Medium | New adapter implementation |
| Remote Server | High | New project, security considerations |
| Settings UI | Low | Add remote connection options |
| Testing | High | Desktop + mobile, network scenarios |

### Recommendations

1. **Start with Phase 1** - Create WebSocket adapter, validate concept
2. **Consider companion app approach** - Simpler security model
3. **Document self-hosting** - Clear instructions for running remote server
4. **Maintain desktop-first** - Keep local process support as primary
