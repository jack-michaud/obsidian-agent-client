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

#### Phase 1: Plugin Infrastructure
- [ ] Create `WebSocketAcpAdapter` class implementing `IAgentClient`
- [ ] Update settings to support remote agent configuration
- [ ] Add platform detection for adapter selection

#### Phase 2: Remote Server ✅ (Complete)
- [x] Create standalone Node.js WebSocket server
- [x] Implement agent process management
- [x] Proxy terminal operations through WebSocket
- [x] Add authentication/security

#### Phase 3: Integration & Polish
- [ ] Connection status UI
- [ ] Reconnection handling
- [ ] Error messages and troubleshooting
- [ ] Documentation for self-hosting the remote server

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
| WebSocketAcpAdapter | Medium | To do - new adapter implementation |
| Remote Server | High | ✅ Complete - already developed |
| Settings UI | Low | To do - add remote connection options |
| Testing | Medium | To do - desktop + mobile scenarios |

### Recommendations

1. **Implement WebSocketAcpAdapter** - Create the plugin-side adapter to connect to the existing remote server
2. **Document self-hosting** - Clear instructions for users to run the remote server
3. **Maintain desktop-first** - Keep local process support as the default for desktop users
4. **Graceful fallback** - Show helpful message on mobile if remote server is not configured
