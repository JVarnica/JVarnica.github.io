---
layout: page
title: ExecuChat
permalink: /projects/execuchat/
---

# ExecuChat

A dual-mode Android AI chat app — offline inference running directly on device via ExecuTorch, and an online mode connecting to a self-hosted LLM gateway with search, RAG, and a deep research agent.

## Why I Built It

The starting question was straightforward: how coherent can a language model actually be when running entirely on a smartphone, with no network requests? The offline mode answered that — Llama and Qwen3 models run on device via ExecuTorch, with Whisper for voice input and LLaVA for image understanding. It worked, but the limitation became obvious quickly. Models under ~3B parameters are too constrained for genuinely useful tasks, and conversation stays stale — the model only knows what it was trained on.

That's why the online mode was built. The goal was a useful AI assistant for a small number of users, backed by a self-hosted server running on a 5060ti (16 GB VRAM), with web search and persistent memory to compensate for the model size constraint.

---

## Offline Mode

Runs entirely on device — no network requests, no authentication, no external dependencies.

- **Inference**: ExecuTorch with Vulkan (GPU) and XNNPACK (CPU) backends
- **Models**: Llama family, Qwen3-4B
- **Voice input**: Whisper ASR for speech-to-text
- **Image understanding**: LLaVA for multimodal input
- **Android**: Kotlin, custom chat UI, model asset management via AssetMover

The primary constraint is VRAM — consumer Android hardware limits practical model size to around 3B parameters. Beyond that, inference becomes too slow to be usable. This ceiling is what motivated the online mode.

---

## Online Mode

Connects to a self-hosted FastAPI gateway which handles authentication and routes requests to each service.

### Gateway & Auth
- **FastAPI** gateway with JWT authentication — 30-minute sessions with silent token refresh
- **Redis** for session context storage across requests
- User identification at the gateway so all downstream services remain stateless

### Inference
- **vLLM** serving Qwen3-8B-NVFP4 with streaming responses via SSE
- NVFP4 quantization chosen to fit within 16 GB VRAM while preserving coherence
- Streaming chunks queued via **Redis Streams** for reliable delivery and downstream embedding

### Search & RAG
- **SearxNG** self-hosted search engine — web content retrieved and injected as context
- **Qdrant** vector store — Redis stream embeddings persisted for retrieval
- RAG query pipeline retrieves relevant past context on each new message

### Research Agent
- Containerised deep research agent with **LangGraph** — frontend-agnostic, communicates via SSE events
- Performs multi-step web research with reflection and iterative search loops
- Designed as a standalone service so it can be consumed by any frontend, not just this app

### Persistence
- **SQLite** (async) for saved conversations — supports multiple concurrent users
- Conversation list synced to the Android client

---

## Architecture

<!-- Add architecture diagram here -->

---

## Repositories

| Component | Language | Repo |
|---|---|---|
| Android app | Kotlin | [Execu_Chat](https://github.com/JVarnica/Execu_Chat) |
| Gateway + inference server | Python | [vllm-server](https://github.com/JVarnica/vllm-server) |
| Research agent | Python | [research-agent](https://github.com/JVarnica/research-agent) |

For a detailed breakdown of the server architecture, inference configuration, and multi-user design decisions, see the [Python Server hub](/projects/python-server/).

---

## Related Blog Posts

- [Why vLLM as inference engine]({{ site.baseurl }}{% post_url 2026-03-24-inference %})
- [How I made a vLLM Docker inference server]({{ site.baseurl }}{% post_url 2026-04-08-inference-server %})
- [Making a Research Agent using LangGraph]({{ site.baseurl }}{% post_url 2026-03-24-Research-agent %})
- [Incrementally Building Research Agents]({{ site.baseurl }}{% post_url 2026-05-27-Building-research-agent %})
- [Web Search Tool]({{ site.baseurl }}{% post_url 2026-04-20-agentic-search %})