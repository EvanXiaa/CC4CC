# Bridge System Analysis

> **Educational Analysis Only** - No source code included.

---

## Overview

The bridge system provides bidirectional communication between Claude Code CLI and IDE extensions (VS Code, JetBrains). This enables rich IDE integration while keeping the CLI as the core engine.

---

## Extend Analysis

> Community-contributed extended analyses are stored in [analysis/](./analysis/) directory.

**Quick Link**: [analysis/](./analysis/) - View all extended analyses for this module

| PR | Title | Author | Description |
|----|-------|--------|-------------|
| - | - | - | *Waiting for contributions* |

---

## Architecture

### Communication Flow

```
┌─────────────────┐         ┌─────────────────┐
│   IDE Extension │◄───────►│   Bridge Layer  │◄───────►│ Claude Code CLI │
└─────────────────┘         └─────────────────┘
```

### Core Components

| Component | Purpose |
|-----------|---------|
| `bridgeMain.ts` | Main event loop for bridge |
| `bridgeMessaging.ts` | Message protocol definition |
| `bridgePermissionCallbacks.ts` | Permission request handling |
| `replBridge.ts` | REPL session bridging |
| `jwtUtils.ts` | JWT-based authentication |
| `sessionRunner.ts` | Session execution management |

---

## Message Protocol

### Message Types

| Type | Direction | Purpose |
|------|-----------|---------|
| Request | IDE → CLI | Initiate an action |
| Response | CLI → IDE | Return result |
| Event | CLI → IDE | Notify of state change |
| Permission | CLI → IDE | Request user approval |

### Message Structure

| Field | Purpose |
|-------|---------|
| Type | Message category |
| ID | Correlation identifier |
| Payload | Message-specific data |
| Timestamp | When message was sent |

---

## Key Features

### Session Management

- **Start sessions from IDE** - Launch Claude Code from editor
- **Resume sessions** - Continue previous conversations
- **Session isolation** - Multiple concurrent sessions

### Permission Handling

- **Permission requests routed to IDE** - User approves in editor UI
- **Batch approvals** - Approve multiple operations at once
- **Remember decisions** - Cache approval decisions

### File Synchronization

- **Open files in IDE** - Claude can open files for editing
- **Change notifications** - IDE notifies Claude of file changes
- **Cursor positioning** - Navigate to specific locations

---

## Authentication

### JWT-Based Auth

| Component | Purpose |
|-----------|---------|
| Token Generation | Create session tokens |
| Token Validation | Verify incoming requests |
| Refresh Flow | Maintain long-lived sessions |

### Security Considerations

- Tokens have limited lifetime
- Scope-limited permissions
- Secure token storage

---

## Integration Patterns

### VS Code Integration

- Extension activates bridge on startup
- Uses VS Code's webview for UI
- Leverages VS Code's authentication API

### JetBrains Integration

- Plugin-based architecture
- Uses IDE's notification system
- Integrates with VCS operations

---

## Key Design Patterns

### 1. Protocol Abstraction

The bridge protocol is IDE-agnostic:
- Same protocol for VS Code and JetBrains
- Easy to add new IDE support
- Versioned for compatibility

### 2. Asynchronous Messaging

All communication is async:
- Non-blocking operations
- Timeout handling
- Cancellation support

### 3. State Synchronization

Bidirectional state sync:
- CLI notifies IDE of state changes
- IDE can query CLI state
- Conflict resolution for concurrent changes

---

## Lessons for Builders

1. **Protocol-first design** enables multiple clients
2. **Async messaging** prevents UI blocking
3. **State synchronization** is complex but essential
4. **Security boundaries** must be explicit

---

## Contributing

We welcome contributions to this analysis:

- Detailed protocol analysis
- IDE integration patterns
- Comparison with other bridge architectures
- Diagrams and visualizations

**Remember**: No actual source code in contributions - only educational analysis.