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

ExecuChat is a dual-mode Android chat app.

- **Offline mode** uses ExecuTorch for on-device inference
- **Online mode** connects to a FastAPI gateway in front of vLLM

## Why I built it

I wanted one app that could run locally on-device but also switch to a more capable server-backed mode.

## Related repositories

- Main Android app
- Python gateway / backend

## Blog posts in this series

These posts explain the main technical decisions behind the project.