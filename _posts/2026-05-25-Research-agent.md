---
layout: post
title: "Making a Research Agent using LangGraph"
date: 2026-05-25
categories: [backend, vLLM, LangGraph]
tags: [research, vllm, searxng, langchain]
project: execuchat
---

# Making a Research Agent 

## Intro

The research agent has its own container within the docker server as it calls vllm directly. Only the gateway can access this container as its on expose, so the endpoints on the gateway just need to forward to the research agent's endpoints. Why is Langraph used in the first place - couldn't this just have been a tool loop like for search? For research this is harder as there are several distinct tasks: querying, searching, summarizing, reflecting, planning and writing. Furthermore the model needs to search a few different queries, and getting a model to output multiple correct tool calls reliably for this is flaky. The graph like approach of LangChain makes all these steps a lot easier with its abstractions. 
<table>
<tr>
<td><pre><code># route.py in research  
@router.post("/research")
async def create_research(req: ResearchRequest, request: Request):

@router.get("/research/{task_id}")
async def research_status(task_id: str, request: Request):
 
</code></pre></td>
<td><pre><code> # gateway 
@app.post("/research")
async def research(request: Request):
    return await forward(request, os.environ["DPR_URL"])

@app.get("/research/{task_id}")
async def research_status(request: Request):
    return await forward(request, os.environ["DPR_URL"])
</code></pre></td>
</tr>
</table>

## How it works

Redis is used extensively as the source of truth because of its data types and its in memory nature. Each time the research endpoint is called it creates a string key with a record. This record has a task_id to identify each research process, and a status field with PENDING,RUNNING,COMPLETED,CANCELLED & FAILED to indicate progress. It also stores the user's query and the final report if available. Then a payload with task_id, query, and max_research_loops is pushed to a redis list called dpr:queue.
```
record = {
        "task_id": task_id,
        "query": query,
        "status": TaskStatus.PENDING,
        "created_at": time.time(),
        "final_report": "",
        "error": "",
    }
await redis.set(_task_key(task_id), json.dumps(record), ex=TASK_TTL_SECONDS)
await redis.rpush("dpr:queue", json.dumps({"task_id": task_id, "query": query, "max_research_loops": max_research_loops,}))
```

So on the redis side the research data is there. Will use Redis's blpop method which blocks until an element appears in the list, then takes and removes it. This is better than polling as blpop parks on connection and awakes the instance a job arrives, whereas polling would return empty polls. A list is deliberately preferred over a semaphore in the worker loop, as it is a source truth and always sure the workers will get the first element. A Redis stream is an option but not enough elements are getting taken from it to warrant one, and the task last a while (not like in the embedding pipeline).


As the container is a FastAPI app it has a lifespan, so when the app starts, worker tasks are created running worker_loop method, which pulls from the list. The number of workers is 3, so there can be 3 concurrent research task at once and the 4th stays in the queue. Once it has gotten the job data then _run_graph is ran, meaning each worker completes the job task then is available. 
```
# main.py
state.worker_tasks = [
            asyncio.create_task(worker_loop(state.redis, state.graph),
            name="research_worker_{i}",)
            for i in range(NUM_WORKERS)
]
async def worker_loop(redis: aioredis.Redis, graph):
    while True:
        try:
            item = await redis.blpop("dpr:queue", timeout=5)
            if not item:
                continue  # timeout, loop again
            job = json.loads(item[1])
            task_id = job["task_id"]

            # cancellation logic if cancelled while in queue
            ...

            await _run_graph(
                task_id = task_id,
                query = job["query"],
                max_research_loops = job["max_research_loops"],
                graph = graph,
                redis = redis
            )
```
On shutdown the worker tasks are cancelled, then gathered with a timeout so no suspended calls just stuck there. The http connections are also closed to search and redis clients, don't aclose the vllm connection. The task is cancelled with the cancel key, a Redis string called "dpr:cancel:task_id", is created in the cancel research endpoint. The worker loop then checks if this string exists if so then the redis string is deleted. This logic is applied in the worker loop to cancel queued tasks, and in the run_graph method where a separate worker polls the cancel key.
``` 
# in _run_graph() in task.py
        config = {"configurable": {"thread_id": task_id},
                "recursion_limit": 35}
        run_task = asyncio.create_task(graph.ainvoke(initial_state, config=config))
        watcher = asyncio.create_task(_cancel_watcher(redis, task_id, run_task))
        try:
            final_state = await run_task
        finally: 
            watcher.cancel()
```
The connection pools for the clients to vllm, redis and search are also set up in lifespan; don't want to create a new client on each call. 
For the connection to vllm a client class is created, with the instantiated LLMs, calling **ChatOpenAI** only once on startup. There are 3 different model configurations, two for structured calls returning a schema one with 8196(fast) and 2048(cheap) max tokens. Then the writer which writes free prose for the sections so has a repetition penalty, which cannot be done for non structured calls. The ChatOpenAI class has a **with_structured_format** method which has schema as input and told to output a json schema. Luckily vllm can use xgrammar to have constrained generation or this wouldn't be possible.
```
def structured_llm(self, schema: Type[T]) -> BaseChatModel:
    
    return self._fast.with_structured_output(schema, method="json_schema")
```
This structured format is a must, and using Pydantic's **BaseModel** allows the model to follow the schema set on generation. Each node in the graph has its own task and schema. Another useful thing is the **AsyncRedisSaver**, when setup as a checkpointer so the graph is saved to redis. Each state has its own checkpointer keyed by thread_id so no corruption. Honestly didn't use it much though as LangGraph errors are quite descriptive.  

