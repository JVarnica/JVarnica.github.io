---
layout: project
title: ExecuChat
summary: A dual-mode Android AI chat app with on-device inference via ExecuTorch and optional online mode through a self-hosted gateway.
github: https://github.com/JVarnica/Execu_Chat

---

## Overview

ExecuChat is an android app with an offline and online mode. The offline mode was started first to see how well a model could run on a mobile phone. This was rather successful could run llama1B & llama3B, Qwen3-3B, whisper and Llava; when quantized to 4 bit integers. The limitation was apparent the conversation was stale, and could only use its pre-trained knowledge, so it had no up to date facts. Not really useful as a working chatbot hence online mode was created.

Using ExecuTorch to run these models is a great achievement not possible a few years ago, and will continue to improve what is capable on small devices. 

- **Offline mode**- Executorch vulkan & XNNPACK backend, chat with llama models, and qwen3_4B. Voice to text (asr) using whisper and image understanding using llava. More details in github repository. 

- **Online mode** connects to a FastAPI gateway for authentication and identification of the users, the gateway then forwards to each service. The features

    +  **stream chat vLLM**, for inference stream Qwen3-4B-NVFP4 responses
    +  **SearxNG**, the search engine to access web content
    +  **Research agent**, research-agent container
    +  **Auth/JWT**, authentication using JWT tokens so 30 mins session with silent refresh 
    +  **Redis**, to store session context 
    +  **Redis stream**, to queue the text chunks so can be retrieved and embedded.
    +  **Saved convos**, save to sqlite database using asynchronous sqlite (aiosqlite) for multiple users.
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

For a detailed breakdown of the server architecture, inference configuration, and multi-user design decisions, see the [Python Server hub](/projects/python-server-hub).
For detailed breakdown of research-agent architecture, see [Research Agent hub](/projects/research-agent-hub).

## Blog posts 

To understand the repositories, explain challenges and improvements blog posts are written. For example one is written on the docker server for vLLM and the challenges faced when building for multiple users, next why vLLM was used for inference and what configurations used. The blog posts:

- **why & inference vLLM**-[inference]({{ site.baseurl }}{% post_url 2026-03-24-inference %})
- **Part 1:Building a chatbot with vLLM** — [inference-server]({{ site.baseurl }}{% post_url 2026-04-08-inference-server %})
- **Part 2: Web Search Tool** - [agentic-search]({{ site.baseurl }}{% post_url 2026-04-20-agentic-search %})

