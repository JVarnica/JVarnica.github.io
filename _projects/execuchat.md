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

ExecuChat an android app with an offline and online mode:

- **Offline mode**- Executorch vulkan & XNNPACK backend, chat with llama models, and qwen3_4B. Voice to text (asr) using whisper and image understanding using llava. 

- **Online mode** connects to a FastAPI gateway for authentication and identification of the users, the gateway then forwards to each service. The features

    +  **stream chat vLLM**, for inference stream Qwen3-4B-NVFP4 responses
    +  **SearxNG**, the search engine to access web content
    +  **Deep Research**, search online and give out a detailed report on findings
    +  **Auth/JWT**, authentication using JWT tokens so 30 mins session with silent refresh 
    +  **Redis**, to store conversation as pair list so can chunk for context management. 
    +  **Redis stream**, to queue the text chunks so can be retrieved and embedded.
    +  **Saved convos**, save to sqlite database using asynchronous sqlite for multiple users
    +  **RAG**, uses Qdrant to store Redis embeddings persistently, and then retrieve with query. 

## Why I built Execuchat

How coherent & intelligent are models on smartphones, how large can they be? Is multi-modality a possibility? On device is the safest no network requests, so no security issues but limited to running max three billion models which are a tad too small for real tasks. Therefore the online mode was built so the app is useful for a small amount of users. A 5060ti gpu is used with 16gb RAM therefore cannot use large models such as Qwen3-32B-NVFP4, this would take around 20gb RAM so have to contend with smaller than 10B models. Thus Qwen3-8B-NVFP4 is used, which is coherent and useful when search capabilities are added so can have relevant context. 

## Architecture Diagram (Online/Server)

/images/architecture-diagram.svg

## GIFs 

//TODO OF FINISHED VERSION. 


## Related repositories

For more details on the Android app look at the Execuchat repository, combines both UIs into one app.  For the details on the docker server look at the python-server. 

- Main Android app ![alt text](https://github.com/JVarnica/Execu_Chat)
- Docker/ python server ![alt text](https://github.com/JVarnica/vllm-server)

## Blog posts 

To understand the repositories, explain challenges and improvements blog posts are written. For example one is written on the docker server for vLLM and the challenges faced when building for multiple users, next why vLLM was used for inference and what configurations used. The blog posts:

- **why & inference vLLM**- /_posts/2026-03-24-inference.md
- **Building a chatbot API vLLM**- /home/julien/Documents/JVarnica.github.io/_posts/2026-04-08-inference-server.md

