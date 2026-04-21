---
layout: post
title: "How I made a vllm docker inference server"
date: 2026-03-24
categories: [backend, vLLM, auth, inference, agent]
tags: [docker, vllm, qdrant, redis, sql, jwt]
project: execuchat
---

# How I made a vLLM docker inference server

### Architecture
```
                                        
                                        chat/python
                                        auth
                                        redis
Android Client --->  API Gateway ---->  aiosqlite
                                        Qdrant
                                        vllm

 
```
### Introduction

Now (March 2026) there are many open sourced models and different inference engines you just need a gpu, intelligence is not closed. I have a 5060ti gpu with 16gb RAM so can run a model on it, the logic of why vLLM is used and what models have been used is [here][inference-md]. Essentially the model used is Qwen3-8B-NVFP4 it has thinking, it's quick with new 4-bit quantization method and uses 6gb RAM so now fits. Any model on huggingface can be picked just need to be careful with quantization, for example as I am using a blackwell gpu can use nvfp4 but this would not work on any amd gpu.

The goal was for small scale use, so my family on the same WIFI network could use this app. This means the system needs to have identification, security and concurrency measures with the chatbot features: chat (streaming), persistent memory, rag, search and image generation. The app is structured as a series of endpoints (chat, save,load), the API is created using FastAPI as its quick and can easily setup a microservice architecture. 

The building process was an iteration of issues which were solved as they came up, each one is explained below.

### Problem 1: Concurrency 

Each endpoint is a request so within each asynchronous function variables can be initialized such as model, temperature, message, enable_rag and enable_search, then they are lost when the request is sent to vLLM. This is good so FastAPI will give each user its own local variables so there is no data mix up, however for information which needs to stay across requests normal python objects can't be used. For example, the message list would need to be created on global, then each request (user) an async write/read the dict will get completely corrupted can't deal with asynchronous operations. You can lock it but then it would be slow as each user would have to wait until the task is done, if you add workers to speed up it doesn't work as can't share memory. 

Redis is therefore used as the cache layer, because each structure is independent so each user has their own key/session which can be accessed super quickly. It handles the concurrency for you, for example you can rpush to a list, each lists is identified by its key (session_id), so data will go to the correct list. This is the wait approach a request to vLLM is sent, as the answer is streaming to client, a local variable is also stored which accumulates the chunks so can be added to Redis list on end. 

### Problem 2: Context Bloat

As vLLM is stateless it has no memory between requests, so you feed it the full conversation history this will quickly bloat up the context when chatting. So first solution is to stop giving the model the full conversation just the latest N messages, the longer N the more context is kept (opted for 6). Furthermore, if the prompt is complicated it will require the model to think which takes up a lot of tokens, and are of less value than the actual response in future turns so can be removed from the response. This saves significant context as thinking is longer than reply, but to not loose all context from thinking a summary of it is kept instead. This summary is created from the following system prompt the summary tags are created for indexing purposes.
```
SYSTEM_PROMPT = """
 You are a helpful AI assistant.

For complex questions requiring analysis, wrap your reasoning in <think>...</think> tags before 
responding. For simple/factual questions, respond directly without thinking.

When you do use <think>, end your reasoning with a one-line <summary>...</summary> tag capturing
your key conclusion, placed just before </think>."""
```

The messages are appended to a Redis list each turn as explained previously, Redis List has lrange so can take the latest N messages, don't need to remove earlier messages from the list. Messages are stored as pairs, so a user & assistant message is a pair, this is because in rag both the query and answer are embedded, so can use same list for rag and chat context. This pair is a dict inside a Redis list so new pairs are appended to the list.
```
#save.py
pairs_key = f"session:{session_id}:pairs"
pair_obj = {
        "pair_index": pair_index,
        "user_text": user_text,
        "assistant_text": assistant_text,
        "created_at": now,
    }

await redis_pool.rpush(pairs_key, json.dumps(pair_obj))
#chat.py
raw = await redis_pool.lrange(pairs_key, -MAX_CONTEXT_PAIRS, -1)
```
Each user has their own session and list (pairs key), which is indexed by session_id. so now Each user will have their own session_id so can find the pairs key, and meta key which is created upon session start with session_id, and stores user_id. The user_id within each request is then compared to one stored if correct, then can access pairs key. 

### Problem 3: Users 

