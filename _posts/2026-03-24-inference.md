---
layout: post
title: "Why vLLM as inference engine!!"
date: 2026-03-24
categories: [backend, vLLM]
tags: [quantization, vllm, inference, llm]
project: execuchat
---

### vLLM why?
You cannot use naive Pytorch for model inference, it is used for model training. So vLLM was used as it uses PyTorch under the hood TorchAO for quantization and
torch.compile to optimize the model graph which has significant speedups (1.05x to 1.9x). Therefore using vLLM you are able to use a variety of HuggingFace models
with various quantization AWQ, INT4, FP8, NVFP4, you just change the URL to point to that repo. So it is simple to use, TensorRT the next option also from nvidia is now easy
to set up as an OAI compatible server aswell.

Both are quick because of inference techniques with TensorRT being slighlty quicker, but it doesn't have compatibility with new models as quickly as vLLM. Hence, for prototyping vLLM
is better as switching models is easy, this simplicity is its main advantage. The vLLM release I am using is Nvidia's 25.12 vLLM release

#### vLLM Advantages.

So that vLLM can be used as an inference engine there have been the following improvements:
- **Continuous Batching**, what is time-consuming is moving the weights from gpu ram into the cuda cores for computation ( i.e. token x q_proj), with static batching it does this loading
        once then you compute the batch of prompts, each using different memory block. This is more efficient than doing it sequentially without batching, but prompts/requests are of different lengths so compute will be finished at different times for each one. This will leave cuda cores sitting idle as you cannot release them easily. No issue when request fixed length
        but with modern LLMs with 16k context length +, the longest to shortest prompt disparancy can be huge. So continuous batching waits at the token level not batch level, so when the last
        token in a sequence has been computed another request can takes its place. No sitting idle.
- **Prefix caching**, where you first chunk the prompt into blocks and store it in a dictionary, with the key being the text chunk (hash) and value being the kv_cache(computation). 
        Thus for every new prompt it searches the dict, and retrieves the longest matching hash and its value; this saves having to redo the computation. This is important as the model 
        is stateless so expects the previous prompts for context. A long conversation means a lot of computing the exact same hashes therefore this saves time. 
- **Paged Attention**, normally for each request a fixed contiguous buffer of size max_context_length is created ahead-of time. Paged Attention chunks the gpu memory into pages (i.e. 16 tokens)
        then the model generates the required tokens, with the scheduler allocating the amount of pages needed. This is important as output is non-deterministic can generate 20 or 150 tokens, this
        is why allocating ahead-of time is so inefficient. 

Paged Attention allows continous batching to be efficient with a just in time allocation of resources, as requests come in the neccessary pages are allocated, then allocated again straight away to another request when done as now available. Now adding prefix caching the system message and previous converstaion turn's kv_cache will be instantly retrieved with only new tokens being computed. Thus the throughput is much higher than using a naive engine, and adding users increases the speed than using it yourself. This makes it perfect as the foundation to run the models. 
    
#### Considerations

When serving a model a quantized version is needed qwen3-8B is around 16gb in bf16 format, so doesn't fit into 5060ti GPU I am using. Therefore, NVFP4 the new 4-bit representation for blackwell gpu I am using trained by RedHatAI, reduces Qwen3's memory footprint to approx 6gb. This allows there to be nearly 7gb for context (kv_cache) thats 48,224 tokens, allowing me to have nearly 6 users with 8192 context. Furthemore quantization of kv_cache is needed aswell from auto/bf16 to fp8 with per tensor scales allowing 96,224 tokens.

It would be nice to also quantize kv_cache to NVFP4 but not yet supported (march), also probably too much of a loss in info so 8-bit representation of kv_cache is the sweet spot. 



