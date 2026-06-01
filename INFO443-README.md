# INFO 443: Software Architecture Project 2

## What is ZeroClaw?

ZeroClaw (originally written by Argenis De La Rosa along with the Sundai community) is a more optimized version of OpenClaw, using Rust as the primary language to make the runtime highly efficient for autonomous AI agents. Due to it being written in Rust, the benefits include it being memory efficient, modular, and secure.

## Context and Background

From a user perspective, ZeroClaw is an AI agent runtime that runs entirely on your own machine. You configure it with a single TOML file, point it at an LLM provider with your own API key, and it operates across messaging platforms (Discord, Telegram, email, etc.), executes shell commands and tools, and manages its own memory. Everything is local including your keys, your conversation history, your config. There is no cloud backend, no telemetry, and no license server. If you turn off the computer, the agent stops because it is running on your machine. It is designed to be a local runtime for autonomous AI agents, giving users full control over their data and interactions.

The project was originally created by Argenis De La Rosa (GitHub: @theonlyhennygod) along with the Sundai community. It is a Rust reimplementation of the OpenClaw concept by Peter Steinburg, optimized for performance, memory efficiency, and minimal binary size. ZeroClaw is an open-source community project. The project lead is Jordan (@JordanTheJet), and the core maintainers are @WareWolf-MoonWall (governance and docs) and @singlerider (runtime and providers). Significant changes to the project go through an RFC process requiring a two-thirds majority vote from the maintainers, so there is structured governance around architectural decisions.

The following links provide more information about ZeroClaw:

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

This section discusses ZeroClaw's architecture from an evolution perspective, which analyzes the ability of ZeroClaw's architecture to withstand internal and external pressures. While ZeroClaw itself is an evolution from OpenClaw, in an AI landscape that is constantly evolving, ZeroClaw must continue to adapt to the latest trends to ensure that it stays viable. We will primarily focus on the modifiability and reliability of change within ZeroClaw's architecture and examine how it is designed to accommodate major changes in its environment.

### Concerns

1. **Dimensions of Change:** Integration evolution is prominant within ZeroClaw's architecture since it uses a large variety of different channels, tools, providers, and other key systems. As new models and tools are released, these integrations will be expected to evolve alongside these constant changes, making integration evolution a key concern for ZeroClaw's architecture. 

2. **Changes Due to External Factors:** ZeroClaw is subject to constant external pressures such as LLMs or APIs becoming deprecated, changes being made to APIs, and new platforms emerging, which makes adapting to these external pressures a key concern for ZeroClaw staying viable. 

3. **Reliability of Change:** ZeroClaw's reliance on a small team and an open-source community can lead to concerns with the reliability and consistency made to the system over time. As the project grows and more contributors get involved, maintaining a consistent process for these changes is crucial. 

## Styles & Patterns

### Architectural Style

ZeroClaw uses two architectural styles: **layered architecture** and **microkernel architecture**. The layered style describes how the crates are organized relative to each other, while the microkernel style describes how the system is extended with new capabilities.

**Layered architecture** is visible in the four-layer structure described in the Development View (Foundation, Support, Edge, Application). Dependencies only flow downward — Application crates depend on Edge and Support, and nothing in Foundation depends on anything above it. Importantly, these boundaries are not just a convention: Rust's compiler enforces them. A circular or upward dependency would cause a compile failure, so no contributor can accidentally break the layering.

**Microkernel architecture** is visible in the role of `zeroclaw-api`. That crate acts as the microkernel — it is small, stable, and defines only the abstract interfaces that everything else plugs into: `ModelProvider`, `Channel`, `Tool`, `Memory`, `Observer`, and `Peripheral`. All actual functionality lives in separate plug-in crates that register themselves at startup through factory functions. The `AGENTS.md` file makes this explicit, listing the extension points by file and describing the architecture as "trait-driven and modular." The stability tier system reflects the same intent — `zeroclaw-api` is on a path to a stable v1.0.0 while the plug-in crates can evolve freely around it.

### Design Patterns

**Retry pattern** (`crates/zeroclaw-providers/src/reliable.rs`)

ZeroClaw uses the retry pattern in `ReliableModelProvider`, which wraps any LLM provider and automatically re-attempts failed API calls. When a call fails with a transient error, the wrapper sleeps with exponential backoff — doubling the wait on each attempt up to a 10-second cap — and retries up to a configurable limit. If a `Retry-After` value is present in the error, that is used instead. If all retries against one provider are exhausted, the wrapper falls through to the next provider in a priority chain, giving the system two levels of recovery: retrying within a provider, and then failing over across providers.