So to have multiple users there needs to be authentication so can identify each user, this is done using OAuth with jwt tokens, it allows identification on each request to the different services. But for this identification to happen all requests must first pass through a gateway's auth middleware which verifies the token and adds user_id to the request. 
```
# login
access_token = create_token({"sub": username, "type": "access"}, timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))

# gateway
payload = verify_jwt(token)
request.state.user_id = payload.get("sub")
```
User details are created on registration using passlib's CrypContext, if the hashed password is verfied login is successful, the access and refresh tokens are then created. There won't be much concurrency for this process you do it once on registration and login, then tokens are used so a simple sqlite file can be used.

Now on app start up a live session needs to be created for vLLM to have context, the session is a hash set with user_id and pair count, named with session_id for indexing these local variables created in the create_session endpoint. 

```
await redis_pool.hset(f"session:{session_id}:meta",
        mapping={
            "user_id": user_id,
            "pair_count": 0,
            "created_at": now,
            "updated_at": now
        },
    )
```
Now that user.id is stored each request to Redis will first compare if same user using the meta key (authentication), then will append or get from list. Could store user_id in each pair object but was simpler to just use a hash set, you just find the session meta key in Redis connection pool. If it's not there then no session is active for the user. 

### Problem 4: Persistent Memory

Currently have live memory but on close the chats need to persist so a database is used. As its small scale can use sqlite just need the asynchronous version (aiosqlite) so reads and writes happen in background and don't block main thread. Two different data snippets are needed one which has conversation id & titles which is for chat list so can load the selected chat, the other is the messages themselves. As its a relational database, filtering is easy so all messages for all users are stored in the same database, just filter for user and convo_id then fill the messages in order using the pair_index. 
```
        CREATE TABLE IF NOT EXISTS messages (
            msg_id      INTEGER PRIMARY KEY AUTOINCREMENT,
            convo_id    TEXT NOT NULL,
            user_id     TEXT NOT NULL,
            turn_index  INTEGER NOT NULL,
            role        TEXT NOT NULL CHECK(role IN ('user','assistant')),
            content     TEXT NOT NULL,
            created_at  INTEGER NOT NULL,
            FOREIGN KEY(convo_id) REFERENCES conversations(convo_id) ON DELETE CASCADE
        );
```
Memory is essentially searching, finding the most relevant chunk of text from database to add to the prompt. You cannot search the text space for semantic meaning just exact word, so a vector representation of the text is created for semantic search. These embeddings need to be stored in a persistant storage, Qdrant is the choice: as don't need to set up the hnsw graqph (indexing) myself unlike pgvector, but Weaviate and Chroma would work perfectly aswell.

In fact as the scale is small there is no indexing needed as its just saved past chats, so a linear brute search is fine. This is also because Qdrant has hybrid search, so can filter for user_id. 
```
await client.create_collection(
            collection_name=QDRANT_COLLECTION,
            vectors_config=qmodels.VectorParams(
                size=EMBEDDING_DIM, 
                distance=qmodels.Distance.COSINE,
                on_disk=True
            ),
            hnsw_config=qmodels.HnswConfig(m=0) #brute force 
        )
```

### Problem 5: How to make embeddings

Upon save the chat messages are inserted to a sqlite database, but the embeddings still need to be created then flushed to qdrant, so until this sequence is done the user is blocked. So the embeddings are done in the background so on save they only have to be flushed to qdrant. 

Each pair has an index so overlapping window chunking can be used, so text for first pair is embedded, then it queries redis for previous pair then embeds this concatenated text for another chunk. This makes semantic search easier as a single turn doesnt have as much context as 2, but if i used 3 pairs then would start to generalize why you need to keep embedding a single turn aswell. 
```
        prev_raw = await redis_pool.lindex(pairs_key, -2)
        if prev_raw:
            try: 
                prev = json.loads(prev_raw)
                text = _format_chunk_text(
                    [
                        (prev.get("user_text", ""), prev.get("assistant_text", "")),
                        (user_text, assistant_text),
                    ]
                )
                await enqueue_chunk_embedding(request=request,
                session_id=session_id,
                    user_id=user_id,
                    start_turn=pair_index - 1,
                    end_turn=pair_index,
                    chunk_type="pair_2",
                    text=text,
                    created_at=now,
                )
```

Each chunk has its own chunk id which acts as a deterministic key, it is created using the session_id and chunk_type so either pair 1 or 2, with its position so start turn and end_turn.  So the 7th pair might have chunk_id of abc123pair177 the overlapped one abc123pair267, so a different pair has a different id but the same pair will always make same id, so no issue if enqueued twice just over written. 
```
def _chunk_id(session_id: str, start_turn: int, end_turn: int, chunk_type: str) -> str:
    # deterministic to make retries idempotent
    return f"{session_id}:{chunk_type}:{start_turn}:{end_turn}"
```

