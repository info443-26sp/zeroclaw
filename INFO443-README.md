# INFO 443: Software Architecture Project 2

## What is ZeroClaw?

ZeroClaw (originally written by Argenis De La Rosa along with the Sundai community) is a more optimized version of OpenClaw, using Rust as the primary language to make the runtime highly efficient for autonomous AI agents. Due to it being written in Rust, the benefits include it being memory efficient, modular, and secure.

## Context and Background

From a user perspective, ZeroClaw is an AI agent runtime that runs entirely on your own machine. You configure it with a single TOML file, point it at an LLM provider with your own API key, and it operates across messaging platforms (Discord, Telegram, email, etc.), executes shell commands and tools, and manages its own memory. Everything is local including your keys, your conversation history, your config. There is no cloud backend, no telemetry, and no license server. If you turn off the computer, the agent stops because it is running on your machine. It is designed to be a local runtime for autonomous AI agents, giving users full control over their data and interactions.

The project was originally created by Argenis De La Rosa (GitHub: @theonlyhennygod) along with the Sundai community. It is a Rust reimplementation of the OpenClaw concept by Peter Steinburg, optimized for performance, memory efficiency, and minimal binary size. ZeroClaw is an open-source community project. The project lead is Jordan (@JordanTheJet), and the core maintainers are @WareWolf-MoonWall (governance and docs) and @singlerider (runtime and providers). Significant changes to the project go through an RFC process requiring a two-thirds majority vote from the maintainers, so there is structured governance around architectural decisions.

Using the following links below are provided to learn more about Zeroclaw:

## The GitHub repository:
https://github.com/zeroclaw-labs/zeroclaw 

## The official documentation:
https://github.com/zeroclaw-labs/zeroclaw/tree/master/docs

## Development View

This section describes ZeroClaw's architecture from a software development perspective. We identify the modules that make up the system, how they are organized and depend on one another, and how the code is structured, tested, built, and configured.

> **Rust terminology note:** A *crate* is a Rust package of code (like a library in other languages). A *workspace* is a collection of related crates developed together in one repository. *Cargo* is Rust's build tool and package manager (similar to `npm` for Node.js or `pip` for Python). *TOML* is a configuration file format (similar to YAML or JSON). *Feature flags* are compile-time switches that include or exclude optional functionality.

### Module Organization

ZeroClaw has 22 workspace members. The 17 main application crates live under the `crates/` directory and are organized into four layers:

1. **Foundation**: the core building blocks that everything else depends on.
2. **Support**: shared services (logging, configuration, utilities) used across the system.
3. **Edge**: components that talk to the outside world (AI providers, databases, external tools).
4. **Application**: the main programs that wire everything together.

#### Crate Overview

| Crate | Layer | Purpose |
|-------|-------|---------|
| `zeroclaw-api` | Foundation | Defines the plug-in interfaces that all extensions must follow |
| `zeroclaw-macros` | Foundation | Auto-generates repetitive boilerplate code |
| `zeroclaw-log` | Support | Handles all logging across the entire system |
| `zeroclaw-config` | Support | Reads and validates user settings from config files |
| `zeroclaw-infra` | Support | Shared utilities (timers, session storage, stall detection) |
| `zeroclaw-providers` | Edge | Connects to AI model providers (Anthropic, OpenAI, Ollama, etc.) |
| `zeroclaw-memory` | Edge | Stores conversation history and retrieves relevant past context |
| `zeroclaw-tools` | Edge | Tools the agent can invoke (browser, shell, file I/O, web search) |
| `zeroclaw-tool-call-parser` | Edge | Parses AI model responses into tool invocations |
| `zeroclaw-runtime` | Application | The main agent loop — the brain of the system |
| `zeroclaw-channels` | Application | Connects to messaging platforms (Discord, Telegram, Slack, email, etc.) |
| `zeroclaw-gateway` | Application | HTTP and WebSocket server for external API access |
| `zeroclaw-tui` | Application | Terminal-based setup wizard |
| `zeroclaw-plugins` | Application | Loads external WASM plugins dynamically |
| `zeroclaw-hardware` | Application | Controls hardware peripherals (GPIO, I2C, SPI, USB) |
| `aardvark-sys` | Application | Low-level bindings for a hardware adapter board |
| `robot-kit` | Application | Robot control logic for Nucleo-based hardware |

