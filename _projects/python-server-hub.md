---
layout: project
title: LLM Inference Server
summary: A self-hosted FastAPI inference gateway with vLLM, Redis, Qdrant, and SearxNG — powering the online mode of ExecuChat for multi-user streaming chat, RAG, and web search.
github: https://github.com/JVarnica/vllm-server
---

# LLM Server Backend 

A self-hosted LLM backend server built with vllm for multi-user access: has search, RAG, and deep research capabilities. This is the backend server for [ExecuChat](/projects/execuchat/) system. The whole backend runs as a single Docker Compose stack. Every service sits on an internal bridge network - only the gateway (`8080`), Prometheus (`9090`), and Grafana (`3000`) are exposed to the host.

---

## Why

On-device inference tops at ~3B models on high-end consumer phones, which is too small for proper conversations. Thus, a server was built to run larger models, with persistent memory and web search capabilities. The model used is Qwen3-8B-NVFP4 on a 5060ti with 16 GB VRAM, more details for this decision can be found [here](/_posts/2026-03-24-inference.md). The main challenge was making it available to multiple users, this meant a gateway to isolate each chat session for that user. A dedicated database is also needed, as offline only uses files as storage is insufficient for users as locks on read/write. 

---
## Architecture Diagram (Online/Server)

![Architecture diagram]({{ '/images/architecture-diagram.svg' | relative_url }})

---

## Stack

| Service | Technology |
|---|---|
| API Gateway | FastAPI (async) | Entry point, Routing |expose `8080` | 
| Inference | vLLM — Qwen3-8B-NVFP4 | OpenAI compatible |expose `8000`|
| Auth | JWT — 30 min sessions, silent refresh | expose `8090` |
| Redis | redis:8-alpine |Session Context, queue, streams| expose `6379` | 
| Search | SearxNG | Metasearch, JSON API |expose `8080`| 
| Research_agent| LangGraph, research, report, events | expose `8001`
| Qdrant | Vector store | Retrieval, embeddings| expose `6333` |
| Prometheus | Metrics collection | ports `9090` |
| Grafana | Dashboards of metrics | ports `3000` | 

No database used as aioSQLite is sufficient for small amount of users. 

---

## Prerequisites  

- Linux (tested on Ubuntu 24.04)
- NVIDIA GPU with 16 GB+ VRAM (built for an RTX 5060 Ti, SM 12.0 Blackwell)
- NVIDIA Container Toolkit (for GPU passthrough into the vLLM container)
- Docker & Docker Compose
- A Hugging Face token (to pull the model weights)


## Key Design Decisions

**vLLM for inference** — OpenAI endpoint with v1/chat/completions & stateless between requests so easy for multiple users. Main advantages PagedAttention and continuous batching, so can have multiple users at once and won't pad each request to context length. Full reasoning on this in the [inference blog post]({{ site.baseurl }}{% post_url 2026-03-24-inference %}).

**Redis context** —  Used to store user's session data. Redis is on RAM so its quick, so each python process can access the redis key it needs. User's conversation pairs and retrieved chunks are appended to a Redis list (rpush); search context is a Redis string/byte (set).   

**Chat persistence via AioSQLite** - Accessing the data efficiently is now important, can't keep using a file system (single user), so SQLite is used. Though it can only do a single READ/WRITE at a time, this is where aiosqlite comes in as its asynchronous so doesn't block.

**RAG via Qdrant** — Didn't want to have to worry about indexing and retrieval, needed persistent storage, and for it to be a service, easy to start as a docker container. Qdrant fit those perfectly and had separation between embedding and normal chat, postgre could have been used though but more complicated to set up and not a service. 

**Research agent via LangGraph** — Separate container as will call vllm itself, didn't want to mix tasks. Emits events (searching, triaging, summarizing, reflecting, planning, writing) by adding to a Redis lists, which is then polled by the client. More detail [research agent blog post]({{ site.baseurl }}{% post_url 2026-05-25-Research-agent %}) and on reasearch-agent hub. 

**JWT security/identification** — API framework as for mobile, so each request goes through gateway to validate the token which identifies the user, and whether it can access the redis keys. Automatic 30 mins refresh.

**Async chunking** - Needed chunking to happen in a background process while conversing, so doesn't block when trying to save the chat. So before chat call finishes the pair turn gets appended to redis, with overlapping windows for chunking. Each chunk id is added to the Redis stream, while chunks are added to hash sets so embedding json can be added when available.

- More information on the following discussed decisions can be found in the blog post about building a inference server [here]({{ site.baseurl }}{% post_url 2026-04-08-inference-server %}).

**Web Search Tool** - Model needs to know why it has searched, and as Qwen3 has been trained to use tools. This is far more effective than context injection. More details in the [agentic search blog post]({{ site.baseurl }}{% post_url 2026-04-20-agentic-search %}).

---

## Repository

[vllm-server on GitHub](https://github.com/JVarnica/vllm-server)
[research-agent](/_projects/research-agent-hub.md)

---

## Related Blog Posts

- [Why vLLM as inference engine](/_posts/2026-03-24-inference.md)
- [How I made a vLLM Docker inference server](/_posts/2026-04-08-inference-server.md)
- [Search with Tool Use](/_posts/2026-04-20-agentic-sear.md)
- [Making a Research Agent using LangGraph](/_posts/2026-05-25-Research-agent.md)
- [Incrementally Building Research Agents](/_posts/2026-05-27-Building-research-agent.md)()


