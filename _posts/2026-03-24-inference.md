---
layout: post
title: "Why vLLM as inference engine!!"
date: 2026-03-24
categories: [backend, vLLM]
tags: [quantization, vllm, inference, llm]
project: execuchat
---

### vLLM why?
Pytorch for model inference is not optimal, as its primary use case is model training. This is mainly because a basic PyTorch inference setup is unbatched and unpaged, making it a lot slower than a proper inference server. For the inference server vLLM was used as it uses PyTorch under the hood - TorchAO for quantization and torch.compile to optimize the model graph (it's 1.05x to 1.9x quicker). 

Furthermore, vLLM supports a variety of HuggingFace models with various quantization techniques, including AWQ, INT4, FP8, and NVFP4. TensorRT, another option also from NVIDIA is now also easy to set up as an OAI compatible server and is a fine choice. 

Both are fast with their inference optimizations, with TensorRT being slightly quicker. But TensorRT doesn't support new models as quickly as vLLM. Hence, for prototyping vLLM is better because switching models is easy; this simplicity is its main advantage. I am using Nvidia's 25.12 vLLM release.

#### vLLM Advantages.

The following reasons are why vLLM can work as an inference engine:

- **Continuous Batching**: The time consuming part is moving the weights from gpu ram into the cuda cores for computation. With static batching, this loading happens once, after which a batch of prompts is computed, each with its fixed memory block. This is more efficient than doing it sequentially without batching. However, prompts(requests) are of different lengths so computation will finish at different times for each one. This leaves cuda cores sitting idle as finished requests can't be released easily; and the longest to shortest prompt disparancy can be huge. Thus, continuous batching therefore schedules work at the token level rather than batch level. When the last token in a sequence has been computed, another request can takes its place, preventing hardware from sitting idle.

- **Paged Attention**: Normally, each request is assigned a fixed contiguous buffer of size max_context_length ahead-of time. Paged Attention divides gpu memory into pages (i.e. 16 tokens). As the model generates the required tokens, the scheduler allocates only the amount of pages required. This is important as output is non-deterministic: a request can generate 20 or 150 tokens. This is why allocating the full buffer ahead-of time is so inefficient. 

- **Prefix caching**: The prompt is first divided into blocks and stored in a dictionary, with the hash of each text chunk as the key and its KV-cache computation as the value. For each new prompt, vLLM searches the dict and retrieves the longest matching hash and its value; avoiding repeated computation. This is important as the model is stateless, and so requires the previous prompts for context. A long conversation or many concurrent requests with same system message, would require recomputing many of the same hashes; so prefix caching saves time.

Paged Attention allows continuous batching to be efficient through just-in-time resource allocation. As requests arrive the necessary pages are allocated, and then immediately reassigned to another request when available. With prefix caching added, the system message and KV cache from the previous conversation will be instantly retrieved from cache, meaning only new tokens are computed. Hence, throughput is much higher than with a naive engine, and adding more users can even improve the hardware utilization compared to serving only a single request. This makes vLLM perfect as the foundation to serve the models. 
    
#### Considerations

When serving a model a quantized version is needed. Qwen3-8B is around 16gb in bf16 format, so it doesn't fit on the RTX 5060Ti GPU I am using. Therefore, NVFP4 the new 4-bit quantization for blackwell gpus is used, trained by RedHatAI. NVFP4 reduces Qwen3-8B memory footprint to approximately 6gb. This allows there to be nearly 7gb for context thats 48,224 tokens, allowing me to support nearly 6 users with 8192-token context each. Furthermore, quantization of kv_cache is needed as well from auto/bf16 to fp8 with per tensor scales allowing an increased capactiy of 96,224 tokens.

It would be nice to quantizethe  KV cache to NVFP4 as well, but not yet supported as of March 2026. This would probably also cause too much information loss, so an 8-bit representation of the KV cache is the current sweet spot. 