## The graph
```
generate_queries → search_all_queries → scrape_node
      ↑                                      │
      │                              fan-out summarize_one_doc (parallel, one per doc)
      │                                      │
      └── reflect ←──── extract_claims ←─────┘
            │
            └→ plan_report → fan-out write_section (parallel, one per section)
                                   │
                              stitch_report → END
```

The graph becomes active once **_run_graph** is called, it first updates the status to running and then initializes **OverallState** this stores the important variables with their respective reducers so LangGraph knows what operation to perform. Each is useful for a specific node, doc_summaries is where overview, quotes and key_findings are stored so need to be accumulated- operator.add is hence used for appending. Though the urls found for that loop are overwritten, search_hits has no operator.
```
class OverallState(TypedDict):
    # ---- Input ----
    task_id: str
    original_query: str
    max_research_loops: int
    search_queries: Annotated[list[Query], operator.add]
    search_hits: list[Document] # un-scraped content
    hits_to_scrape: list[Document] # overwrite per loop 
    raw_docs: Annotated[list[Document], operator.add] # scraped content
    doc_summaries: Annotated[list[DocSummary], operator.add]
    claims: Annotated[list[Claim], merge_claims_by_id]
    understanding_history: Annotated[list[str], operator.add] 
    reflection_history: Annotated[list[ReflectionResult], operator.add]
    reflected_doc_ids: Annotated[list[str], operator.add]
    seen_urls: Annotated[list[str], operator.add]
    written_sections: Annotated[list[WrittenSection], operator.add]
    research_loop_count: int
    is_sufficient: bool
    plan: NotRequired[ReportPlan] #not required optional as first nodes wont output anything to it
    final_report: NotRequired[str]
```
## Nodes

**Generate_queries**, uses the cheap structured model configuration, to generate 3-5 diverse queries. The result outputs a `QuerySet`, each query is a pydantic basemodel with query, rationale and category fields. The category field is used as SearXNG won't provide papers on a general query, only science for example. When the call to Searxng allowed all categories it got too much junk so a category per query was made, still mostly outputs 'general' though. The query_ids though are created afterwards server side can't ask a model to do this.

**Search_queries_seq**, searching SearXNG needs to happen sequentially as state needs to know which url has been seen, this cannot happen on a parallel process. First it needs to also keep track of the completed query, and seen_urls is a list turned into a set which is the previous loops urls, if a hit url isn't in seen_url then its appended to new_hist and added to seen_urls. These new_hits are only for this loop known as the search_hits, and what will be scraped. If accumulated search_hits throughout loops would re-scrape the hits. 
```
for query in pending:
        _emit(task_id, "searching", query=query.query)
        hits = await search_query(query, seen_urls=seen_urls, max_results=15)
        completed_queries_id.append(query.id)

        for h in hits:
            if h.url not in seen_urls:
                new_hits.append(h)
                seen_urls.add(h.url)
        _emit(task_id, "search_complete",query=query.query, hit_count=len(hits))

    return {
        "search_hits": new_hits,
        "searched_queries_ids": completed_queries_id,
        "seen_urls": [h.url for h in new_hits],
    }
```
Another thing to note is each search_hit is a `Document` with a consistent structure, so has an id, url, title, snippet, source_query_id, and search_score. This metadata is now easily accessible, otherwise couldn't be used in a loop to discern the urls if all in a unstructured string. Searched_query ids is also tracked so sequentially loops won't ask a similar query.

