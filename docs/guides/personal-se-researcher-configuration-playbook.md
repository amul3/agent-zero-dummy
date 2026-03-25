# Personal Software Engineer + Research Scientist Configuration Playbook

This playbook provides a practical configuration profile for Agent Zero when used as a high-capability personal **software engineer** and **research scientist**, with a focus on balancing:

- Performance (quality of outputs)
- Efficiency (latency + cost)
- Precision (factual and implementation accuracy)
- Power (ability to execute complex, long tasks)

## 1) Operating Model: Split Responsibilities Across Models

Agent Zero already separates concerns across model roles:

- **Chat model**: main reasoning + tool orchestration
- **Utility model**: summarization, memory extraction, query prep/filtering
- **Embedding model**: retrieval quality for memory/knowledge
- **Browser model**: web browsing automation

Use this architecture intentionally: keep the main model strong, utility fast/cheap, embeddings stable.

## 2) Recommended Baseline (Balanced Profile)

Start with these principles, then tune:

- **Chat model**: strong coding/reasoning model with large context (100k+ preferred)
- **Utility model**: lower-cost fast model that still follows instructions reliably
- **Embedding model**: keep stable over time to avoid expensive re-index churn
- **Memory**: enabled with moderate recall limits and conservative thresholds
- **Projects**: use project isolation for each client/research domain

### Suggested first-pass settings

```env
# Main model
A0_SET_chat_model_provider=openrouter
A0_SET_chat_model_name=anthropic/claude-sonnet-4.6
A0_SET_chat_model_ctx_length=120000
A0_SET_chat_model_ctx_history=0.60

# Utility model
A0_SET_util_model_provider=openrouter
A0_SET_util_model_name=google/gemini-3-flash-preview
A0_SET_util_model_ctx_length=64000
A0_SET_util_model_ctx_input=0.60

# Embeddings (keep stable)
A0_SET_embed_model_provider=huggingface
A0_SET_embed_model_name=sentence-transformers/all-MiniLM-L6-v2

# Memory recall
A0_SET_memory_recall_enabled=true
A0_SET_memory_recall_interval=3
A0_SET_memory_recall_similarity_threshold=0.74
A0_SET_memory_recall_memories_max_search=12
A0_SET_memory_recall_solutions_max_search=8
A0_SET_memory_recall_memories_max_result=4
A0_SET_memory_recall_solutions_max_result=3
A0_SET_memory_recall_query_prep=true
A0_SET_memory_recall_post_filter=false

# Memorization
A0_SET_memory_memorize_enabled=true
A0_SET_memory_memorize_consolidation=true

# Optional global guardrails
A0_SET_litellm_global_kwargs={"stream_timeout":60,"timeout":120,"a0_retry_attempts":2,"a0_retry_delay_seconds":1.5}
```

## 3) Why these values work

### 3.1 Context allocation (`chat_model_ctx_history`)

Agent Zero dedicates a configurable ratio of chat context to history. Higher values improve continuity but can starve space for system prompt, RAG inserts, and long outputs. For engineering/research, **0.55–0.70** is a practical range.

- Use **~0.60** as baseline
- Raise toward **0.70** for long architectural discussions
- Lower toward **0.50** for heavy RAG/tool workflows

### 3.2 Memory recall threshold

`memory_recall_similarity_threshold` controls precision/recall tradeoff:

- Too low (<0.65): more noise
- Too high (>0.82): misses useful prior work

For mixed software + research tasks, start at **0.72–0.78** (suggest **0.74**).

### 3.3 Recall cadence (`memory_recall_interval`)

Recall runs every N monologue turns.

- **1–2**: best precision, highest utility-model/token cost
- **3**: strong balanced default
- **4–6**: cheaper but can delay relevant recalls

### 3.4 Query prep and post-filter toggles

These each add utility model calls to improve relevance.

- Enable **query prep** when tasks are ambiguous or long-horizon.
- Keep **post-filter** off by default; turn on when memory noise is noticeably high.

### 3.5 Consolidation

Enable `memory_memorize_consolidation=true` for durable memory quality. It increases utility-model calls but prevents memory bloat and duplicate fragments over time.

## 4) Configuration by workload mode

### Mode A — Deep coding / repo refactor

