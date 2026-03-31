# Context Management Analysis

> **Educational Analysis Only** - No source code included.

---

## Overview

Context management is critical for long-running agent sessions. Claude Code must efficiently manage the limited context window while preserving important information across conversations.

---

## Extend Analysis

> Community-contributed extended analyses are stored in [analysis/](./analysis/) directory.

**Quick Link**: [analysis/](./analysis/) - View all extended analyses for this module

| PR | Title | Author | Description |
|----|-------|--------|-------------|
| - | - | - | *Waiting for contributions* |

---

## Architecture

### Context Sources

| Source | Type | Priority |
|--------|------|----------|
| System Prompt | Static | Highest |
| Tool Definitions | Static | High |
| Conversation History | Dynamic | Medium |
| File Contents | Dynamic | Medium |
| Memory | Persistent | Low |

### Context Lifecycle

```
┌─────────────────────────────────────────────────────┐
│                   Context Window                    │
├─────────────────────────────────────────────────────┤
│  System Prompt │ Tools │ Conversation │ Files │ ... │
└─────────────────────────────────────────────────────┘
        │
        ▼ (approaches limit)
┌─────────────────────────────────────────────────────┐
│              Compression Triggered                   │
│  - Summarize old messages                           │
│  - Remove redundant content                         │
│  - Preserve critical information                    │
└─────────────────────────────────────────────────────┘
```

---

## Key Components

### Token Estimation (`services/tokenEstimation.ts`)

| Function | Purpose |
|----------|---------|
| Count estimation | Approximate token usage |
| Budget tracking | Monitor context budget |
| Overflow prediction | Anticipate compression needs |

### Context Compression (`services/compact/`)

| Component | Purpose |
|-----------|---------|
| Summarizer | Compress old messages into summaries |
| Selector | Choose what to compress |
| Formatter | Structure compressed output |

### Memory Persistence (`memdir/`)

| Component | Purpose |
|-----------|---------|
| Memory storage | Persist long-term memories |
| Memory retrieval | Load relevant memories |
| Memory sync | Share memories across sessions |

---

## Compression Strategies

### Priority-Based Retention

Content is prioritized for retention:

| Priority | Content Type | Retention Strategy |
|----------|--------------|-------------------|
| Critical | System prompt, tool definitions | Never remove |
| High | Recent conversation, active files | Keep until necessary |
| Medium | Older conversation | Summarize |
| Low | Old summaries | Discard |

### Summarization Approach

When compression is needed:

1. **Identify candidates** - Find old or redundant content
2. **Summarize** - Create concise summary preserving key facts
3. **Replace** - Swap original content with summary
4. **Verify** - Ensure context remains coherent

---

## Memory System

### Memory Types

| Type | Purpose | Lifetime |
|------|---------|----------|
| Session Memory | Current conversation | Session duration |
| Persistent Memory | Long-term knowledge | Across sessions |
| Team Memory | Shared knowledge | Team scope |

### Memory Extraction (`services/extractMemories/`)

Automatic memory creation:

| Step | Action |
|------|--------|
| Detection | Identify memorable information |
| Extraction | Pull relevant facts |
| Storage | Save to memory store |

### Memory Retrieval

Memories are loaded based on:

- Relevance to current task
- Recency of access
- User-defined priority

---

## Context Window Management

### Budget Allocation

| Component | Typical Allocation |
|-----------|-------------------|
| System Prompt | ~5% |
| Tool Definitions | ~10% |
| Conversation History | ~50% |
| File Contents | ~30% |
| Memory | ~5% |

### Dynamic Adjustment

Allocation adjusts based on:
- Task type (coding vs. analysis)
- File sizes involved
- Conversation length

---

## Key Design Patterns

### 1. Incremental Compression

Don't wait until full:
- Compress incrementally
- Maintain buffer space
- Smooth user experience

### 2. Semantic Preservation

When summarizing:
- Preserve decisions and rationale
- Keep file references
- Maintain conversation flow

### 3. Lazy Loading

Load content on demand:
- Files loaded when referenced
- Memories retrieved when relevant
- Tools discovered when needed

---

## Lessons for Builders

1. **Proactive compression** prevents sudden context loss
2. **Priority systems** ensure important content survives
3. **Memory persistence** enables long-term learning
4. **Budget tracking** enables predictable behavior

---

## Contributing

We welcome contributions to this analysis:

- Detailed compression algorithm analysis
- Memory extraction patterns
- Comparison with other context management approaches
- Diagrams and visualizations

**Remember**: No actual source code in contributions - only educational analysis.