**Circuit breaker** (`crates/zeroclaw-providers/src/reliable.rs`)

Alongside the retry loop, `ReliableModelProvider` applies the circuit breaker pattern through its error-classification functions `is_non_retryable()` and `is_non_retryable_rate_limit()`. When a call fails with an error that retries cannot fix — such as an invalid API key (401), a missing model (404), or a business-quota rejection — the wrapper breaks out of the retry loop immediately without sleeping and without consuming the remaining retry budget. This prevents the system from wasting time and API quota on failures that will never resolve on their own.

**Publish-subscribe** (`crates/zeroclaw-api/src/observability_traits.rs`, `crates/zeroclaw-runtime/src/observability/multi.rs`)

ZeroClaw uses the publish-subscribe pattern for observability. The agent runtime publishes lifecycle events — `AgentStart`, `LlmRequest`, `LlmResponse`, `ToolCall`, `TurnComplete`, and others — through a single `Arc<dyn Observer>` interface without knowing which backends are listening. Subscribers are concrete `Observer` implementations: a structured log writer, a Prometheus metrics exporter, an OpenTelemetry trace emitter, and a verbose stdout observer. `MultiObserver` acts as the broker, holding a list of subscribers and fanning each published event out to all of them.

**Blackboard pattern** (`crates/zeroclaw-memory/`, `crates/zeroclaw-runtime/src/agent/memory_loader.rs`)

ZeroClaw uses the blackboard pattern through its `zeroclaw-memory` crate, which provides a shared knowledge store backed by Markdown files, SQLite, or vector embeddings depending on configuration. During a session, the runtime writes conversation turns and tool results to the memory backend; the `memory_loader` reads from it to inject relevant past context into each new LLM request. Neither component depends on the other directly — they both communicate through the shared store, keeping the agent loop and the memory system decoupled.

## Architectural Assessment

### Dependency Inversion Principle (DIP)

ZeroClaw adheres to the Dependency Inversion Principle through the `zeroclaw-api` crate, which sits at the bottom of the dependency graph (see **Figure 1**) and defines only abstract trait interfaces: `Provider`, `Channel`, `Tool`, `Memory`, `Observer`, and `Peripheral`. The agent runtime (`zeroclaw-runtime`) depends just on these traits. It never imports a concrete Anthropic client, a Discord bot library, or a specific search engine implementation. Concrete providers, channels, and tools implement the relevant traits and are registered at startup via factory functions, which the runtime queries through a registry rather than through hard-coded references.

This inverts the typical dependency direction: the high-level agent loop (policy) does not import low-level API clients (details). Instead, both depend on the abstract interfaces in `zeroclaw-api`. A practical consequence is that swapping from OpenAI to Ollama requires changing the configuration file, not any code in the runtime.

### Open-Closed Principle (OCP)

ZeroClaw adheres to the Open-Closed Principle across all four extension points defined in the API crate. Adding a new messaging platform requires creating a new struct that implements the `Channel` trait and registering it in the channel factory. The runtime, security subsystem, and all other existing code remain untouched, as they reference channels only through the trait interface. The same pattern applies to providers, tools, and memory backends.

This is enforced by the architecture boundary rules in AGENTS.md: extension crates must register via factory functions and must never modify runtime code. A contributor adding a new channel never needs to edit `zeroclaw-runtime`. The core is closed to modification while the extension surface is open. Feature flags further support this by letting users compile only the extensions they need without changing the system's logic.

### Encapsulation

ZeroClaw enforces encapsulation at three levels.

1. At the **crate boundary**, each crate exposes only a public API through its `lib.rs` file. Internal submodules, helper functions, and implementation details are kept private. For example, `zeroclaw-providers` exposes a factory function and the `Provider` trait implementation, but the individual HTTP client logic for Anthropic, OpenAI, and Ollama are internal modules that callers cannot import.

2. At the **trait boundary**, concrete types are never exposed to callers. The runtime holds `Arc<dyn Provider>` references, as in it never sees the concrete AnthropicClient struct. This means the internals of AnthropicClient (connection pooling, retry logic, token counting) can be refactored entirely without any change to the runtime.

3. At the **module level** within crates, submodules follow the same pattern. For instance, `zeroclaw-runtime/src/security/` contains policy enforcement, sandbox detection, and emergency stop logic, all of which are internal to the security subsystem and invisible to the rest of the runtime.