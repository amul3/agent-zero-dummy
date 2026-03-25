# Agent Zero Optimization Playbook (Personal SWE + Research Scientist)

This guide maps Agent Zero's settings to practical operating profiles so you can tune for a balance of performance, efficiency, precision, and power.

## 1) How Agent Zero is wired (important before tuning)

- Four model roles are used: `chat`, `utility`, `browser`, and `embedding`.
- Memory and knowledge retrieval are vector-search based, with auto recall + auto memorization extensions.
- Chat history is compressed/summarized once it crosses the configured history context budget.

## 2) High-impact knobs and what they do

### Models

- **Chat model**: primary coding/reasoning engine.
- **Utility model**: summarization, memory extraction/filtering, memory consolidation assistance.
- **Browser model**: web nav + page understanding (vision helps).
- **Embedding model**: memory/knowledge indexing + retrieval quality.

### Context and compression

- `chat_model_ctx_length`: total token budget available to the chat model.
- `chat_model_ctx_history`: fraction of that total reserved for history before aggressive compression kicks in.

### Memory recall behavior

- `memory_recall_interval`: how often to run retrieval in message loop iterations (lower = more recall, more cost/latency).
- `memory_recall_similarity_threshold`: recall strictness (higher = precision, lower = recall).
- `memory_recall_query_prep`: asks utility model to generate focused retrieval query.
- `memory_recall_post_filter`: asks utility model to prune retrieved memories.

### Memory write behavior

- `memory_memorize_enabled`: controls automatic memory writebacks.
- `memory_memorize_consolidation`: enables LLM-assisted dedupe/merge/replace logic.
- `memory_memorize_replace_threshold`: dedupe aggressiveness in non-consolidation mode.

### Throughput / reliability

- `*_rl_requests`, `*_rl_input`, `*_rl_output`: per-model rate-limiter controls.
- `litellm_global_kwargs`: default timeout/retry-level behavior for all model calls.

## 3) Recommended operating profile (balanced)

Use this first, then tune from metrics.

### Model role strategy

- **Chat model**: high-quality coding model (strong tool use + instruction following).
- **Utility model**: one tier cheaper but still high quality. Do not use tiny local models here.
- **Browser model**: same as chat model or one step down, but keep vision enabled.
- **Embedding model**: start with default CPU embedding for cost efficiency; move to higher-quality remote embeddings if retrieval quality is insufficient.

### Suggested baseline values

- `chat_model_ctx_length`: `128000`
- `chat_model_ctx_history`: `0.55` (keeps more room for active tool results and current objective)
- `memory_recall_enabled`: `true`
- `memory_recall_interval`: `2`
- `memory_recall_history_len`: `12000`
- `memory_recall_memories_max_search`: `10`
- `memory_recall_solutions_max_search`: `8`
- `memory_recall_memories_max_result`: `4`
- `memory_recall_solutions_max_result`: `3`
- `memory_recall_similarity_threshold`: `0.74`
- `memory_recall_query_prep`: `true`
- `memory_recall_post_filter`: `true`
- `memory_memorize_enabled`: `true`
- `memory_memorize_consolidation`: `true`
- `memory_memorize_replace_threshold`: `0.92`

### Why this balance works

- Higher threshold + query prep + post-filter generally improves precision and cuts noisy memory injection.
- Slightly lower history fraction than default leaves room for tools/results in long coding tasks.
- Consolidation keeps memory quality from drifting over long autonomous sessions.

## 4) Mode presets by objective

### A) Maximum coding power (cost-tolerant)

- Increase `chat_model_ctx_length` to model max practical window.
- Keep `chat_model_ctx_history` around `0.55-0.65`.
- Use strongest chat + browser models.
- Keep recall filters ON (`query_prep`, `post_filter`) and threshold `0.75-0.82`.

### B) Efficient daily driver

- Keep strong utility model but use cheaper chat/browser model variants.
- Set recall interval to `3`.
- Keep threshold around `0.72-0.78`.
- Trim max search/results slightly.

### C) Research-heavy exploration

- Keep browser model strong + vision enabled.
- Increase `memory_recall_history_len` and search caps.
- Lower threshold slightly (`0.68-0.73`) to surface broader related context.
- Use project-isolated memory per domain/client.

## 5) Knowledge base architecture for your use case

For personal SWE + research, avoid one giant mixed corpus.

- Keep `default` for general evergreen references.
- Create multiple custom knowledge subdirs by domain (e.g., `ml-systems`, `infra`, `product`, `papers`).
- Use project-specific knowledge+memory isolation when work must not bleed across clients/topics.
- Reindex embeddings only when changing embedding provider/model is truly justified (it forces memory reload/reindex behavior).

## 6) Hyperparameter guidance (practical)

Set provider/model kwargs in `*_model_kwargs`:

- **Coding precision tasks**: lower temperature (`0.0-0.3`), conservative sampling.
- **Ideation/research synthesis**: moderate temperature (`0.4-0.7`).
- **If reasoning models are slow/expensive**: disable reasoning/thinking where provider supports it.

For reliability and stability:

- Add global defaults in `litellm_global_kwargs` (timeouts, retry-friendly behavior).
- Use per-model rate limits if you hit provider quotas.

## 7) Observability loop (how to tune intelligently)

Run 20-30 representative tasks and score:

- Correctness of code / research output
- Tool-call quality
- Latency per turn
- Token and cost usage
- Memory precision (irrelevant memories shown?)

Then tune in this order:

1. Utility model quality
2. Memory recall threshold + query prep/post-filter
3. Chat context length/history split
4. Embedding model quality
5. Rate limits/timeouts

## 8) Safe production posture

- Run in isolated environments (Docker/VM) for risky tasks.
- Use project isolation for clients/sensitive domains.
- Keep secrets in built-in secrets management, not plain prompt text.
- Periodically review and prune memory dashboard entries to prevent stale behavior lock-in.

