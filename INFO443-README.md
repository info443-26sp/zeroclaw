# Proposal

## What is ZeroClaw?

ZeroClaw (originally written by Argenis De La Rosa along with the Sundai community) is a more optimized version of OpenClaw, using Rust as the primary language to make the runtime highly efficient for autonomous AI agents. Due to it being written in Rust, the benefits include it being memory efficient, modular, and secure.

## Context and Background

From a user perspective, ZeroClaw is an AI agent runtime that runs entirely on your own machine. You configure it with a single TOML file, point it at an LLM provider with your own API key, and it operates across messaging platforms (Discord, Telegram, email, etc.), executes shell commands and tools, and manages its own memory. Everything is local including your keys, your conversation history, your config. There is no cloud backend, no telemetry, and no license server. If you turn off the computer, the agent stops because it is running on your machine. It is designed to be a local runtime for autonomous AI agents, giving users full control over their data and interactions.

The project was originally created by Argenis De La Rosa (GitHub: @theonlyhennygod) along with the Sundai community. It is a Rust reimplementation of the OpenClaw concept by Peter Steinburg, optimized for performance, memory efficiency, and minimal binary size. ZeroClaw is an open-source community project. The project lead is Jordan (@JordanTheJet), and the core maintainers are @WareWolf-MoonWall (governance and docs) and @singlerider (runtime and providers). Significant changes to the project go through an RFC process requiring a two-thirds majority vote from the maintainers, so there is structured governance around architectural decisions.

Using the following links below are provided to learn more about Zeroclaw:

# The GitHub repository:
https://github.com/zeroclaw-labs/zeroclaw 

# The official documentation:
https://github.com/zeroclaw-labs/zeroclaw/tree/master/docs