**Scrape_node**, here just `scrape_hits` is called and returns raw_docs. This uses a semaphore (6) and scrapes these urls concurrently. This is done as might have 50 urls so cannot scrape them all at once. Content is capped at 8000 characters so each page has enough detail, and drops anything under 500 characters.  

### Summarization

**fan-out summarization** , here is where LangGraph's orchestration is useful. This is a conditional edge so a single doc is given per send to summarise_one_doc node, which will then run in parallel. It also checks whether this doc has been already summarized, if so skips straight to `extract_claims`.
```
def fan_out_summarize(state: OverallState) -> list[Send]:
    summarized_ids = {s.doc_id for s in state.get("doc_summaries", [])}
    pending = [d for d in state["raw_docs"] if d.id not in summarized_ids]
    if not pending:
        return "extract_claims" # skip summary no docs
    return [
        Send("summarize_one_doc", {"task_id": ..., "doc": d, "original_query": ...,})
        for d in pending
    ]
```
Each branch produces a DocSummary with following fields:
    - title, this is the url title 
    - overview, what this document covers, its scope and arguments in 1-3 sentences.
    - key_findings, 3-8 specific findings, each being a date, number or named person/place/event. This adds detail
    - quotes, up to 4 short quotes this is used for citations. 
    -relevance, does this doc help answer the question?

The Doc_summary node is very important as it first triages the scraped content based on whether this doc is relevant or not. The plan at the start was to have a  separate triaging call to the model for relevancy once scraped, but this was inefficient took nearly 2 minutes to triage 60 urls (with concurrency).
With send which also works in parallel you can see the summaries as they appear, more information for the user; the overall time is very similar of both strategies though. Lastly, if a branch's LLM call fails then relevant is just set to false no raising, and manually input the title, url,doc_id.

### Claims

**Extract_claims** aggregates the new summaries into topic-level claims. A claim is deliberately just a topic label plus the doc_ids covering it, the actual evidence stays in the doc summaries, which the section writer reads directly. The example below shows for the topic 'EU economic stagnation 2023' that there are 7 docs, so has a high confidence for this topic label. The prompt forces one of three actions per topic: MERGE into an existing claim (same id re-emitted so the reducer replaces it, doc_ids combined), NEW, or DROP for anything too vague to use. This keeps the claim set from fragmenting across loops.
```
 "claims": [
    {
      "id": "c_964447aa",
      "statement": "EU economic stagnation, 2023",
      "source_doc_ids": [
        "74879cc64fe7",
        "ba77cd698989",
        "c522fd9b8893",
        "9b9434180fa6",
        "995afe430134",
        "5f5cc5ccfe50",
        "ed4f12d2e14d"
      ],
      "confidence": "high",
      "merges_into": null
    },]
```
### Reflection Loop

**Reflect** is the loop decision. It's shown the previous reflections, the claims index, the evidence split into *previous*(last loops) vs *new*(this loop), and every query already searched. Previous reflections is a string format of reflection history instead of json, so model can understand the current understanding and knowledge gap better. The model reflects whether this evidence is sufficient if true moves onto `plan_report`, if not sufficient, then a knowledge gap is written. The model also has to output its current understanding so knows its progress next loop; but the key is the model judges whether the new evidence resolves the identified gap from last loop. If a gap is identified then goes back to `search_all_queries` so can find relevant content to fulfill this gap. If a gap was searched and nothing came back, it's told to stop asking, as that gap is unfillable from web search.
 
There are two non-LLM guards on top. If a loop produced zero new relevant summaries, sufficiency is forced without spending an LLM call. And if the model says "not sufficient" but all its follow-up queries are duplicates, termination is forced. The router then sends it back to search if all good but insufficient or on to planning:
 
```python
def reflect_router(state: OverallState) -> str:
    if state["is_sufficient"]:
        return "plan_report"
    if state["research_loop_count"] >= state["max_research_loops"]:
        return "plan_report"
    return "search_all_queries"
```
### Writing the report
 
**plan_report** produces a structure only, a title and 3–5 sections, each with a specific angle (max 300 chars) and the claim IDs. Here the overall plan is created and each section has a max of 8 claims, so its concise. The prompt explicitly bans writing prose here, and angle has a max_chars, as angle went into prose which derails whole report at first. Also tries to limit hallucination by making sure a section only exists, if enough evidence. Weak claims are allowed to be orphaned as don't help to answer the question
 
