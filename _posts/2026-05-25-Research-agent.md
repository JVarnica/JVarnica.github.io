---
layout: post
title: "Making a Research Agent using LangGraph"
date: 2026-03-24
categories: [backend, vLLM, LangGraph]
tags: [research, vllm, searxng, langchain]
project: execuchat
---

## Making a Research Agent 

First the research agent has its own container within the docker server as will be calling vllm directly; just the endpoints/routes need to be named the same. Only the gateway can access this container as its on expose not ports like all containers.  

Now why is Langraph used in the first place, can just have a tool loop like for search; but this is not possible for research as the model will need to search a few different queries. It is difficult to robustly make the model output multiple correct tool calls, and harder to code without the graph like approach of LangChain.  Will show both result for the same query later on. 

```
# routes.py
@router.post("/research")
async def create_research(req: ResearchRequest, request: Request):

# gateway 
@app.post("/research")
async def research(request: Request):
    return await forward(request, os.environ["DPR_URL"])

@app.get("/research/{task_id}")
async def research_status(request: Request):
    return await forward(request, os.environ["DPR_URL"])

# etc events and delete as well 
```

This endpoint create_research calls create_task function, which calls redis.set which creates a string with the following fields: task_id, query, status, created_at, final_report and error. A hash could be used 
