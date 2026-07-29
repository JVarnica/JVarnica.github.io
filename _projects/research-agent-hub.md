---
layout: project
title: Research Agent
summary: "LangGragh agent: plans queries, searches in parallel, extracts and merges claims, reflects on gaps, and writes a structured markdown report"
github: https://github.com/JVarnica/research-agent
---

# Deep Research Agent

An autonomous LangGraph agent: plans queries, searches in parallel, extracts summaries, merges summaries into topic claims, reflects on gaps, and writes a structured markdown report. It is frontend agnostic, just emits events onto a Redis List which is then polled by the client. Given a research question, the agent plans a set of search queries, these queries are searched and the urls are then scraped and triaged into Documents for summarizing. From the docs a set of factual topic claims are made with sources, so can map docs to a specific topic. The evidence gathered is reflected upon whether it is sufficient or not, if not loops back with follow-up queries, if sufficient then goes to writing the report.

Currently consumed by [ExecuChat](/projects/execuchat/) but deployable independently with any frontend.

#### /research-agent
<video src="{{ '/videos/roman_emp_dp_varspeed.mp4' | relative_url }}"
       autoplay loop muted playsinline
       style="max-width: 360px; border-radius: 12px; display: block;">
</video>

## Architecture

The system is built as a LangGraph graph with a Redis-backed worker queue. Tasks are submitted via a FastAPI endpoint and processed asynchronously — the client connects to the polling endpoint which is called every 0.5 seconds, and receives events as the graph progresses through each node.

```
Query → Plan queries → [Search → Scrape → Summarise → Extract claims] → Reflect → Loop? → Write sections → Stitch report
                              ↑__________________________|  (if not sufficient)
```

### Graph Nodes

| Node | What it does |
|---|---|
| **Plan** | Generates structured search queries with rationale for each |
| **Search** | Runs queries in parallel, deduplicates URLs across loops |
| **Scrape** | Fetches and cleans page content |
| **Summarise** | Compresses each document to key findings and supporting quotes |
| **Extract claims** | Maps each doc summary to a specific topic|
| **Reflect** | Decides if evidence is sufficient or identifies a remaining knowledge gap |
| **Write sections** | Parallel section writing — each section receives only its assigned claims |
| **Stitch** | Assembles sections into a final report with reference list |

### Key Design Decisions

**Claims as topic map** — Aggregates the doc summaries into topic-level claims, it's a topic label with the doc_ids. This maps documents to specific topics making it easier for planning, as will just put the documents from that specific topic for different sections.

**Section Writer sees summaries & claims** - The doc summaries have the facts and quotes, claims just more info on topic. By having the information not aggregated it can write proper grounded reports not generalizations. 

**Reflection loop with gap detection** — the reflect node doesn't just ask "is this enough?" — it identifies the specific knowledge gap and generates targeted follow-up queries. If the same gap remains unfilled after a search loop, the node marks it as unfillable and proceeds rather than looping indefinitely.

**Reflection current understanding** - the reflect node doesn't just have a gap, it now also has current understanding so it knows what it has learned so far. So it can build on what it knows, this was added as would say the same gap in further loops now it has stopped.

**Worker queue over background tasks** — tasks are pushed to a Redis list and processed by a dedicated worker loop. This is more robust than asyncio background tasks for long-running jobs and survives server restarts without losing queued work.

**Frontend-agnostic event stream** — the agent emits named events at each phase (`status`, `planning`, `searching`, `doc_summarised`, `claims_extracted`, `section_written`, `complete`). These events are added on a Redis List, so the frontend can then poll this list for information.

---

## State

The graph uses a typed `OverallState` with custom reducers for parallel-safe accumulation — search queries, documents, summaries, and claims are all appended concurrently across parallel branches without race conditions.

---

## Stack

- **LangGraph** — graph compilation and state management
- **FastAPI** — lifecycle and API endpoints
- **Redis** — data in different Redis structures

---

## Repositories

[research-agent on GitHub](https://github.com/JVarnica/research-agent)

---

## Related Blog Posts

- [Making a Research Agent using LangGraph]({{ site.baseurl }}{% post_url 2026-05-25-Research-agent %}).
- [Incrementally Building Research Agents]({{ site.baseurl }}{% post_url 2026-05-27-Building-research-agent %})

