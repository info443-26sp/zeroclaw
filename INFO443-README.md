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

> **Rust terminology:** A *crate* is a Rust package of code (like a library in other languages). A *workspace* is a collection of related crates developed together in one repository. *Cargo* is Rust's build tool and package manager (similar to `npm` for Node.js or `pip` for Python). *TOML* is a configuration file format (similar to YAML or JSON). *Feature flags* are compile-time switches that include or exclude optional functionality.

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

![Figure 1: ZeroClaw Module Layer Diagram](/images/ZeroClaw_System_Architecture_Diagram_Updated.svg)

*Figure 1: ZeroClaw Module Layer Diagram. Each box is a crate grouped into one of four layers (Application, Edge, Support, Foundation). Solid arrows indicate required dependencies (A → B means "A depends on B"). Dashed arrows indicate optional dependencies compiled only when the corresponding feature is enabled.*

**Dependency rules:**

1. **Layers flow one direction.** Foundation crates never depend on Support, Edge, or Application crates. Support crates never depend on Edge or Application crates. This keeps the architecture clean and prevents circular dependencies.
2. **One universal plug-in interface.** The `zeroclaw-api` crate sits at the very bottom and defines the contracts (traits) that every extension implements. Almost every other crate depends on it.
3. **No circular dependencies allowed.** The build tool enforces this automatically — if a circular dependency were introduced, the code would not compile.

**Stability tiers:**

| Tier | Meaning | Crates |
|------|---------|--------|
| Beta | Safe to use; breaking changes are documented in release notes | `zeroclaw-config`, `zeroclaw-log`, `zeroclaw-infra`, `zeroclaw-providers`, `zeroclaw-memory`, `zeroclaw-macros`, `zeroclaw-tool-call-parser` |
| Experimental | May change without notice; not yet stable | `zeroclaw-runtime`, `zeroclaw-channels`, `zeroclaw-gateway`, `zeroclaw-tools`, `zeroclaw-tui`, `zeroclaw-plugins`, `zeroclaw-hardware`, `zeroclaw-api` |

#### External Dependencies

Several crates communicate with external systems outside the Rust codebase:

| Crate | External dependency | Purpose |
|-------|-------------------|---------|
| `zeroclaw-providers` | Anthropic, OpenAI, Ollama, Azure, Google, OpenRouter APIs | Sends prompts and receives LLM responses |
| `zeroclaw-channels` | Discord, Telegram, Slack, email (SMTP/IMAP) APIs | Sends and receives messages on each platform |
| `zeroclaw-memory` | SQLite, filesystem | Persists conversation history and embeddings |
| `zeroclaw-tools` | System shell, HTTP, filesystem | Executes commands, web searches, file operations |
| `zeroclaw-gateway` | HTTP, WebSocket | Exposes external REST API |
| `zeroclaw-plugins` | WASM runtime | Loads and executes plug-in binaries |
| `zeroclaw-hardware` | USB, I2C, SPI, GPIO | Communicates with peripheral hardware |
| `aardvark-sys` | USB, I2C (Aardvark adapter) | Low-level hardware bridge |
| `robot-kit` | Serial (Nucleo board) | Robot control commands |
| `zeroclaw-log`, `zeroclaw-config` | Filesystem | Reads config files, writes log output |

All external dependencies are runtime-only; the compilation process requires no internet access. Many of them are optional and compiled only when the corresponding Cargo feature flag is enabled.

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

#### Build System

The project uses Cargo with five build profiles, each optimized for a different scenario:

| Profile | When to use | What it optimizes for |
|---------|-------------|----------------------|
| `dev` | Daily development | Fast compile time |
| `release` | Shipping to users | Small file size, fast runtime |
| `release-fast` | Local test builds | Balance of size and speed |
| `ci` | Automated testing | Speed across parallel builds |
| `dist` | Distribution artifacts | Smallest possible file size |

**CI/CD pipeline.** Every push and pull request triggers a multi-stage pipeline in GitHub Actions:

1. **Lint.** Runs `cargo fmt --check` for code style and `cargo clippy` for common bugs and anti-patterns. Both must pass or the build fails.
2. **Build.** Compiles all crates across multiple profiles (dev, ci) and platforms (Windows, macOS, Linux). Feature combinations (default, all features, no features) are tested in parallel.
3. **Test.** Runs all unit, component, and integration tests with code coverage tracking. If any test fails, the pipeline stops.
4. **Audit.** Scans dependencies for known security vulnerabilities using `cargo audit`. Commits with vulnerable dependencies are rejected.
5. **Post-merge release.** When code lands on the master branch, the pipeline generates build artifacts (compiled binaries, package archives) and publishes them to the project's GitHub Releases page with auto-generated release notes.

### Testing

Testing is organized into five levels, each testing a different boundary:

| Level | What it tests | What's real vs. simulated |
|-------|--------------|--------------------------|
| Unit | A single function | Everything else is simulated |
| Component | One crate in isolation | Neighboring crates are simulated |
| Integration | Several crates working together | Only external services (APIs, databases) are simulated |
| System | Full application from start to finish | Only internet-facing services are simulated |
| Live | Full application with real services | Nothing is simulated — runs against real accounts and costs real money |

The project provides shared testing tools including mock AI providers that return scripted responses, mock tools with controlled behavior, and test channels that capture outgoing messages. Complex test scenarios are defined as declarative JSON trace fixtures that describe a conversation as a series of turns, where each turn contains LLM response steps and a set of expected outcomes, replacing inline mock setup with reusable scripts verified automatically. A dedicated `test_architecture.rs` suite uses static analysis to detect duplicate state across the codebase, failing the build if any is discovered, enforcing this invariant automatically rather than relying on developer discipline. All test levels except Live run automatically on every code change through the continuous integration (CI) system; Live tests require real API credentials and must be run manually.

### Configuration

All user settings live in a single TOML configuration file. This file controls which AI provider to use (Anthropic, OpenAI, Ollama, or others), which messaging platforms to connect to, security and autonomy settings, memory storage options, and tool permissions. API keys and other secrets are encrypted before being saved to disk. Every crate reads its configuration through a standardized system that validates all settings at startup. This ensures that if a setting is invalid or missing, the system reports the error immediately rather than failing later at an unpredictable point. 

The config file carries a schema version field, and when the system loads an older file it runs a migration step that transforms the file to match the current schema before validation, allowing the configuration format to evolve across releases without breaking existing user setups. Optional capabilities are gated behind Cargo feature flags at the workspace level. For example, hardware peripherals are compiled only when the `hardware` feature is enabled, and WASM plugin support is compiled only when the `plugins` feature is enabled. This keeps the default binary small while letting users add capabilities as needed.

## Applied Perspective

This section examines ZeroClaw's architecture from the Evolution perspective (Rozanski & Woods, Ch 28), which focuses on a system's ability to accommodate change after deployment while balancing the cost of providing that flexibility. For a runtime that operates in the rapidly shifting AI ecosystem where LLM providers deprecate APIs, new messaging platforms appear, and security threats evolve, evolvability is a critical architectural concern. We identify the dimensions of change most relevant to ZeroClaw, evaluate how the architecture handles them, and perform structured activities to assess the system's current ease of evolution.

### Concerns

1. **Dimensions of Change.** The Rozanski framework identifies four dimensions of change: functional (what the system does), platform (hardware and OS), integration (interaction with external systems), and growth (volume and scale). ZeroClaw's primary dimension is integration evolution. The system connects to dozens of external AI providers, messaging platforms, and hardware peripherals, each with its own API contract and release cycle. New LLM models are released frequently, and existing APIs are often deprecated or modified. On the other hand, platform evolution is minimal, where ZeroClaw runs on standard operating systems and has no hardware dependency beyond the optional peripherals crate. Growth is handled by the system's stateless, config-driven design. Each session is independent, so scaling to more users means running more instances rather than redesigning the architecture.

2. **Changes Due to External Factors.** ZeroClaw has no control over the APIs it depends on. 

    For example: 
    - An AI provider can change its response format, deprecate a model, or introduce rate limits at any time. 
    - A messaging platform can update its authentication requirements. 
    - New security vulnerabilities can emerge in dependencies. 
    
    Each of these external forces can break the system if changes are not contained. ZeroClaw addresses this through its trait-based plug-in architecture. When a provider API changes, only the crate implementing that particular provider needs to be updated. The runtime, all other providers, channels, and tools are unaffected. The dependency audit stage in CI uses `cargo audit` to catch vulnerable dependencies before they reach production.

