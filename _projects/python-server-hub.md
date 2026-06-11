---
layout: page
title: Python Server
permalink: /projects/python-server/
---

# Python Server

A self-hosted LLM backend built for multi-user access — handles inference, search, RAG, authentication, and a deep research agent as independent containerised services.

This is the backend server for [ExecuChat](/projects/execuchat/) system.

---

## Why

On-device inference tops at ~3B models on high-end consumer phones, which is too small for proper conversations. Thus, a server was built to run larger models, with persistent memory and web search capabilities. The model used is Qwen3-8B on a 5060ti with 16 GB VRAM, more details for this decision can be found [here](/_posts/2026-03-24-inference.md). The main challenge was making it available to multiple users, this meant a gateway so can isolate each chat session for that user. A dedicated database is also needed, as offline only files as storage is sufficient.

---
## Architecture Diagram (Online/Server)

![Architecture diagram]({{ '/images/architecture-diagram.svg' | relative_url }})

---

## Stack

| Service | Technology |
|---|---|
| API Gateway | FastAPI (async) |
| Inference | vLLM — Qwen3-8B-NVFP4 |
| Auth | JWT — 30 min sessions, silent refresh |
| Session context | Redis |
| Search | SearxNG (self-hosted) |
| chunking async | Redis Streams |
| Vector store | Qdrant |
| Saved conversations | SQLite (async) |
| Research agent | LangGraph, containerised |
| Deployment | Docker Compose |
| Monitoring | Prometheus |

---

## Key Design Decisions

**vLLM for inference** — OpenAI endpoint with v1/chat/completions & stateless between requests so easy for multiple users. Main advantages PagedAttention and continuous batching, so can have multiple users at once and won't pad each request to context length. Full reasoning on this in the [inference blog post]({{ site.baseurl }}{% post_url 2026-03-24-inference %}).

**Redis context** —  Used to store user's session data. Redis is on RAM so its quick, so each python process can access the redis key it needs. User's conversation pairs and retrieved chunks are appended to a Redis list (rpush); search context is a Redis string/byte (set).   

**Chat persistence via AioSQLite** - Accessing the data efficiently is now important, can't keep using a file system (single user), so SQLite is used. Though it can only do a single READ/WRITE at a time, this is where aiosqlite comes in as its asynchronous so doesn't block.

**RAG via Qdrant** — Didn't want to have to worry about indexing and retrieval, needed persistent storage, and for it to be a service easy to start as a docker container. Qdrant fit those perfectly and had seperation between embedding and normal chat, postgre could have been used for though but more complicated to set up and less reliable i felt.

**Research agent LangGraph** — Separate container will call vllm lots. Emits events (searching, triaging, summarizing, reflecting, planning,...) to client so can see progress. More detail [research agent blog post]({{ site.baseurl }}{% post_url 2026-05-25-Research-agent %}).

**JWT security/identification** — API framework as for mobile, so each request goes through gateway to validate token which identifies the user and whether it can access the redis keys. Automatic 30 mins refresh.

**Async chunking** - Needed chunking to happen in a background process while conversing so no block when trying to save the chat. So before chat call finishes the pair turn gets appended to redis, with overlapping windows for chunking. Each chunk id is added to the Redis stream, while chunks are added to hash sets so embedding json can be added when available.

- More information on the following discussed decisions can be found in the blog post about building a inference server [here]({{ site.baseurl }}{% post_url 2026-04-08-inference %}).

**Web Search Tool** - Model needs to know why it has searched, and as Qwen3 has been trained to use tools; it is a lot more effective than context injection. More details in the [agentic search blog post]({{ site.baseurl }}{% post_url 2026-04-20-agentic-search %}).

---

## Repository

[vllm-server on GitHub](https://github.com/JVarnica/vllm-server)

---

## Related Blog Posts

- [Why vLLM as inference engine]({{ site.baseurl }}{% post_url 2026-03-24-inference %})
- [How I made a vLLM Docker inference server]({{ site.baseurl }}{% post_url 2026-04-08-inference-server %})
- [Making a Research Agent using LangGraph]({{ site.baseurl }}{% post_url 2026-03-24-Research-agent %})
- [Incrementally Building Research Agents]({{ site.baseurl }}{% post_url 2026-05-27-Building-research-agent %})
