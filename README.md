# CC4CC - A collection about all you need for Claude Code source code analysis. 

> **Educational Analysis Only** - This repository does not contain any source code of the exposed artifacts and only contains analysis for educational purposes. No actual code is included in this documentation.

**English | [中文](./README_CN.md)**

---

## Extended Analysis


| Title | Author | Links |Description |
|----|-------|--|-------------|
  | How Claude Code Works | Windy3f |[link]((https://github.com/Windy3f3f3f3f/how-claude-code-works))| |
| i read claude code's entire source code. here's what anthropic got wrong.  | Rohan | [link](https://x.com/rohan_2502/status/2038927786228998194) |
| Claude Code v2.1.88 源码逆向分析 | 秋实 | [link](http://xhslink.com/o/8YaWaT3ZfDr)|  |
| - | - | - | *Waiting for contributions* |
---

## Learn Directory Structure

This directory contains detailed analysis of each critical module. Each subdirectory focuses on a specific architectural component:

```
learn/
├── README.md                 # This file - overview and navigation
├── README_CN.md              # Chinese version
├── analysis/                 # Extended analysis index (all modules)
│
├── tool-system/              # Tool system architecture analysis
│   ├── README.md
│   └── analysis/             # Extended analysis for this module
│
├── command-system/           # Slash command system analysis
│   ├── README.md
│   └── analysis/
│
├── service-layer/            # External service integrations
│   ├── README.md
│   └── analysis/
│
├── bridge-system/            # IDE integration bridge
│   ├── README.md
│   └── analysis/
│
├── permission-system/        # Permission and safety model
│   ├── README.md
│   └── analysis/
│
├── context-management/       # Context window and memory management
│   ├── README.md
│   └── analysis/
│
├── multi-agent/              # Multi-agent coordination
│   ├── README.md
│   └── analysis/
│
├── plugin-system/            # Plugin extensibility
│   ├── README.md
│   └── analysis/
│
├── skill-system/             # Skill and workflow system
│   ├── README.md
│   └── analysis/
│
└── ui-architecture/          # Terminal UI architecture
    ├── README.md
    └── analysis/
```

---

## Module Overview

### [Context Management](./context-management/)
Token budget management, compression, and memory persistence.

#### Extended Analysis

| Title | Author | Links |Description |
|----|-------|--|-------------|
|  How Claude Code Manage and Compact the Contexts. | Yifan | [link](./context-management/analysis/compact.md) |


### [Tool System](./tool-system/)
The core abstraction for all agent actions. Each tool is a self-contained module with input schema, permission model, and execution logic.
- **Key insight**: Dynamic registration enables deferred discovery and lazy loading

### [Command System](./command-system/)
User-facing slash commands for controlling Claude Code.
- **Key insight**: Centralized registry with schema-based argument parsing

### [Service Layer](./service-layer/)
External integrations including API, MCP, OAuth, and LSP services.
- **Key insight**: Abstraction layers isolate external dependencies

### [Bridge System](./bridge-system/)
Bidirectional communication between CLI and IDE extensions.
- **Key insight**: Protocol-first design enables multiple IDE clients

### [Permission System](./permission-system/)
Safety layer controlling when actions require user approval.
- **Key insight**: Declarative permissions with semantic matching
- 
### [Multi-Agent](./multi-agent/)
Sub-agent spawning, team management, and inter-agent communication.
- **Key insight**: Context isolation enables safe parallel work

### [Plugin System](./plugin-system/)
Extensibility framework for adding tools, commands, and hooks.
- **Key insight**: Capability declaration enables safe plugin loading

### [Skill System](./skill-system/)
Bundled workflows for domain-specific functionality.
- **Key insight**: Good prompts are the heart of effective skills

### [UI Architecture](./ui-architecture/)
Terminal UI built with Ink (React for terminals).
- **Key insight**: Component architecture scales well for complex UIs

---

## Architecture Summary

### High-Level Architecture

The system follows a **hub-and-spoke architecture** with:

- A central query engine that orchestrates LLM interactions
- Modular tool system for extensibility
- Command system for user-facing features
- Service layer for external integrations
- Bridge system for IDE connectivity

### Key Design Patterns

| Pattern | Application |
|---------|-------------|
| Tool abstraction | Every action is a self-contained tool |
| Permission-first | Safety built into the core |
| Context management | Critical for long-running sessions |
| Event-driven | Enables extensibility via hooks |
| Protocol abstraction | IDE-agnostic bridge design |

---

## Key Takeaways for Builders

1. **Modular tool design** - Each tool is self-contained with clear interfaces
2. **Permission-first architecture** - Safety considerations built into the core
3. **Context management** - Critical for long-running sessions
4. **Extensibility** - Plugin and skill systems allow customization
5. **IDE integration** - Bridge pattern enables rich integrations
6. **Multi-agent patterns** - Sub-agent spawning for complex tasks

---

## Research / Ownership Disclaimer

- This repository is an **educational and defensive security research archive**.
- It exists to study source exposure, packaging failures, and the architecture of modern agentic CLI systems.
- The original Claude Code source remains the property of **Anthropic**.
- This repository is **not affiliated with, endorsed by, or maintained by Anthropic**.
- **No actual source code is included** - only high-level architectural analysis and educational observations.

---

## Contributing

**We welcome all kinds of pull requests for learning purposes!**

### How to Contribute

Each module has its own subdirectory under `learn/`. We encourage contributions to the corresponding subdirectory:

| Module | Directory | What to Contribute |
|--------|-----------|-------------------|
| Tool System | [tool-system/](./tool-system/) | Tool patterns, specific tool analysis |
| Command System | [command-system/](./command-system/) | Command patterns, argument parsing |
| Service Layer | [service-layer/](./service-layer/) | API patterns, integration analysis |
| Bridge System | [bridge-system/](./bridge-system/) | Protocol analysis, IDE integration |
| Permission System | [permission-system/](./permission-system/) | Security model, approval flows |
| Context Management | [context-management/](./context-management/) | Compression algorithms, memory patterns |
| Multi-Agent | [multi-agent/](./multi-agent/) | Agent spawning, communication patterns |
| Plugin System | [plugin-system/](./plugin-system/) | Extension patterns, hook system |
| Skill System | [skill-system/](./skill-system/) | Prompt engineering, workflow design |
| UI Architecture | [ui-architecture/](./ui-architecture/) | Component patterns, state management |

### Types of Contributions Welcome

- **Detailed analysis** of specific components or patterns
- **Diagrams and visualizations** of architecture
- **Design pattern explanations** with examples
- **Comparison with other frameworks** (LangChain, AutoGPT, etc.)
- **Educational tutorials** based on the architecture
- **Translations** of documentation
- **Code flow diagrams** (without actual code)

### Extended Analysis Index

Each module has an `analysis/` subdirectory for community-contributed deep dives. The main [analysis/](./analysis/) directory provides a cross-module index.

**How to submit extended analysis:**

1. Create your analysis file in the appropriate module's `analysis/` directory
   - Naming: `PR{number}_{short-title}.md`
   - Example: `PR42_permission-flow.md`
2. Update the index table in that module's `analysis/README.md`
3. Optionally update the main [analysis/README.md](./analysis/README.md) for cross-module visibility

**Quick links to module analysis directories:**

| Module | Analysis Directory |
|--------|-------------------|
| Tool System | [analysis/](./tool-system/analysis/) |
| Command System | [analysis/](./command-system/analysis/) |
| Service Layer | [analysis/](./service-layer/analysis/) |
| Bridge System | [analysis/](./bridge-system/analysis/) |
| Permission System | [analysis/](./permission-system/analysis/) |
| Context Management | [analysis/](./context-management/analysis/) |
| Multi-Agent | [analysis/](./multi-agent/analysis/) |
| Plugin System | [analysis/](./plugin-system/analysis/) |
| Skill System | [analysis/](./skill-system/analysis/) |
| UI Architecture | [analysis/](./ui-architecture/analysis/) |

### Contribution Guidelines

Please ensure all contributions:

1. **Contain no actual source code** - Only high-level descriptions
2. **Focus on educational value** - Help others learn
3. **Respect intellectual property** - This is analysis, not reproduction
4. **Link to related modules** - Show connections between components

### Getting Started

1. Choose a module directory that interests you
2. Read the existing README.md in that directory
3. Identify gaps or areas for deeper analysis
4. Submit a PR with your educational contribution

---

## License

This analysis is provided for educational purposes only. The original Claude Code software is property of Anthropic.
