# Tool System Analysis

> **Educational Analysis Only** - No source code included.

---

## Overview

The tool system is the core abstraction that enables Claude Code to interact with the world. Every action the agent takes - reading files, running commands, searching the web - is mediated through tools.

---

## Extend Analysis

> Community-contributed extended analyses are stored in [analysis/](./analysis/) directory.

**Quick Link**: [analysis/](./analysis/) - View all extended analyses for this module

| PR | Title | Author | Description |
|----|-------|--------|-------------|
| - | - | - | *Waiting for contributions* |

---

## Architecture

### Tool Interface Pattern

Each tool follows a consistent interface:

| Component | Purpose |
|-----------|---------|
| Input Schema | Zod-based type validation for parameters |
| Permission Model | Determines when user approval is needed |
| Execution Logic | Performs the actual operation |
| Result Formatter | Structures output for LLM consumption |

### Tool Registration

Tools are registered dynamically, enabling:
- Deferred discovery and lazy loading
- Runtime tool availability changes
- Conditional tool exposure based on context

---

## Tool Categories

### File Operations

| Tool | Purpose | Educational Insight |
|------|---------|---------------------|
| `FileReadTool` | Read file contents | Handles multiple formats (text, images, PDFs, notebooks) |
| `FileWriteTool` | Create/overwrite files | Demonstrates safe write patterns with permission checks |
| `FileEditTool` | Partial file modification | String replacement approach for minimal diffs |
| `GlobTool` | Pattern-based file search | Efficient glob matching for file discovery |
| `GrepTool` | Content search | Uses ripgrep for performance |

### Web Access

| Tool | Purpose | Educational Insight |
|------|---------|---------------------|
| `WebFetchTool` | Fetch URL content | Handles various content types and encoding |
| `WebSearchTool` | Web search | Integrates with search APIs |

### Agent Orchestration

| Tool | Purpose | Educational Insight |
|------|---------|---------------------|
| `AgentTool` | Spawn sub-agents | Context isolation, task delegation, result aggregation |
| `SkillTool` | Execute skills | Bundled tool calls for domain workflows |

### Task Management

| Tool | Purpose | Educational Insight |
|------|---------|---------------------|
| `TaskCreateTool` | Create tasks | Dependency-based task scheduling |
| `TaskUpdateTool` | Update task status | State machine for task lifecycle |
| `TaskListTool` | List tasks | Query interface for task discovery |

### External Integration

| Tool | Purpose | Educational Insight |
|------|---------|---------------------|
| `MCPTool` | Invoke MCP server tools | Standardized protocol for external tools |
| `LSPTool` | Language Server Protocol | Code intelligence integration |

### Session Control

| Tool | Purpose | Educational Insight |
|------|---------|---------------------|
| `EnterPlanModeTool` / `ExitPlanModeTool` | Toggle plan mode | User approval workflow |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git worktree isolation | Safe parallel development |

---

## Key Design Patterns

### 1. Permission-First Design

Every tool defines its permission requirements:
- **Auto-allowed** - Safe operations (reading, searching)
- **Prompt-required** - Destructive operations (writing, executing)
- **Mode-dependent** - Varies by session mode

### 2. Schema-Driven Validation

Input validation using Zod provides:
- Type safety at runtime
- Clear error messages for invalid inputs
- Self-documenting parameter requirements

### 3. Result Streaming

Tools can stream results back:
- Progress indicators for long operations
- Partial results for large outputs
- Cancellation support

---

## Lessons for Builders

1. **Consistent interfaces** make tools composable and testable
2. **Permission models** should be declarative, not hardcoded
3. **Schema validation** catches errors early
4. **Streaming results** improve user experience for long operations

---

## Contributing

We welcome contributions to this analysis:

- Detailed analysis of specific tools
- Design pattern explanations
- Comparison with other agent frameworks
- Diagrams and visualizations

**Remember**: No actual source code in contributions - only educational analysis.