So each chunk has its own hash set named using chunk_id it stores a few fields with embedded being zero. The advanatge of a Redis hash is its dynamic so can add another field and mofify one without getting the full hash. Therefore a Redis stream is used so job can be queued/ added to the stream, then a worker takes the chunks asynchronosuly so can have multiple workers embedding text at once. Once a vector is created the embedded field is changed to 1 signifying success and the vector list is added to a new field embedding_json.
```
 await r.hset(
                        payload_key,
                        mapping={
                            "embedded": 1,
                            "embedding_json": json.dumps(vec_list),
                        },
                    )
```
This embedding is done on app start as soon as a job is in the stream so can happen in background so no UI blocking occurs. 

### Problem 6: External Context & Search

Most of the time when chatting current context is needed but models have a knowledge cutoff, for example the model currently used qwen3-8B has a cutoff of april 2024.  This means adding context from the web to the model so it has up to date information, makes a huge difference in the quality of the answers. What is important is how its done though as a tool or as context injection. 

When having search as a tool the model would decide itself to search based off the query, this is more complicated as model might hallucinate it has searched. Therefore I opted to add search as context, same with the text chunks recieved from Qdrant. 

The search happens using a seperate SearxXNG container which does all the heavy lifting and returns json results, which is limited to 5 responses of 240 characters currently to limit context bloat. 
```
url = f"{SEARXNG_INTERNAL_URL.rstrip('/')}/search"
r = await http_client.get(url, params={"q": query, "format": "json"})
```
For rag the vector similarity search is filtered by users and only the top 6 chunks are added to the context. 
```
hits = await qdrant.search(
        collection_name=QDRANT_COLLECTION,
        query_vector=query_vec,
        limit=top_k,
        with_payload=True,
        with_vectors=False,
        query_filter={
            "must": [
                {"key": "user_id", "match": {"value": user_id}},
            ]
        },
```
So now before each message the latest pairs are recieved from the Redis List in get_session_pair method, which also acts as authentication to see if there is a session and if user is correct. Once live memory is recieved, there might now be rag and search context which also need to be added to the prompt. This logic is sorted in the build_context method, where rag and search context are given the system role and pairs are formatted to include user and assistant prefix before. 
```
 messages = [{"role": "system", "content": SYSTEM_PROMPT}] 
    if rag_context: 
        messages.append({"role": "system", "content": rag_context}) 
    if search_context: 
        messages.append({"role": "system", "content": search_context}) 
    for pair in pairs[-MAX_CONTEXT_PAIRS:]: 
        messages.append({"role": "user", "content": pair.get("user_text", "")}) 
        condensed, _ = condense_assistant(pair.get("assistant_text", "")) 
        messages.append({"role": "assistant", "content": condensed}) 
    messages.append({"role": "user", "content": current_message}) 
```
In theory this worked well but in practise it did not, had to improve the search mechanism. 

### Problem 7 : Why search not good yet & context improvement

Context is all per request currently. The injected rag and search context are added in build_prompt but then they are not stored anywhere.Thus a redis list is used to store this context for rag its useful as turn 1 you have the query it gets most similar chunks then this text is stored in redis so can be accessed every turn of that convo. This is useful as can switch topics and still have the information on topic one, when new query would not embed as lot of noise in prompt after turns. Currently this has worked out fine for retrieved context from conversations, but not for search. 

When search context is injected as a system message the model sees it and can answer the first question, but the next ones not aswell compared to when search is added as a tool call (more on this later). It wasn't coherent as has the search context but no clue why it has searched, so leaving toggle on and trying to converse would always search so couldn't really have multiple turn conversations. This is probably because of prompting aswell so added a search grounding prompt telling it how to use the search data and this improved it considerably.

#### SearXNG is insufficient 

Furthermore before this though searXNG is very good meta search engine so will give you urls for your queries but the content is not even a summary basically just says the title. This is an issue as the model doesn't have enough information to work from, can be seen in the figure below. Figure 2 asks it to answer based on current knowleadge then I use search but as you can see the model doesn't even use the search results (why i made grounding prompt and date in system prompt)

![alt text](/images/bad_results_sr.jpg)
![alt text](/images/Screenshot_20260408_113410_Execu_Chat.jpg)

