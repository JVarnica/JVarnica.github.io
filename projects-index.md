---
layout: page
title: Projects
permalink: /projects/
---

## [ExecuChat](/projects/execuchat/)

A dual-mode Android AI chat app — offline inference on device via ExecuTorch, and an online mode backed by a self-hosted LLM gateway with streaming chat, web search, RAG, and a deep research agent.

**Android · Kotlin · ExecuTorch · Vulkan**

[View project →](/projects/execuchat/)

---

## [Python Server](/projects/python-server/)

The self-hosted backend powering ExecuChat's online mode — FastAPI gateway with JWT auth, vLLM inference, Redis session management, SearxNG search, and Qdrant RAG. Built for multi-user correctness with streaming via Redis Streams.

**Python · FastAPI · vLLM · Redis · Qdrant · Docker**

[View project →](/projects/python-server-hub/)

---

## [Deep Research Agent](/projects/research-agent-hub/)

A containerised LangGraph research agent that takes a query through iterative search, claim extraction, reflection, and report writing. Frontend-agnostic — communicates entirely via SSE events. Standalone deployable.

**Python · LangGraph · FastAPI · Redis · Docker**

[View project →](/projects/research-agent-hub/)

---

## [Caltech256 Classification](/projects/caltech256-classification/)

A systematic investigation into transfer learning — comparing 11 vision architectures across three fine-tuning strategies, augmentation regimes, and QAT vs PTQ quantization. Wanted to understand what worked, what happened if I tweak this learning rate,

**Python · PyTorch · timm · NVIDIA DALI · Executorch**

[View project →](/projects/caltech256-classification/)