3. **Reliability of Change.** An open-source project with a small team of maintainers face a reliability challenge, where many contributors touch the codebase, but few have deep knowledge of every subsystem. Changes must be safe and reviewable without requiring full-system expertise. ZeroClaw addresses this through multiple approaches. The layered architecture with compiler-enforced dependency rules ensures that a change in one crate cannot accidentally affect unrelated crates. The CI pipeline runs linting, building, testing, and security auditing on every pull request. Architecture invariant tests in `test_architecture.rs` catch violations of the no-duplicate-state rule automatically. Stability tiers communicate the expected breaking-change risk of each crate, letting contributors know which areas are safe to modify and which require more care.

### Applying the Perspective

**Activity 1: Characterize Evolution Needs.** We assess what parts of ZeroClaw are likely to change, how often, and with what impact.

| Change category | Examples | Frequency | Magnitude | Containing mechanism |
|---|---|---|---|---|
| New AI provider | Add support for a new LLM API | High (monthly) | Small (one new crate + config entry) | `FamilyProviderFactory` trait + macro dispatch |
| New messaging channel | Add Discord, Telegram, Slack | Medium (quarterly) | Small (one new crate + config entry) | `Channel` trait + factory registration |
| API change in provider | Anthropic modifies response format | Medium (quarterly) | Small to medium (update one crate) | Trait isolates change to a single crate |
| Core trait change | Modify the `ModelProvider` interface | Low (yearly) | Large (update all providers) | API is experimental; RFC process governs |
| New tool | Add web search, file browser | Medium (quarterly) | Small (one new crate) | `Tool` trait + factory registration |
| Dependency vulnerability | CVE found in a library | Ad-hoc | Variable | `cargo audit` in CI catches it |

The dominant pattern is many small, frequent changes that are well contained by the trait interfaces. Large changes to the core API traits are rare and governed by the RFC process, which requires a two-thirds maintainer vote.

**Activity 2: Assess Current Ease of Evolution.** We walk through a concrete change scenario to evaluate how well the architecture supports it.

*Scenario: A new LLM provider, "ExampleAI", releases an API. A developer needs to add support to ZeroClaw.*