Okay so not enough information from content, how much information does it give lets check here the query was, "what were the round 16 champions league results" and here is the full output. As you can see below it doesn't even tell you the results basically its content field which we are getting is just showing the title. At least you get 10 results so model can get a jist of results (fig 2) but its not a good answer.  
```
  "url": "https://www.nbcsports.com/soccer/news/uefa-champions-league-schedule-knockout-round-fixtures-path",
  "title": "UEFA Champions League knockout phase schedule: Fixtures, dates, kick off times, full details - NBC Sports",
  "content": "And then there were only eight teams left in the 2025-26 UEFA Champions League following the completion of the round of 16. MORE — UEFA Champions League league phase final table Yes, we're into the thick of it now, but only two Premier League teams are still in the race for the European Cup after four were bounced in the last 16.",
  "engine": "brave",
  "parsed_url": [
    "https",
    "www.nbcsports.com",
    "/soccer/news/uefa-champions-league-schedule-knockout-round-fixtures-path",
    "",
    "",
    ""
  ],
  "engines": [
    "startpage",
    "duckduckgo",
    "brave"
  ],
  "score": 4.0,
  "category": "general"
}
```

So the solution is to use a search engine not a html parser such as beautful soup, so trafilatura was used from the links given by searxng. Here is the result from trafilatura (only a part):
```
 Trafilatura extraction (3367 chars):
  And then there were only eight teams left in the 2025-26 UEFA Champions League following
  the completion of the round of 16.
  MORE — UEFA Champions League league phase final table
  Yes, we’re into the thick of it now, but only two Premier League teams are still in the
  race for the European Cup after four were bounced in the last 16. Those teams are
  Arsenal and Liverpool, and they can’t meet until the final on May 31 in Budapest,
  Hungary.
  Chelsea were thrashed and hammered by defending champs PSG, 8-2. Manchester City were
  battered by Real Madrid, 5-1. Newcastle kept Barcelona close for 135 minutes, but the
  tie ended 8-3 at Camp Nou. Interestingly enough, Tottenham Hotspur’s two-goal defeat
  (7-5) to Atletico Madrid was the most respectable scoreline of the lot.
  Full UEFA Champions League knockout phase fixtures
  UEFA Champions League quarterfinals
  First legs
  Tuesday, April 7
  Real Madrid 1-2 Bayern Munich — Recap, video highlights
  Sporting Lisbon 0-1 Arsenal — Recap, video highlights
  Wednesday, April 8
  PSG 2-0 Liverpool — Recap, video highlights
  Barcelona 0-2 Atletico Madrid — Recap, video highlights
  Second legs
  Tuesday, April 14........
```
As you can see there is so much more information when using trafilatura, that you only need to scrape a few searches from searxng and pay attention to context bloat cannot give it full content, so each url has maximum of 2000 chars for 10,000 max char context this is just ruff but so has enough information to make decisions.

This made a huge difference with the grounding prompt now the model has more data so it can answer for complex questions and give much more detail, however it was still struggling with multi-turn conversation. It would just search every prompt into a query, as you can imagine this is an issue as with conversations you have connecting sentences which would make no sense to search such as, "yes thats true thanks for the information." Thus, the solution is to tell the model when it needs to search so it becomes agentic, the way it is done is through a tool schema and a loop so can evaluate on first call whether to call the tool based on the prompt. Making search agentic by having the model decide whether to call the search tool solves this issue well. More info here [/_posts/2026-04-20-agentic-search]


### Conclusion  

You can call vllm easily and very quickly, but for a working chatbot there are a lot of things you need to take into account such as how are the embeddings created for rag, what vector store, what database to use, what search engine, authentication for users etc. So there are a lot of things to think about, this blog has explained my main struggles and how I went to solve each challenge to create a working chatbot, there are a lot more details and improvements to do still. For example one would be agentic rag in a project settings, so you give it documents and it decides whether to access the documents based on the query, like for search.

Making a single user app compared to multiple users even if small scale is a major difference, before it was just call vllm and store conversations on phone file, not possible for users needs to be server side. As the structure is API based I went with jwt for authentication, so gateway verifies the jwt_token and extracts user_id to send on each request. But session authentication would have worked fine as well because after login we create the jwt tokens currently but instead it could store the session_ID in redis and send this cookie on each request. But it would also need a lookup on every request whereas jwt tokens no lookup needed and no cookie so stateless gateway. Sessions are used here but are per-chat per user, a hash set used to say if a conversation exists, how large it is, and whether this authenticated user owns this conversation so can access the pairs list. 

The compute is also very important at the moment it is very constrained as just a small consumer GPU is used (5060ti) which has 16gb VRAM, this isn't enough to use a larger model than Qwen3-8B (6gb NVFP4) as need kv_cache space currently it is quantized to fp8 so have room for 110,000 tokens (7.6gb) enough for 6 plus concurrent users at a context length of 16,384 tokens. Luckily this model has been trained with tool use so can instead try and make it more agentic in the future. 
