---
layout: project
title: ExecuChat
slug: execuchat
summary: A dual-mode Android AI chat app with on-device inference via ExecuTorch and optional online mode through a self-hosted gateway.
image: /images/execuchat-arch.png
github: https://github.com/JVarnica/Execu_Chat.git
server_repo: https://github.com/JVarnica/vllm-server.git
---

## Overview

ExecuChat is an android app with an offline and online mode, the offline mode was started first as wanted to see how well a model could run on a mobile phone. This was successful could run different llama models with whisper and llava; but the limitation was apparent the conversation was stale and can oly use its pre-trained knowledge. Not really useful as a working chatbot hence online mode was created. 

- **Offline mode**- Executorch vulkan & XNNPACK backend, chat with llama models, and qwen3_4B. Voice to text (asr) using whisper and image understanding using llava. 

- **Online mode** connects to a FastAPI gateway for authentication and identification of the users, the gateway then forwards to each service. The features

    +  **stream chat vLLM**, for inference stream Qwen3-4B-NVFP4 responses
    +  **SearxNG**, the search engine to access web content
    +  **Research agent**, research-agent container
    +  **Auth/JWT**, authentication using JWT tokens so 30 mins session with silent refresh 
    +  **Redis**, to store session context 
    +  **Redis stream**, to queue the text chunks so can be retrieved and embedded.
    +  **Saved convos**, save to sqlite database using asynchronous sqlite for multiple users
    +  **RAG**, uses Qdrant to store Redis embeddings persistently, and then retrieve with query. 

## Why I built Execuchat

How coherent & intelligent are models on smartphones, how large can they be? Is multi-modality a possibility? On device is the safest no network requests, so no security issues but limited to running max three billion models which are a tad too small for real tasks. Therefore the online mode was built so the app is useful for a small amount of users. A 5060ti gpu is used with 16gb RAM therefore cannot use large models such as Qwen3-32B-NVFP4, this would take around 20gb RAM so have to contend with smaller than 10B models. Thus Qwen3-8B-NVFP4 is used, which is coherent and useful when search capabilities are added so can have relevant context. 

## Architecture Diagram (Online/Server)

![Architecture diagram]({{ '/images/architecture-diagram.svg' | relative_url }})

## GIFs 

### Offline mode — Llama 3B 
<video src="{{ '/videos/llama3B_vulkan.mp4' | relative_url }}"
       autoplay loop muted playsinline
       style="max-width: 360px; border-radius: 12px; display: block;">
</video>

### Online mode — Qwen3_8B 
#### /chat
<video src="{{ '/videos/execu_chat_demo_720p.mp4' | relative_url }}"
       autoplay loop muted playsinline
       style="max-width: 360px; border-radius: 12px; display: block;">
</video>


## Related repositories

For more details on the Android app look at the Execuchat repository, combines both UIs into one app.  For the details on the docker server look at the python-server. 

- Main Android app [Execu_Chat](https://github.com/JVarnica/Execu_Chat)
- Docker/ python server [vllm-server](https://github.com/JVarnica/vllm-server)
- research agent [research-agent](https://github.com/JVarnica/research-agent)

For a detailed breakdown of the server architecture, inference configuration, and multi-user design decisions, see the [Python Server hub](/projects/python-server-hub.md/).
For detailed breakdown of research-agent architecture, see [Research Agent hub](/_projects/research-agent-hub.md).

## Blog posts 

To understand the repositories, explain challenges and improvements blog posts are written. For example one is written on the docker server for vLLM and the challenges faced when building for multiple users, next why vLLM was used for inference and what configurations used. The blog posts:

- **why & inference vLLM**-[inference.md](/_posts/2026-03-24-inference.md)
- **Part 1:Building a chatbot with vLLM** — [inference-server.md](/_posts/2026-04-08-inference-server.md)
- **Part 2: Web Search Tool** - [agentic-search.md](/_posts/2026-04-20-agentic-search.md)