**write_section** is the second `Send` fan-out, one branch per section. Each branch gets its section's claims plus the supporting summaries, ranked by doc importance and capped at 8 summaries so the prompt stays around 5k tokens. The writer which again has a repetition penalty must cite every fact inline as `[d_<doc_id>]` and is forbidden from introducing facts that aren't in the claims or summaries. I haven't forced a length for the body as each section feels well written and concise, the max generation tokens is 8192 tokens anyway.
 
**stitch_report** assembles everything. Sections are ordered per the plan, doc_ids are collected in order with standard academic numbering, and a regex pass rewrites `[d_fb2843aa4730]` into `[3]`, dropping citations that don't resolve to a real scraped document — so the model can't hallucinate a reference. Adjacent citation groups get merged (`[8, 9][10]` → `[8, 9, 10]`), an intro and conclusion are generated from the section summaries. The report is made in bits with the title first appended, with each ordered section with the merged numeric citations, then the conclusion and references are added. The references list links each number back to its URL. The final markdown is returned as `final_report` a state field.

### Progress Events 

The graph has now reached completion (run task is done), final report is now available. The method `_update_status` is now called with the final report, and TaskStatus.COMPLETE, it reads the `dpr:task:{task_id}` string, and then overwrites it with the final report and TaskStatus.COMPLETE. Next `channel.emit("complete",...)`, appends (rpush) to Redis List `events:{task_id}`. The string `dpr:task:{task_id}` is the current state snapshot created at the start, and the list holds all the events in order and is what the Android client polls. 

As mentioned, the final report is emitted when done instead of streamed, so a polling endpoint is used. This endpoint polls the Redis List with `lrange(f"events:{task_id}", since -1)`, so it takes all elements since its last poll this is quite simple. So each node needs to emit data to this Redis List for an active UI.
Here below is all emit calls in the research pipeline:
```
_emit(task_id, "stage", stage="generating_queries", message="Planning search strategy")
_emit(task_id, "queries_generated", count=len(queries),
          queries=[{"query": q.query, "rationale": q.rationale, "category": q.category} for q in queries])
_emit(task_id, "searching", query=query.query)
_emit(task_id, "search_complete", query=query.query, hit_count=len(hits))
_emit(task_id, "scrape_complete", scraped=len(docs))
_emit(task_id, "doc_summarized", doc_title=summary.title, overview=summary.overview, relevant=summary.relevant) #per doc
_emit(task_id, "stage", stage="extracting_claims", message="Aggregating sources into topic claims")
_emit(task_id, "claims_extracted", count=new_count, merged=merged_count, statements=statements)
_emit(task_id, "stage", stage="reflecting", loop=loop)
_emit(task_id, "reflection", sufficient=is_sufficient, current_understanding=result.current_understanding, knowledge_gap=result.knowledge_gap)
_emit(task_id, "stage", stage="planning", message="Structuring the report")
_emit(task_id, "plan_ready", title=plan.title, sections=[{"title": s.title, "claim_count": len(s.claim_ids)} for s in plan.sections])
_emit(task_id, "writing_section", title=section.title)
_emit(task_id,"section_written",title=section.title,citations=len(citations_used),)
_emit(task_id, "stage", stage="stitching", message="Assembling final report")
_emit(task_id, "report_ready", length=len(final))
```

### Conclusion

This is the structure of the research agent, with LangGraph have been able to make this an iterative process to get a final well structured report. The overall state is a nice addition just needs a reducer and you return it and don't worry about it. The graph approach with conditional edges so can go from extracting claims back to generating queries is good, so re-usable a very good framework. This wasn't the first iteration did one with Redis queues each tasks was queued and would happen sequentially the report difference isn't comparable.
 
There isn't subgraphs used as still had to keep the graph simple, can't be running too many processes on a single 5060Ti GPU. Hence it also blpops from a list with a number of workers for concurrency, this isn't possible on a bigger scale. What I could have done differently is not run the graph to completion with `graph.ainvoke(...)`,and used `stream_mode="messages"` so could stream the final report; this would have been done if bigger model but instead done by sections so quicker and more focused. 

For more information on how the research agent was built read this blog [Building a research-agent](_posts/2026-05-27-Building-research-agent.md)

 


