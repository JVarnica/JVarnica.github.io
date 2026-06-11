---
layout: page
title: Deep Research Agent
permalink: /projects/research-agent/
---

# Deep Research Agent

An agentic search system built with LangGraph, takes a query and outputs a report. 

A containerised agentic research system built with LangGraph — takes a query, runs iterative web research loops, and produces a structured, cited report. Designed to be frontend-agnostic, communicating entirely through SSE events.

Currently consumed by [ExecuChat](/projects/execuchat/) but deployable independently with any frontend.

---

## What It Does

Given a research question, the agent plans a set of search queries, scrapes and summarises the results, extracts factual claims with source attribution, reflects on whether the evidence is sufficient, and either loops with follow-up queries or writes the final report. Each phase emits events in real time so the client can show progress as it happens.

---

## Architecture

The system is built as a LangGraph graph with a Redis-backed worker queue. Tasks are submitted via a FastAPI endpoint and processed asynchronously — the client connects to an SSE stream and receives events as the graph progresses through each node.

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
| **Extract claims** | Aggregates findings into atomic, source-cited factual claims |
| **Reflect** | Decides if evidence is sufficient or identifies a remaining knowledge gap |
| **Write sections** | Parallel section writing — each section receives only its assigned claims |
| **Stitch** | Assembles sections into a final report with reference list |

### Key Design Decisions

**Claims as the unit of knowledge** — rather than passing raw documents to the writer, the graph first extracts atomic claims with source attribution. The planner then assigns specific claim IDs to each section. This keeps the writer grounded and makes citation audit straightforward.

**Reflection loop with gap detection** — the reflect node doesn't just ask "is this enough?" — it identifies the specific knowledge gap and generates targeted follow-up queries. If the same gap remains unfilled after a search loop, the node marks it as unfillable and proceeds rather than looping indefinitely.

**Worker queue over background tasks** — tasks are pushed to a Redis list and processed by a dedicated worker loop. This is more robust than asyncio background tasks for long-running jobs and survives server restarts without losing queued work.

**Frontend-agnostic event stream** — the agent emits named SSE events at each phase (`status`, `planning`, `searching`, `doc_summarised`, `claims_extracted`, `section_written`, `complete`). The client subscribes and renders whatever it wants from those events — the agent has no knowledge of the UI.

---

## State

The graph uses a typed `OverallState` with custom reducers for parallel-safe accumulation — search queries, documents, summaries, and claims are all appended concurrently across parallel branches without race conditions.

---

## Stack

- **LangGraph** — graph compilation and state management
- **FastAPI** — task submission and SSE streaming endpoints
- **Redis** — worker queue and task state storage
- **Docker** — containerised for standalone deployment

---

## Repositories

[research-agent on GitHub](https://github.com/JVarnica/research-agent)

---

## Related Blog Posts

- [Making a Research Agent using LangGraph]({{ site.baseurl }}{% post_url 2026-03-24-Research-agent %})
- [Incrementally Building Research Agents]({{ site.baseurl }}{% post_url 2026-05-27-Building-research-agent %})
- [Web Search Tool]({{ site.baseurl }}{% post_url 2026-04-20-agentic-search %})