#### Layer Diagram

To visualize how these components relate, **Figure 1** maps every crate to its layer and shows the dependency flow between them. The diagram is the primary reference for understanding ZeroClaw's module structure.

![Figure 1: ZeroClaw Module Layer Diagram](/images/ZeroClaw_System_Architecture_Diagram.svg)

*Figure 1: ZeroClaw Module Layer Diagram. Each box is a crate grouped into one of four layers (Application, Edge, Support, Foundation). Solid arrows indicate required dependencies (A → B means "A depends on B").*

**Dependency rules:**

1. **Layers flow one direction.** Foundation crates never depend on Support, Edge, or Application crates. Support crates never depend on Edge or Application crates. This keeps the architecture clean and prevents circular dependencies.
2. **One universal plug-in interface.** The `zeroclaw-api` crate sits at the very bottom and defines the contracts (traits) that every extension implements. Almost every other crate depends on it.
3. **No circular dependencies allowed.** The build tool enforces this automatically — if a circular dependency were introduced, the code would not compile.

**Stability tiers:**

| Tier | Meaning | Crates |
|------|---------|--------|
| Beta | Safe to use; breaking changes are documented in release notes | `zeroclaw-config`, `zeroclaw-log`, `zeroclaw-infra`, `zeroclaw-providers`, `zeroclaw-memory`, `zeroclaw-macros`, `zeroclaw-tool-call-parser` |
| Experimental | May change without notice; not yet stable | `zeroclaw-runtime`, `zeroclaw-channels`, `zeroclaw-gateway`, `zeroclaw-tools`, `zeroclaw-tui`, `zeroclaw-plugins`, `zeroclaw-hardware`, `zeroclaw-api` |

### Codeline Organization

#### Directory Structure

Each crate lives in its own folder under `crates/`, so the source code structure mirrors the component structure exactly:

```
zeroclaw/
├── Cargo.toml              # Workspace definition (lists all crates)
├── src/                    # Main program entry point
├── crates/                 # All component crates
│   ├── zeroclaw-api/       # Plug-in interface definitions
│   ├── zeroclaw-config/    # User settings handling
│   ├── zeroclaw-log/       # Logging system
│   ├── zeroclaw-infra/     # Shared utilities
│   ├── zeroclaw-macros/    # Auto-generated code
│   ├── zeroclaw-providers/ # AI model connections
│   ├── zeroclaw-memory/    # Conversation storage
│   ├── zeroclaw-tools/     # Agent-callable tools
│   ├── zeroclaw-tool-call-parser/ # Tool call parsing
│   ├── zeroclaw-channels/  # Messaging integrations
│   ├── zeroclaw-runtime/   # Core agent logic
│   ├── zeroclaw-gateway/   # HTTP/WebSocket server
│   ├── zeroclaw-tui/       # Terminal setup wizard
│   ├── zeroclaw-plugins/   # WASM plugin loader
│   ├── zeroclaw-hardware/  # Hardware control
│   ├── aardvark-sys/       # Hardware adapter bindings
│   └── robot-kit/          # Robot control
├── tests/                  # Test suites
├── docs/                   # User documentation
└── .github/workflows/      # Automated build and test configuration
```

## Applied Perspective

This section discusses ZeroClaw's architecture from an evolution perspective, which analyzes the ability of ZeroClaw's architecture to withstand internal and external pressures. While ZeroClaw itself is an evolution from OpenClaw, in an AI landscape that is constantly evolving, ZeroClaw must continue to adapt to the latest trends to ensure that it stays viable. We will primarily focus on the modifiability of ZeroClaw's architecture and examine how it is designed to accommodate major changes within its environment. 