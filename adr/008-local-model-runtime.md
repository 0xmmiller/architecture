# ADR-008: Local models are a first-class agent runtime

- Status: Accepted
- Date: 2026-09-18
- Deciders: Mark Miller

## Problem

Agent backends that only speak a hosted API are demos. Atlas already
exposes portfolio and intents as tools. The model that calls those
tools should be swappable: laptop (Ollama / llama.cpp), cluster
(vLLM), or a vendor — same OpenAI-compatible wire.

## Options considered

### 1. Vendor SDK only (OpenAI/Anthropic)

Fastest demo. Locks evals, CI, and air-gapped work to a bill. Wrong
lesson for a lab that already cares about failure domains.

### 2. Custom llama.cpp bindings in-process

Maximum control. Couples the API process to GPU drivers and model
files. Restarts become 10GB events.

### 3. OpenAI-compatible HTTP to a local server

Ollama, vLLM, llama.cpp `llama-server` all speak `/v1/chat/completions`.
The agent process stays a backend. The model process can die without
taking Postgres with it. Tests use `FakeLLM`; laptops use Ollama.

## Decision

**AgentKit talks OpenAI-compat HTTP. Default base URL is local.**

`OLLAMA_BASE_URL=http://127.0.0.1:11434/v1` (or vLLM on 8000).
Cloud is an env change, not an architecture change.

Tools Atlas already has (`get_portfolio`, `create_intent`) are
registered the same way as `echo`. The model never gets a raw SQL
or RPC URL (ADR security: tools are the sandbox).

MCP-shaped `list_tools` / `call_tool` is the boundary so a desktop
client or another agent runtime can attach without importing Python.

## Trade-offs

- **Accepted:** one extra process (Ollama/vLLM) locally.
- **Accepted:** tool-calling quality varies by local model. FakeLLM
  remains the CI path.
- **Rejected:** baking a 7B into the API image.
