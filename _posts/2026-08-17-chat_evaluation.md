---
layout: post
title: "Evaluating deployed Chatbot: Tool usage & Chat Endpoint"
date: 2026-08-17
categories: [backend, vLLM, Qwen, evaluation]
tags: [evaluation, vllm, searxng, tools]
project: execuchat
---

## Evaluating deployed Chatbot: Tool usage & Chat Endpoint


The chatbot has been successfully deployed onto the phone, this indicates the server and Android app work as expected. The system now needs to be evaluated, so its performance can be understood better; and whether there are faults & bugs. So there are quite a few components which need to be checked for the gateway.

    - gateway: 1) Tool routing, are the calculator and web search tools called when needed. What percentage of time are they not?
               2) Chat programmic test, asks a query to real chat endpoint to mock an user, this tests the programmatic nature of an answer. For example for maths check exact correct answer, etc. Other checks are entity coverage with alias groups, tool chains, and latency. 
               3) Now the answers are needing to be judged for semantic correctness, this will be done using a larger model.


### Routing Evaluation

To test whether the tools were called reliably created a 62 questions set [routing_questions_v4](https://github.com/JVarnica/vllm-server/blob/master/gateway/evals/routing/routing_questions_v4.jsonl), each questions has a stratum which are different question types. Clearly Current, for recent questions which are not in model's training, so the model must search to get the required context. Then for general knowledge you don't want it to search should have been trained on this. Next there is conversational, multi-entity, temporally ambiguous, adversial. For calculator calc basic and calc word, to see if it understands to call calculator when asking it with words and with numbers.

```
{"id": "cur-01", "stratum": "clearly_current", "q": "What were the Champions League quarter-final results?", "min_search": 1, "min_calc": 0}
{"id": "gen-10", "stratum": "general_knowledge", "q": "Summarise the plot of Hamlet.", "min_search": 0, "min_calc": 0}
{"id": "multi-05", "stratum": "multi_entity", "q": "Compare current UK and German inflation rates.", "min_search": 1, "min_calc": 0, "entities": ["UK", "German"]}
{"id": "amb-01", "stratum": "temporally_ambiguous", "q": "Who is the CEO of Intel?", "min_search": 1, "min_calc": 0}
...
```

Each question was repeated 10 times, 620 overall calls.

rep 0: 100.0%                               
rep 1: 98.4%
rep 2: 96.8%
rep 3: 100.0%
rep 4: 100.0%
rep 5: 100.0%
rep 6: 98.4%
rep 7: 100.0%
rep 8: 98.4%
rep 9: 98.4%

=== prod_parity3  (620 calls, 62 items, 10 reps) ===
overall accuracy: 99.0%    latency: p50 16885ms  p90 29528ms  p99 37501ms

stratum                   acc  over_s  under_s  over_c  under_c     n
adversarial            100.0%       0        0       0        0    50
calc_basic             100.0%       0        0       0        0    70
calc_trap_units         93.3%       0        0       0        2    30
calc_trivial           100.0%       0        0       0        0    30
calc_unsupported       100.0%       0        0       0        0    40
calc_word               98.0%       0        0       0        1    50
clearly_current        100.0%       0        0       0        0    80
conversational         100.0%       0        0       0        0    60
general_knowledge      100.0%       0        0       0        0    90
multi_entity           100.0%       0        0       0        0    50
temporally_ambiguous    95.7%       0        3       0        0    70

Results on github (https://github.com/JVarnica/vllm-server/blob/master/gateway/evals/runs/prod_parity3.jsonl)

Can clearly see the model calls the tool when required, for search it searched on all questions except one amb-04. This is actually not an error just a bad question on my end, the question is "whether Qwen3 supports tool calling", and as I am asking about itself it won't search just uses its own knowledge. Same for calculator can use its pre-trained knowledge to answer the sine of 30 degrees question. 

Overall can say the system prompt, and tool description is correctly describing when the model should call the tools; hence the pipeline has passed the first evaluation. Next is a programmatic evaluation to see quick errors on certain scenarios without needing to call a judge, such as will it chain the web_search and calculator tool if needed. 

### Deterministic Test

Routing only says whether the right tool was used, doesn't measure the answer itself. This test is a cheap first pass, with programmatic checks. The chat endpoint is called 5 times for each question (215 runs), each question has its own expected behavior depending on its stratum. 

The scorer checks different things for each stratum:

    - exact answer, the exact answer is needed for maths questions so it passes. Null for strings.
    - entities/sub string, does the answer contain the correct entity from the question. 
    - graceful, made up questions, model needs to say it cannot answer.
    - trajectory, are the tool calls in the correct order if called. 

Accuracy is not really accuracy it's just whether the question has passed these tests. Can see that chained questions only have the correct trajectory 66% of the time. For example for ch-01 which asks to 'search for Mount Everest height and convert it to feet', one time the calculator was never called so got 4/5 right. Another one ch-02, is about China's great wall length and convert it to miles, the calculator didn't call so got wrong number. So only 3/5 this is the model unable to decide what number to use as reference sometimes.

It has also not been able to answer question gf-03, got every single repetition wrong without an answer. The error was a RemoteProtocolError: peer closed connection without sending complete message body, meaning the searches were crap with too much input tokens so reached max context of 8196. For ta-04 'how many member states does the European Union have' was correct 3/5 times as had to include the number 27 in the answer. 

```
 python gateway/evals/score_answer.py gateway/evals/runs/answer_ref_rescored.jsonl

stratum                      acc     n
--------------------------------------
calc_basic                100.0%    20
calc_trap_units           100.0%    15
calc_word                  95.0%    20
chained                    66.7%    30
clearly_current           100.0%    30
conversational            100.0%    10
general_knowledge         100.0%    25
graceful_failure           75.0%    20
multi_entity               96.0%    25
temporally_ambiguous       85.0%    20
--------------------------------------
TOTAL (programmatic)       90.7%   215

-- flaky / failing items --
  ch-01 [chained] 4/5
      trajectory: never called calculate
  ch-02 [chained] 3/5  UNSTABLE answers=[13170.62, 2012.0, 13170.62, 2119.69, 13170.62]
      exact_answer: none of [13170.7] in answer numbers [2012.0, 2012.0]
      trajectory: never called calculate
  ch-03 [chained] 3/5
      trajectory: never called calculate
  ch-04 [chained] 2/5  UNSTABLE answers=[473.0, 472.857, 828.0, 473.14, 473.14286]
      trajectory: never called calculate
  ch-05 [chained] 4/5
      trajectory: repeated call web_search {"query": "average distance from Earth to Moon in km"}
  ch-06 [chained] 4/5
      trajectory: never called calculate
  cw-04 [calc_word] 4/5
      exact_answer: none of [42739.2, 42739.0] in answer numbers []
      error: empty final answer
  gf-03 [graceful_failure] 0/5
      graceful: no admission of failure
      error: RemoteProtocolError: peer closed connection without sending complete message body (incomplete chunked read)
  me-01 [multi_entity] 4/5
      substrings: missing ['2007', '2008']
  ta-02 [temporally_ambiguous] 4/5
      substrings: missing ['Burj Khalifa']
  ta-04 [temporally_ambiguous] 3/5
      substrings: missing ['27']
```

More than 90% of the model's answers pass the test, the worst performing stratum was chained questions were the model sometimes decided to mentally calculate the answers. This isn't always an issue as answered correctly but more likely to get the calculation wrong, when using the calculator this is not a problem. 
One multi-entity question was incorrect didn't have the release date 2007 and 2008 for iphones and android phones. For the other me questions it correctly mentions all the requested entities, this doesn't mean the model answered though as for current CEOs for Google and Microsoft the model itself says doesn't have the neccessary information. Thus, this test is just a baseline as you would otherwise intuitevely think the model actually answered. This happens when Searxng closes an engine for timeout, once this happened to google_cse for timeout the results become junk due to keyword matching. Hence ta-02 and ta-04 erorrs can be accounted for by this, as was told to answer from search but couldn't even if it knew the answer. This was understood when trying to get a frozen set, so had to do a 15 seconds timer between questions otherwise rate-limiting kicked in sometimes.
```
{"id": "me-02", "stratum": "multi_entity", "question": "Who are the current CEOs of Microsoft and Google?", "rep": 1, "run": "reference", "model": null, "expected": {"entities": [["Microsoft"], ["Google", "Alphabet"]], "needs_judge": true}, "answer": "The web search results provided do not contain information about the current CEOs of Microsoft or Google. The links retrieved are related to \"Current\" (a fintech banking app, its support, or dictionary definitions of the word \"current\"). There is no mention of Microsoft's CEO (e.g., Satya Nadella) or Google's CEO (e.g., Sundar Pichai) in the results. For accurate and up-to-date information, further verification from reliable sources is recommended.", "turns": [{"tool_calls": [{"tool": "web_search", "args": {"query": "current CEOs of Microsoft and Google"}}], "tool_results": []}], "n_iterations": 1, "n_tool_calls": 1, "usage": [{"phase": "final", "prompt_tokens": 732, "total_tokens": 1063, "completion_tokens": 331}], "latency_ms": 34303.0, "trace_id": "d8201667eb97b87e23637a3998b863a0", "error": null, "_score": {"checks": {"exact_answer": {"passed": null, "detail": ""}, "substrings": {"passed": null, "detail": ""}, "entities": {"passed": true, "detail": "all covered"}, "trajectory": {"passed": null, "detail": ""}, "graceful": {"passed": null, "detail": ""}, "hygiene": {"passed": true, "detail": "clean"}}, "prog_passed": true, "prog_failures": [], "n_checks_applicable": 2, "needs_judge": true, "final_verdict": null}}
```

Now that is done a judge model will now have to be used, to actually judge whether these answers were of any use. A frozen baseline question set is used, as now each repetition sees the same context, any variation is based on the model this allows us to see the reliability of the model. 

### Judge Test

One thing to note is that these tests only evaluate this system, it is not comparable to other benchmarks at all. The evaluation is still useful as it will show what scenarios the system fails at, so it doesn't do so in production. Also the score is useful with the frozen set so can be compared to other vllm models in the future

The frozen set created based off the question set [here](https://github.com/JVarnica/vllm-server/blob/master/gateway/evals/questions_searchv1.jsonl), most questions are similar with the calculation tool removed as the failure modes were identified. More clearly current and multi-entity questions were added. Each question will have required facts based off the search results, which the judge will use to judge the model's answers. The frozen set freeze_setv4.jsonl stores tool content not raw search results, as its smaller so more concurrency and no variations from retrieval pipeline. For the experiment there are 10 repeats of each question for a total of 300 calls [answer-v2](https://github.com/JVarnica/vllm-server/blob/master/gateway/evals/runs/answer-v2.jsonl).

Kept using vllm instead of an API as its free but this limited us to use Qwen3-14B-NVFP4, the bigger version of 8B so should have enough reasoning capabilities to act as a judge. Next was the rubric, the overall score was mainly based off correctness and writing, slightly longer than shown below and mentioned to not downscore rows which have more information than the required facts, if no contradictions.

```
CORRECTNESS (1-5)
5 - Conclusion agrees with the required facts; all relevant facts present or implied;
    no contradictions.
4 - Conclusion correct, with only a minor omission or imprecision. 
3 - Meaningful correct content, but an important required fact is missing, unclear, or
    contradicted; the conclusion is ambiguous or only partly correct.
2 - Conclusion contradicts a required fact or answers the central question wrongly.
1 - Fundamentally wrong, unusable, or fails to answer.

WRITING (1-5)
5 - Clear, direct, well organized, appropriately detailed.
4 - Clear, with minor verbosity, repetition, or awkwardness.
3 - Understandable but verbose, repetitive, or poorly organized.
2 - Hard to follow or badly structured.
1 - Unusable.
```

The results of the judge are shown below, the mean overall score and correctness score are 4.3.  There are 29 samples scored 1 and 10 samples scored 2, so 39 samples have clearly failed. But upon investigation a lot of these are from judge failures. For example cc-02 has a correctness score of 2, each repitition states the required fact verbatim but has a 1.72 stdev between answers. This is the same for sp-07, which is about reigning formula 1 champion has score of 2.3 instead of 5. Thus, a 14B model without reasoning is not a reliable judge,maybe this would not be the case with thinking on (for concurrency reasons its off).

An inaccurate representation of the data would have been taken if had not used Opus 4.1 as the final judge. It has scored the mean overall score as 4.64, has improved score of 5 to 254 samples instead of 211. On other questions the score has gone down such as, for cc-10 it's a where and  when question and nine reps omit the month. So it is rescored to 4.1 from 4.8.

<p>300 rows · 30 unique questions</p>

<table>
  <tr>
    <th></th>
    <th colspan="3" style="text-align: center; border-bottom: 2px solid #000;">Judge 14B</th>
    <th colspan="3" style="text-align: center; border-bottom: 2px solid #000;">Ous 4.1 Judge</th>
  </tr>
  <tr>
    <th>Metric</th>
    <th>Correctness</th>
    <th>Writing</th>
    <th>Overall Score</th>
    <th>Correctness</th>
    <th>Writing</th>
    <th>Overall Score</th>
  </tr>
  <tr>
    <td>Mean</td>
    <td>4.3579</td>
    <td>4.6589</td>
    <td>4.3043</td>
    <td>4.6367</td>
    <td>4.7867</td>
    <td>4.63</td>
  </tr>
  <tr>
    <td>Std dev</td>
    <td>1.2941</td>
    <td>0.6574</td>
    <td>1.2897</td>
    <td>1.035</td>
    <td>0.5726</td>
    <td>1.0359</td>
  </tr>
  <tr>
    <td>Min / Max</td>
    <td>1 / 5</td>
    <td>3 / 5</td>
    <td>1 / 5</td>
    <td>1 / 5</td>
    <td>3 / 5</td>
    <td>1 / 5</td>
  </tr>
  <tr>
    <td>Score 1</td>
    <td>29</td>
    <td>0</td>
    <td>29</td>
    <td>21</td>
    <td>0</td>
    <td>21</td>
  </tr>
  <tr>
    <td>Score 2</td>
    <td>10</td>
    <td>0</td>
    <td>10</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
  </tr>
  <tr>
    <td>Score 3</td>
    <td>12</td>
    <td>31</td>
    <td>13</td>
    <td>0</td>
    <td>24</td>
    <td>0</td>
  </tr>
  <tr>
    <td>Score 4</td>
    <td>22</td>
    <td>40</td>
    <td>36</td>
    <td>25</td>
    <td>16</td>
    <td>27</td>
  </tr>
  <tr>
    <td>Score 5</td>
    <td>226</td>
    <td>228</td>
    <td>211</td>
    <td>254</td>
    <td>260</td>
    <td>252</td>
  </tr>
  <tr>
    <td>Pass ≥ 4</td>
    <td>82.94%</td>
    <td>89.63%</td>
    <td>82.61%</td>
    <td>93.0%</td>
    <td>92.0%</td>
    <td>93.0%</td>
  </tr>
  <tr>
    <td>Perfect</td>
    <td>75.59%</td>
    <td>76.25%</td>
    <td>70.57%</td>
    <td>84.67%</td>
    <td>86.67%</td>
    <td>84.0%</td>
  </tr>
</table>



### Discussion 

It can be said the model is reliable, as when it answered correctly the question it did so for all 10 repetitions. This also means when it answered a one question poorly it did so on most reps, this was the case for sp-05 and ta-07 (Current undisputed heavyweight champion & top grossing film all time). The question about boxing champion is a trick question to test reasoning, because Usyk vacated his heavyweight belts recently. In September 2026 there is no undisputed heavyweight champion, and the model fails to say so in any repitions, even with the answer in the context. 

This error happenned as most evidence urls given confirm Usyk is heavyweight champ and one says otherwise that he has given then up, but no where does it say there is no heavyweight champion, the model needs to derive the negative conclusion. A more intelligent model catches this difference between undisputed champion and champion, as others hold some belts, but none all. For top grossing film on the other hand the answer is clearly in context "the third-biggest release ever globally behind 2009's Avatar ($2.9 billion) and Avengers: Endgame ($2.79 billion)." The model quotes it in an answer but then concludes SpiderMan is number 1. It is also the answer which takes the longest, an average of 55.39 seconds and longest rep was 74.66 seconds. So the reasoning doesn't help at all. 

On easier questions where no reasoning is needed, as the answer is in the context, the model does well it highlights the correct fact. This is seen when asking for PM of UK, the model has been trained on probably Keir Starmer, but always correctly answers Andy Burnham. Thus, this pipeline can overturn a prior conflict which is a very important to know.

Overall the model works decently well you can converse with it about new things, can calculate using the calculator, and these can be chained. However the 8B model does have reasoning faults, upgrading to the 14B wouldn'thelp too much. Most likely a more recent model of this size might not have trouble with this question, this is the next test to use another model family. 






