---
name: arch-knowledge-search
description: Set up, tune, test and debug knowledge bases (SearchAI) used by ABL agents on the Kore.ai Agent Platform, including ingestion and chunking, search tool bindings, retrieval settings (max_search_tokens, Top K, similarity threshold, keyword weight, object-aware retrieval), permissions, citations, the automatic first search, retrieval evals and wrong or missing knowledge-base answers. Use when setting up a knowledge base, or when a knowledge answer is wrong, missing, slow or expensive.
---

# Knowledge and search

A wrong knowledge-base answer has one of four causes: the document never got indexed, the search
did not return the right chunk, the chunk was cut by a budget, or the model ignored it. Find which
one from the evidence before changing anything.

## How an agent uses a knowledge base

- Each knowledge base is bound as a project tool, `search_kb_<name>(query)` with `type: searchai`.
  The binding can carry `search_instructions`, `retrieval_profile` and the release-dependent
  `kb_fast_path`. An agent file can set `search_instructions`, `compaction:` and `kb_fast_path` for the tool (its
  `kb_fast_path` choice wins); its `search_instructions` is ignored when the project tool already
  has some.
- The model picks between knowledge bases by their descriptions, so write a specific one for each.
- Search parameters (`queryType`, `filters`, `topK`, `similarityThreshold`) can be declared in the
  tool signature; the runtime may also add query type and filters when search instructions call for
  them. Filters the model chooses are kept only when grounded in the user's words, and `filters`
  must be an array. A filter that must always apply belongs in the tool's enforced inputs.
- **Citations:** when on, the model cites with `[n]` and text channels add a "Sources:" list. Citations are set per tool in current
  releases (in 1.7.3, one knowledge base with citations off turned them off for all).
- **Automatic first search.** For an agent whose only business tool is one knowledge base (not a
  router or orchestrator, no handoff or delegate tools, no GATHER fields with inline gather), the
  platform can search before the first model call and save a model round. In release 1.7.3 it is
  always on for agents that qualify, skips the tool's hooks and `MODEL_CONTEXT`, and shows the model
  only the last 5 earlier messages. Later releases make it opt-in per tool with `kb_fast_path: on`,
  run it as a normal tool call, and exclude tools with confirmation or `on_result`/`on_error` actions. Check which release the project runs.

## Ingestion checklist

1. Choose sources: manual text, web, database, API or SharePoint (`platform_knowledge_bases`
   `create_source`). Creating a web source does not start a crawl; start the crawl in Studio (the MCP tools cannot
   start one), then check `list_documents`.
2. Choose chunking: token-based (fixed, semantic, sliding window), markdown-aware or page-based.
   A new index defaults to page-based chunking (heading and section aware); token-based pipelines
   default to 1,000 tokens with 200 overlap. Re-ingest after changing it.
3. After every import or upload, check document status and knowledge-base health
   (`list_documents`, `health`, `activity`) until every document is indexed. A successful
   `add_text` can still end in a processing failure, and an imported project can arrive with an
   empty knowledge base.
4. Prove retrieval with a real question, not just an indexed document count.
5. Document changes apply on the next search with no publish. Binding changes need a new
   conversation, and publish and deploy for channels.
6. Ingestion pipelines (`platform_ingestion_pipelines`): `validate` and `test_selection` before
   `publish`; `reindex` and `publish` take minutes and need confirmation.

## Retrieval settings

| Setting                                  | Default                                        | Effect and cost                                                                                                                                                                         |
| ---------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `max_search_tokens` (tool `compaction:`) | No cap up to 10 chunks; 40,000 tokens above 10 | Whole chunks are dropped from the end, with no marker in the prompt. Set it on every KB tool; 2,000–4,000 for voice                                                                     |
| Top K                                    | 10 (1–100)                                     | Fewer chunks means a smaller prompt and less noise                                                                                                                                      |
| Similarity threshold                     | 0.2 (0–1)                                      | Raise it to drop weak matches                                                                                                                                                           |
| Keyword weight                           | 0.5                                            | Raise toward 0.7 for FAQ, product codes and exact terms                                                                                                                                 |
| Search type                              | Hybrid                                         | Reranking runs only for semantic or hybrid search with rerank on                                                                                                                        |
| `search_instructions`                    | None                                           | On the automatic first search (`kb_fast_path: on`) it adds a classify call of up to about 5 s; elsewhere it shapes the prompt and mandatory filters. Use only where it measurably helps |
| `retrieval_profile: object-aware`        | Off                                            | Splits a question into up to 10 categories searched in parallel with reranking. Spell the value exactly; costs extra searches                                                           |
| Parallel tool calls in one turn          | Shared 100,000 characters                      | Each gets about 100,000 ÷ N characters, never less than 800 or more than 40,000                                                                                                         |

## Permissions

Search runs as the user only when the platform knows the user's identity (email); otherwise it is
public and returns only documents visible to everyone. Test permission-sensitive content with a
real customer identity, not only the Studio test chat.

## Debug a wrong or missing answer

Read the session with `debug_traces` (`projectId`, `sessionId`, `source: "historical"`) and
`debug_get_span_tree`:

| Evidence                                                      | What it tells you                                                                                       |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| The knowledge-base tool call event                            | The effective query, filters and query type, result count, search latency, included and omitted results |
| A `decision` event `searchai_kb_token_budget` (where present) | `maxSearchTokens`, included and omitted results, actual tokens                                          |
| A `kb_fast_path` event                                        | Whether the automatic first search ran, if the query was rephrased, results per tool, latency           |

Then decide the layer: not indexed (ingestion), not retrieved (query, filters, Top K, threshold,
keyword weight, chunking), cut (budget), or retrieved but not used (instructions, model). Use the
Studio knowledge-base Search & Test playground to try the exact query.

## Evaluate retrieval

- There is no built-in citation or knowledge-grounding evaluator template (the trajectory `groundedness`
  metric measures tool-call success, not grounding in documents); write custom evaluators for
  "answer is supported by the retrieved text", "cites the right source" and "says it does not know
  when the knowledge base has no answer".
- Build a golden set per knowledge base: common questions, paraphrases, exact-term questions
  (codes, product names), questions whose answer is not in the documents, and permission-restricted
  questions run as different users.
- Watch production signals with `platform_analytics` reports such as hallucination detection and
  knowledge gaps, and turn misses into scenarios (`arch-eval-catalog`).

## Rules

1. Write a specific description for each knowledge base.
2. Set `max_search_tokens` on every knowledge-base tool.
3. Lower Top K and raise the similarity threshold for fewer, better chunks.
4. Raise keyword weight for FAQ and exact-term content.
5. Add `search_instructions` and object-aware retrieval only when an eval shows they help.
6. Keep a knowledge-only agent free of other business tools if you want the automatic first
   search, and know which release you are on.
7. On release 1.7.3, do not rely on hooks or `MODEL_CONTEXT` to shape results on the automatic path.
8. Check document status after every import before binding the tool.
9. Re-ingest after changing chunking.
10. Put required filters in enforced inputs, not model-supplied filters.
11. Test permissions with real identities.

## Output contract

1. **Story:** what the user asked, what was searched, what came back, and where it went wrong.
2. **Retrieval table:** turn | query sent | filters | results returned | included or omitted |
   tokens | latency.
3. **Layer:** ingestion, retrieval, budget, or model use, with quoted evidence.
4. **Fix** as a setting or ABL change, and the eval questions that prove it.
5. **Glossary** and **confidence and gaps** (including which release behavior applies).