- Keep chat model premium
- `chat_model_ctx_history=0.65`
- `memory_recall_interval=2`
- `memory_recall_query_prep=true`
- `memory_recall_post_filter=true` only if noisy memory

### Mode B — High-volume research sweeps

- Utility model should be very fast
- `chat_model_ctx_history=0.50–0.60`
- `memory_recall_interval=3–4`
- Raise `memory_recall_solutions_max_result` only if reusable workflows matter

### Mode C — Cost-constrained daily operations

- Use a cheaper chat model tier
- `memory_recall_interval=4`
- `memory_recall_query_prep=false`
- keep consolidation enabled but monitor latency

## 5) Projects, memory, and knowledge isolation strategy

For your personal SE + research setup:

- Create one project per domain/client/research stream
- Use **own memory** for sensitive or domain-specific work
- Use global memory only for shared foundational patterns
- Keep project-specific documents in each project knowledge folder

This prevents cross-domain contamination while preserving high reuse inside each domain.

## 6) Knowledge base curation workflow

Treat knowledge ingestion as an editorial process:

1. Import only high-signal documents (design docs, runbooks, APIs, papers)
2. Remove stale docs aggressively
3. Separate:
   - evergreen knowledge (global)
   - project-specific corpora (project scope)
4. Re-index after major corpus changes

Recommended chunk hygiene:

- prefer markdown/text extracts for cleaner embeddings
- break giant PDFs into structured notes when possible
- include dates/version labels in filenames

## 7) Hyperparameters and extra kwargs that matter most

Use provider/model kwargs sparingly; over-tuning hurts reliability.

Highest-impact knobs:

- `temperature`: lower for coding precision (e.g., 0.1–0.3)
- `max_tokens`: ensure long-form outputs don’t truncate
- `timeout`, `stream_timeout`: improve resilience for long calls
- `a0_retry_attempts`, `a0_retry_delay_seconds`: transient reliability

Example per-chat-model kwargs (UI textarea format):

```txt
temperature=0.2
max_tokens=16000
```

Example global kwargs:

```txt
stream_timeout=60
timeout=120
a0_retry_attempts=2
a0_retry_delay_seconds=1.5
```

## 8) Model portfolio recommendations (decision logic)

When selecting providers/models, use this matrix:

- **Primary chat model**: best coding + long-context instruction fidelity you can afford
- **Utility model**: prioritize speed + low cost + structured output reliability
- **Embedding model**: prioritize stability and semantic retrieval quality over benchmark hype
- **Browser model**: same as chat unless web automation needs different strengths

If budget allows, keep browser and chat aligned to reduce behavioral mismatch.

## 9) Operational safeguards for precision

For research-grade precision:

- instruct the agent to cite sources and uncertainty explicitly
- require a verification pass for critical conclusions
- use project instructions to enforce output templates:
  - assumptions
  - method
  - evidence
  - confidence
  - next tests

For software engineering precision:

- require tests + lint/type checks before final answer
- ask for diff summaries and rollback plans
- enforce “explain tradeoffs, not only decisions” in project instructions

## 10) Practical rollout plan

### Week 1: Baseline

- Apply balanced profile
- Keep logs of latency, cost, and subjective quality

### Week 2: Precision tuning

- tune recall threshold ±0.03
- toggle query prep on/off and compare memory relevance

### Week 3: Efficiency tuning

- adjust recall interval
- tune utility model and global timeouts/retries

### Week 4: Specialization

- split into projects per domain
- add project instructions + curated knowledge packs

## 11) Anti-patterns to avoid

- Changing embedding models frequently (forces re-index/retrieval drift)
- Running maximum recall limits by default (context pollution)
- Using one shared memory for unrelated domains
- Overstuffing model kwargs with provider-specific unsupported params
- Keeping stale knowledge files in active corpora

## 12) “Gold” target profile for your use case

If your goal is "personal staff engineer + research scientist":

- premium main model
- fast reliable utility model
- stable local-ish embeddings
- memory recall on, moderate interval, moderate threshold
- consolidation on
- project isolation on for each major stream
- explicit verification/citation instructions in project prompts

This yields a strong balance of power, precision, and cost control without overcomplicating operations.