What the developer must do:
- Add one configuration entry to the provider config schema (a single TOML struct definition).
- Implement `FamilyProviderFactory` for the new config type (one trait impl with construction logic mapping ExampleAI's SDK calls to the `ModelProvider` interface).
- Optionally enable a Cargo feature flag if the provider requires a new library dependency.

What the developer does NOT need to do:
- Modify the agent runtime (`zeroclaw-runtime`).
- Modify any existing provider crate or its construction logic.
- Modify channels, tools, memory backends, or the configuration loading system.
- Understand any code outside the providers crate.

The macro-powered `dispatch_family_factory` function picks up the new variant automatically at compile time. The change is fully additive. In a hypothetical monolithic design, this same scenario would require modifying a central dispatch function with a new conditional branch, updating configuration parsing, adding imports to the main binary, and carefully checking that the new provider does not break error handling in unrelated code paths. ZeroClaw's architecture keeps these changes from spreading.

### Problems and Pitfalls

The Evolution perspective warns against two common pitfalls. The first is supporting changes that never happen. ZeroClaw's trait hierarchy and factory dispatch macros add compile-time complexity and indirection. If AI providers stop changing so often, this flexibility will have added complexity for no reason. The second is the impact on other qualities. Trait objects make the code slower because the compiler cannot optimize function calls through them. For an agent runtime where response latency matters, this is a real trade-off. However, feature flags partially mitigate this by letting users who do not need hardware or WASM support compile a smaller, more optimized binary with less indirection.

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

**Adapter pattern** (`crates/zeroclaw-hardware/src/transport.rs`, `crates/zeroclaw-hardware/src/serial.rs`, `crates/zeroclaw-hardware/src/aardvark.rs`)

ZeroClaw uses the adapter pattern in the hardware subsystem to unify communication with devices over different transport protocols. Hardware tools must send commands to physical devices using different protocols: USB serial for microcontrollers, I2C/SPI for an Aardvark adapter board, and mock transports for testing. Each transport protocol has its own native interface, so wiring every tool to every transport would create combinatorial conditional logic in each tool. The `Transport` trait defines a common interface with a single `send(cmd) -> response` method. Three adapters implement this trait: `HardwareSerialTransport` wraps serial port read/write calls, `AardvarkTransport` wraps the Aardvark C library's I2C/SPI functions, and `MockTransport` provides a test double that returns scripted responses. Hardware tools depend only on `Arc<dyn Transport>` and are completely decoupled from the specific wire protocol, allowing new transports to be added without modifying any tool code.

## Architectural Assessment

### Dependency Inversion Principle (DIP)

ZeroClaw adheres to the Dependency Inversion Principle through the `zeroclaw-api` crate, which sits at the bottom of the dependency graph (see **Figure 1**) and defines only abstract trait interfaces: `Provider`, `Channel`, `Tool`, `Memory`, `Observer`, and `Peripheral`. The agent runtime (`zeroclaw-runtime`) depends just on these traits. It never imports a concrete Anthropic client, a Discord bot library, or a specific search engine implementation. Concrete providers, channels, and tools implement the relevant traits and are registered at startup via factory functions, which the runtime queries through a registry rather than through hard-coded references.

This inverts the typical dependency direction: the high-level agent loop (policy) does not import low-level API clients (details). Instead, both depend on the abstract interfaces in `zeroclaw-api`. A practical consequence is that swapping from OpenAI to Ollama requires changing the configuration file, not any code in the runtime.

### Open-Closed Principle (OCP)

ZeroClaw adheres to the Open-Closed Principle across all four extension points defined in the API crate. Adding a new messaging platform requires creating a new struct that implements the `Channel` trait and registering it in the channel factory. The runtime, security subsystem, and all other existing code remain untouched, as they reference channels only through the trait interface. The same pattern applies to providers, tools, and memory backends.

This is enforced by the architecture boundary rules in AGENTS.md: extension crates must register via factory functions and must never modify runtime code. A contributor adding a new channel never needs to edit `zeroclaw-runtime`. The core is closed to modification while the extension surface is open. Feature flags further support this by letting users compile only the extensions they need without changing the system's logic.

### Interface Segregation Principle (ISP)

ZeroClaw partly follows the Interface Segregation Principle through small traits, which are Rust's version of interfaces, like `Tool` and `Observer`. These traits are easy to work with because they only include the methods that each implementation needs.

ZeroClaw does this less well with larger traits like `Channel`, `Memory`, and `ModelProvider`. These traits include many optional features, so a simple channel may still be connected to methods for drafts, reactions, approvals, pins, redaction, typing indicators, and streaming, even if it only sends and receives messages. This design makes the runtime easier to manage because it can use one trait for many cases, but it also makes some interfaces larger than they need to be.

### Encapsulation

ZeroClaw enforces encapsulation at three levels.

1. At the **crate boundary**, each crate exposes only a public API through its `lib.rs` file. Internal submodules, helper functions, and implementation details are kept private. For example, `zeroclaw-providers` exposes a factory function and the `Provider` trait implementation, but the individual HTTP client logic for Anthropic, OpenAI, and Ollama are internal modules that callers cannot import.

2. At the **trait boundary**, concrete types are never exposed to callers. The runtime holds `Arc<dyn Provider>` references, as in it never sees the concrete AnthropicClient struct. This means the internals of AnthropicClient (connection pooling, retry logic, token counting) can be refactored entirely without any change to the runtime.

3. At the **module level** within crates, submodules follow the same pattern. For instance, `zeroclaw-runtime/src/security/` contains policy enforcement, sandbox detection, and emergency stop logic, all of which are internal to the security subsystem and invisible to the rest of the runtime.

### Liskov Substitution Principle (LSP)

ZeroClaw adheres to the Liskov Substitution Principle through its trait-based architecture. The agent runtime holds `Arc<dyn Provider>`, `Arc<dyn Memory>` and similar references for each trait, which allows for implementations to be substituted without any calling code being changed. 

In the `Provider` trait implementation, providers such as `AnthropicModelProvider` and `OpenAiModelProvider` all implement the same `chat()` interface and the agent calls `provider.chat()` regardless of which type is behind it. The agent also calls `ReliableModelProvider` identically to a bare provider, as the retry and fallback logic it wraps around another `Provider` is entirely invisible to the caller.

The LSP is also present within the `Memory` trait implementation, where `PostgresMemory`, `MarkdownMemory` and `SqliteMemory` are all valid substitutions at the same call sites in the agent loop. Even in the most extreme case with `NoneMemory` (which returns empty results), the substitution is still valid without any hidden dependencies on specific implementation interfering with the calling code.  

### Single Responsibility Principle (SRP)

ZeroClaw adheres to the Single Responsibility Principle at both the crate and module level. Each crate has one clearly scoped job: `zeroclaw-log` handles logging, `zeroclaw-config` handles reading and validating user settings, and `zeroclaw-memory` handles conversation storage. A change to how logs are written never touches configuration parsing, and a change to the memory backend never touches the agent loop.

The clearest crate-level expression of SRP is the separation between `zeroclaw-tool-call-parser` and `zeroclaw-tools`. Parsing an LLM response into structured tool calls is one responsibility; executing those tools is another. The parser crate's own documentation states that it has "no dependency on agent state, memory, model_providers, or channels" and is "pure text transformation." If the LLM response format changes — for example, a new provider uses a different tool call envelope — only the parser crate needs to change. The tool implementations in `zeroclaw-tools` are entirely unaffected.

Within crates, the same principle applies at the module level. `zeroclaw-config` separates its concerns into distinct files: `migration.rs` handles schema version upgrades, `secrets.rs` handles encryption and decryption of API keys, `validation_warnings.rs` reports invalid settings, and `schema.rs` defines the configuration structure. A change to how secrets are encrypted touches only `secrets.rs`. Similarly, `zeroclaw-tools` gives each tool its own file — `web_search_tool.rs`, `file_edit.rs`, `browser.rs`, and so on — so a change to how web search works never risks affecting file editing or browser control.

## References

- Fowler, M. (2018, February 26). The Practical Test Pyramid. martinFowler.com. https://martinfowler.com/articles/practical-test-pyramid.html
- Fowler, M. (2018, January 16). Integration test. martinFowler.com. https://martinfowler.com/bliki/IntegrationTest.html
- Fowler, M. (2018, November). Refactoring: Improving the design of existing code (2nd ed.). Addison-Wesley. O'Reilly Media. https://learning.oreilly.com/library/view/refactoring-improving-the/9780134757681/
- Freeman, E., & Robson, E. (2020, December). Head first design patterns (2nd ed.). O'Reilly Media. https://learning.oreilly.com/library/view/head-first-design/9781492077992/
- Martin, R. C. (2008, August). Clean code: A handbook of agile software craftsmanship. Pearson. O'Reilly Media. https://www.oreilly.com/library/view/clean-code-a/9780136083238/
- Martin, R. C. (2017, September). Clean architecture: A craftsman's guide to software structure and design. Pearson. O'Reilly Media. https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/
- Molyneaux, I. (2014, December). The art of application performance testing: From variables to results (2nd ed.). O'Reilly Media. https://learning.oreilly.com/library/view/the-art-of/9781491900536/
- Richards, M. (2022, July). Software Architecture Patterns, 2nd Edition. O'Reilly Media. https://learning.oreilly.com/library/view/software-architecture-patterns/9781098134280/
- Rozanski, N., & Woods, E. (2011, October). Software systems architecture: Working with stakeholders using viewpoints and perspectives (2nd ed.). Addison-Wesley. O'Reilly Media. https://www.oreilly.com/library/view/software-systems-architecture/9780132906135/
- Rust Project Developers. (2026, March 25). std. Rust Programming Language. https://doc.rust-lang.org/std/
- Shalloway, A., & Trott, J. R. (2001, July). Design patterns explained: A new perspective on object-oriented design (2nd ed.). Addison-Wesley. O'Reilly Media. https://learning.oreilly.com/library/view/design-patterns-explained/0201715945/
- Wikipedia contributors. (2026, March 23). Service-oriented architecture. Wikipedia, The Free Encyclopedia. https://en.wikipedia.org/wiki/Service-oriented_architecture

## AI Disclosure

AI tools were used for proofreading, grammar checking, and consistency review of this report. Relevant prompts to Anthropic Claude included:

- "Review INFO443-README.md and check for grammar, typos, and self-consistency."
- "Simplify [specific sentences] for clarity."
- "Suggest alternative wording for [passages/phrasing]."
