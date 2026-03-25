# Agent Zero Optimization Playbook (Personal SWE + Research Scientist)

This playbook is a practical configuration strategy for users who want Agent Zero to act as a high-quality **software engineer + research scientist** while balancing:

- **Performance** (quality of reasoning/execution)
- **Efficiency** (latency/cost)
- **Precision** (relevance/accuracy)
- **Power** (ability to solve hard, multi-step tasks)

---

## 1) First principles: split models by job

Agent Zero already separates model roles in settings:

- **Chat model** = primary reasoning and execution planner
- **Utility model** = summarization/memory extraction/query prep
- **Embedding model** = memory/knowledge retrieval quality
- **Browser model** = web/browser operations (when used)

This is the most important lever for balancing cost and quality.

### Recommended role allocation

- Use your **best coding/reasoning model** as chat model.
- Use a **fast, cheap but still reliable model** as utility model.
- Use a **stable embedding model** and avoid switching it frequently.
- Keep browser model aligned with chat quality if browser tasks are frequent.

---

## 2) Baseline profile for “high capability, controlled cost”

Start with this profile and then tune by workload:

- Chat model context length: **100k–200k**
- Context window share for history (`chat_model_ctx_history`): **0.55–0.70**
- Utility model: small/fast model with good instruction following
- Memory recall:
  - enabled = `true`
  - delayed = `false` (quality-first) or `true` (latency-first)
  - interval = `2` or `3`
  - similarity threshold = **0.72–0.82**
- Auto-memorize:
  - enabled = `true`
  - AI consolidation = `true` for long-lived usage
- Projects: **always use project isolation** for clients/domains

---

## 3) Memory and knowledge tuning (most impact after model choice)

### Memory recall (search-time quality/cost)

If you want high precision with minimal noise:

- Increase threshold toward `0.8`
- Keep max injected memories low (`3–6` memories, `2–4` solutions)
- Enable post-filter only when quality is insufficient (adds extra utility calls)

If you want broader recall:

- Lower threshold toward `0.65–0.72`
- Increase search pool (not necessarily injected count)

### Memory memorization (write-time quality)

- Keep `memory_memorize_enabled=true`.
- Keep `memory_memorize_consolidation=true` for serious long-term use.
- If consolidation is disabled, use `memory_memorize_replace_threshold` around `0.88–0.95` to prevent duplicate clutter.

### Knowledge base layout

Use separate knowledge subdirectories by domain:

- `knowledge/custom/main` → stable long-term references
- `knowledge/custom/solutions` → reusable playbooks/runbooks
- per-project knowledge for client-specific corpora

Prefer high-signal docs (architecture decisions, coding standards, API contracts, experiment logs) over raw dumps.

---

## 4) Context budget and conversation compression

Agent Zero compresses/summarizes history automatically. To keep coding quality high:

- Don’t push chat history ratio to 1.0; reserve room for tools/RAG/output.
- For deep engineering sessions, set history ratio near `0.6–0.75`.
- For repetitive execution loops, lower to `0.45–0.6` and rely more on memory + project files.

Rule of thumb:

- If the agent “forgets” constraints, increase `chat_model_ctx_history`.
- If tool outputs get truncated or responses degrade, reduce it and/or increase model context length.

---

## 5) Project isolation strategy (critical for consultants/freelancers)

For each client/research stream, create a dedicated Project with:

- Own memory (recommended)
- Own instructions (role, coding standards, paths, acceptance criteria)
- Own secrets/variables
- Optional git repo clone in project workspace

This avoids context bleed and significantly improves reliability for multi-client work.

---

## 6) MCP and tools strategy

For software engineering + research, prioritize MCP integrations for high-leverage operations:

- Browser automation (Playwright/Chrome DevTools MCP)
- Databases (SQLite/Postgres MCP)
- Source-control/dev tooling MCPs
- Workflow automation MCPs (n8n, etc.)

Start with 2–4 reliable MCP servers, then expand. Too many tools increase prompt/tool-selection complexity.

---

## 7) Hyperparameter patterns (LiteLLM kwargs)

Use model kwargs conservatively. Good defaults:

- Temperature: `0.1–0.3` for coding/analysis precision
- Top-p: leave provider default unless you have a reason
- Stream timeout/global timeout: set explicitly for reliability (e.g. `timeout=60`, `stream_timeout=60`)

Use **global** LiteLLM kwargs only for cross-cutting concerns (timeouts/retries), and per-model kwargs for model-specific behavior.

---

## 8) Performance/cost operating modes

### Mode A: Research-heavy (quality-first)
- Strong chat model
- Utility model medium quality
- Recall delayed = false
- Query prep = true, post-filter = true
- Higher history ratio (0.65–0.75)

### Mode B: Coding throughput (speed-first)
- Strong chat model
- Fast utility model
- Recall delayed = true
- Query prep = false initially
- Moderate history ratio (0.5–0.65)

### Mode C: Budget mode
- Mid-tier chat model
- Cheap utility model
- Fewer memory recall events (interval 4–6)
- Lower search/result counts
- Consolidation on, but monitor utility call volume

---

## 9) Suggested initial values (copy/paste checklist)

- `chat_model_ctx_history=0.65`
- `memory_recall_enabled=true`
- `memory_recall_delayed=false`
- `memory_recall_interval=3`
- `memory_recall_similarity_threshold=0.78`
- `memory_recall_memories_max_search=10`
- `memory_recall_memories_max_result=4`
- `memory_recall_solutions_max_search=8`
- `memory_recall_solutions_max_result=3`
- `memory_recall_query_prep=false` (turn on if recall quality is weak)
- `memory_recall_post_filter=false` (turn on only when needed)
- `memory_memorize_enabled=true`
- `memory_memorize_consolidation=true`
- `memory_memorize_replace_threshold=0.9`

---

## 10) 2-week calibration loop

1. Run real workloads for 3–5 days.
2. Track:
   - average response latency
   - recall relevance (subjective score)
   - tool-call success rate
   - token/cost trends
3. Tune only one dimension at a time:
   - recall threshold
   - interval
   - history ratio
   - utility model quality
4. Keep a changelog in project notes.
5. Revisit monthly as models/providers evolve.

---

## 11) Security and reliability guardrails

- Use Docker isolation and project-scoped secrets.
- Avoid exposing broad credentials globally.
- Enable authentication before remote/tunnel usage.
- Keep MCP servers minimal and trusted.

---

## 12) Example environment bootstrap (`A0_SET_`)

```env
# Chat model
A0_SET_chat_model_provider=openrouter
A0_SET_chat_model_name=anthropic/claude-sonnet-4.6
A0_SET_chat_model_ctx_length=120000
A0_SET_chat_model_ctx_history=0.65

# Utility model
A0_SET_util_model_provider=openrouter
A0_SET_util_model_name=google/gemini-3-flash-preview

# Memory
A0_SET_memory_recall_enabled=true
A0_SET_memory_recall_interval=3
A0_SET_memory_recall_similarity_threshold=0.78
A0_SET_memory_memorize_enabled=true
A0_SET_memory_memorize_consolidation=true

# LiteLLM reliability
A0_SET_litellm_global_kwargs={"timeout":60,"stream_timeout":60}
```

---

## 13) Final recommendation

If your goal is a durable “personal SWE + research scientist”, optimize in this order:

1. **Project instructions quality**
2. **Chat model quality**
3. **Memory precision tuning**
4. **MCP tool quality**
5. **Cost optimization**

This ordering tends to deliver the best long-term outcome: reliable, high-agency execution with controllable cost.
