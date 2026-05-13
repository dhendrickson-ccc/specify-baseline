# Agent Memory Patterns

A reference for designing shared knowledge layers in multi-agent LLM systems. These patterns address a specific class of problem: agents that perform repeated work because prior findings are not preserved or accessible to other agents.

This document is intentionally not a tutorial. It describes architectural patterns, the tradeoffs behind them, and the failure modes that each pattern avoids. Use it to inform design decisions, not as a step-by-step implementation guide.

---

## Core Principle: Memory That Doesn't Cost Tokens

The most common mistake in agent memory design is putting memory in the prompt. Every fact injected into the system message or conversation history consumes context window tokens, competes with the actual task for the model's attention, and scales linearly with the amount of knowledge accumulated.

The patterns in this document are built around a different approach: memory that lives in the tool execution layer. The agent calls a tool. Before the tool executes, application code checks a knowledge store. If there's a cached result, it's returned as a normal tool response. If there's a known-negative result, the tool returns an error with a reason. The model never sees the cache. It sees a tool response, same as always.

This means accumulated knowledge has zero context window cost for exact-match lookups. The model's context budget is fully available for reasoning about the current task.

---

## Pattern 1: Tiered Knowledge Cache

Not all knowledge retrieval is the same operation. Different kinds of agent knowledge have different access patterns, and using one retrieval strategy for all of them is either wasteful or lossy.

### Tier 1 - Result Cache (exact key lookup)

Binary, structured outcomes from previous agent actions. "Did we already try this? What happened?"

- Access pattern: direct key match. The query is the same parameters the agent would pass to the tool.
- Context cost: zero. Handled entirely in the tool wrapper.
- Write trigger: after every tool execution, regardless of success or failure.
- Read trigger: before every tool execution of the same type.

This tier catches the most common waste: agents repeating actions that have already been performed and resolved.

### Tier 2 - Reference Data Cache (exact key lookup, lazily populated)

Stable reference information that agents need to validate their work. API schemas, configuration specifications, type definitions, contract documents.

- Access pattern: direct key match by identifier (API endpoint name, resource type, config key).
- Context cost: zero. The schema data stays in application code. Only validation results or relevant snippets enter the tool response.
- Write trigger: first encounter with a new identifier. Batch-populate related records at the same time (see "Batch Write-Through" below).
- Read trigger: when the agent needs to validate or construct something against a known specification.

This tier prevents agents from working with hallucinated or outdated specifications. If the agent tries to use something that doesn't exist in the reference data, the tool wrapper catches it before any real work happens.

### Tier 3 - Institutional Knowledge (semantic search)

Unstructured, experiential knowledge accumulated over time. Past approaches, architectural decisions, lessons learned, contextual recommendations.

- Access pattern: semantic similarity search. The query is a natural language description of the current problem.
- Context cost: small, bounded. Top N results injected into the prompt, where N is a fixed limit.
- Write trigger: after significant agent decisions or task completions, with a summarization step.
- Read trigger: at task start, or when the agent encounters a decision point where prior experience would help.

This tier is the only one that uses embeddings and vector search. It's also the only one that costs context window tokens. Keep N small (2-5 results) and cap total token injection.

### Why Tiers Matter

A common impulse is to put everything in a vector store and do semantic search for all lookups. This is wrong for Tiers 1 and 2.

Semantic search over structured data introduces false positives. When an agent needs to know "does X exist," it needs a binary answer, not a ranked list of things similar to X. Similarity-based retrieval for exact lookups is like using a search engine to check whether a file exists on your local disk - it might work, but `ls` is the right tool.

Separate the tiers. Use exact lookup where the query is structured. Use semantic search where the query is genuinely fuzzy.

---

## Pattern 2: Tool-Layer Interception

The integration point for agent memory is the tool execution layer, not the prompt and not the orchestration layer.

### Why not the prompt?

Injecting memory into the prompt asks the model to remember and apply it. This has three problems:
1. Token cost scales with knowledge volume.
2. The model may ignore injected context, especially under long prompts.
3. You're trusting the model to apply cached knowledge correctly rather than enforcing it deterministically.

### Why not the orchestration layer?

Putting memory logic in the orchestrator (routing decisions, conditional edges) couples your workflow topology to your cache's data shape. If the cache schema changes, the orchestrator breaks. Memory should be an implementation detail of tools, not a control flow concern.

### The wrapper pattern

Every tool that performs an action worth caching gets a wrapper function. The wrapper:

1. Extracts the lookup key from the tool's input parameters.
2. Checks the relevant cache tier.
3. On cache hit: returns the cached result as a normal tool response. The agent cannot distinguish this from a live result.
4. On cache miss: executes the real tool, writes the result to the cache, returns the result.

```
function wrapped_tool(params):
    cache_key = derive_key(params)
    cached = store.get(namespace, cache_key)

    if cached and is_still_valid(cached):
        return format_as_tool_response(cached)

    result = execute_real_tool(params)
    store.put(namespace, cache_key, result_with_metadata(result))
    return format_as_tool_response(result)
```

The model calls a tool. It gets a response. Whether that response was cached or live is invisible to the model. This is the key architectural property - memory is enforced by the runtime, not suggested to the model.

---

## Pattern 3: Negative Knowledge as First-Class Records

Most caching systems only store positive results. Agent memory systems must also store negative results - things that were tried and failed, resources that don't exist, approaches that were ruled out.

Negative knowledge is the most expensive thing for agents to re-derive. A positive result can sometimes be found quickly. A negative result requires exhaustive work before the agent concludes "this doesn't exist" or "this doesn't work." That exhaustive work should happen exactly once.

### Store negatives with context

A bare negative record ("X is invalid") is dangerous without context about why and when. Store:

- **What was tried.** The exact parameters, not a summary.
- **Why it failed.** Error message, reason, evidence.
- **Scope of validity.** Under what conditions is this negative still true? Version numbers, commit identifiers, timestamps - whatever determines when this finding might become stale.
- **Confidence.** Not all negatives are equal. "This resource type doesn't exist in the provider schema" is high confidence. "I searched this directory and didn't find it" is lower - the file might exist elsewhere.

### Expiry conditions on negatives

Negative knowledge becomes harmful when it outlives its validity. An agent that "knows" something doesn't exist will never discover that it now exists. Every negative record needs an expiry condition:

- Version change: negative was for version N, we're now on version N+1. Discard.
- Content change: negative was for commit SHA X, the repo is now at SHA Y. Re-validate.
- Time-based: negative is older than a threshold. Re-check on next access.

The tool wrapper checks expiry conditions on read. Stale negatives are treated as cache misses - the wrapper proceeds with a real lookup and overwrites the record with fresh findings.

---

## Pattern 4: Lazy Population with Batch Write-Through

Reference data (Tier 2) should not be preloaded at pipeline start. Preloading requires knowing in advance what data the agents will need, which defeats the purpose of autonomous agents that explore and discover.

Instead, use lazy loading: the first agent to need a piece of reference data fetches it and writes it to the cache. Every subsequent agent reads from the cache.

### Batch the writes

When one lookup triggers a fetch of reference data, fetch and cache related data at the same time. If an agent needs the schema for one API endpoint, fetch schemas for all endpoints in that API and cache them in one pass.

The reasoning: the fetch is the expensive part (network call, subprocess, parsing). Writing 100 records to the Store is cheap compared to one fetch. And if the agent needs one endpoint from an API, it's likely to need others from the same API soon.

```
function get_or_fetch_reference(identifier, category):
    cached = store.get((category, version), identifier)
    if cached:
        return cached

    # Fetch the entire category, not just the one identifier
    all_data = fetch_full_reference(category, version)

    for item_id, item_data in all_data:
        store.put((category, version), item_id, item_data)

    return store.get((category, version), identifier)
```

### Graceful degradation

If the reference data fetch fails (network error, command not available, permission denied), the tool should proceed without the cache rather than blocking. Log the failure for debugging, but let the agent do its work. The cache is an optimization, not a hard dependency. The result cache (Tier 1) still captures the outcome of whatever the agent does, so future agents still benefit even if the schema cache never populated.

---

## Pattern 5: Namespace-Based Natural Expiry

Instead of implementing TTL logic or expiry sweeps, encode the validity scope into the cache's namespace hierarchy.

```
(cache_tier, domain, version)
```

When the version changes, agents start reading from a new namespace. The old namespace's data is still there but is never consulted for the new version. This is implicit expiry through key design, not explicit expiry through timestamp checks.

### Benefits

- No background cleanup jobs required for correctness (only for storage reclamation).
- No race conditions between expiry checks and concurrent reads.
- Version rollbacks naturally fall back to old cached data if it still exists.
- Easy to reason about: "everything in this namespace was true for version X."

### Choosing namespace dimensions

The namespace should encode every dimension that, when changed, invalidates the cached data. Common dimensions:

- **Version identifier.** API version, dependency version, schema version.
- **Scope identifier.** Repository, project, tenant, environment.
- **Content identifier.** Commit SHA, document revision, last-modified timestamp.

If a dimension changes frequently (every commit), putting it in the namespace creates many short-lived namespaces. For high-churn dimensions, use a field inside the record and check it on read instead. Reserve namespace dimensions for things that change infrequently (version upgrades, environment changes) and have broad impact (invalidate many records at once).

### String formatting constraints

Many key-value store implementations restrict namespace characters. Common restrictions: no dots, no whitespace, no path separators, no quotes. Normalize identifiers before using them as namespace components. Establish a convention early (underscores for dots, hyphens for slashes) and apply it consistently.

---

## Pattern 6: Experiential Corrections

Raw reference data (schemas, specs, contracts) is necessary but not sufficient. Agents discover through experience that the reference data is misleading, incomplete, or wrong in specific contexts. These corrections are a distinct layer on top of reference data.

### What qualifies as a correction

- An argument combination that the schema allows but produces a runtime error.
- A default value that the documentation claims but that doesn't match observed behavior.
- A deprecated pattern that still appears in the reference data.
- A provider-specific or environment-specific constraint not captured in the general schema.

### How to store corrections

Store corrections as an append-only list keyed by the same identifier as the reference data, in a parallel namespace. Don't modify the original reference record - keep raw reference data and experiential corrections separate.

Each correction includes: what field or aspect is affected, what the issue is, what evidence triggered the discovery (error message, unexpected output), and which agent discovered it.

### How to surface corrections

When the tool wrapper returns reference data to the agent, check for corrections and append them to the tool response. Limit to the most recent N corrections (3-5) to bound token cost. Older corrections are retained for audit but not actively surfaced.

Corrections are not authoritative in the way reference data is. They represent "one agent encountered this issue once." High-frequency corrections (the same issue discovered by multiple agents independently) should be promoted to higher confidence, but even a single correction is worth surfacing as a warning.

---

## Pattern 7: Bounded Prompt Briefs

For cases where it's valuable for the model to know about cached knowledge before formulating its first action - not just at tool call time - inject a small, pre-computed summary into the system message.

### When this is useful

The tool wrapper catches bad actions at execution time. The prompt brief prevents the model from even formulating bad actions. If agents are burning tool calls on searches that the wrapper then short-circuits, a prompt brief reduces those wasted round trips.

### Constraints

- Generate once at task start. Do not update mid-run.
- Fixed maximum entry count (e.g., 10 items).
- Fixed maximum token budget (e.g., 200-400 tokens).
- Include only high-confidence records relevant to the current task scope.
- This is supplementary. The tool wrapper is the enforcer. The brief is the hint.

### What to include

Negative knowledge is the highest-value content for a prompt brief. "Don't search for X, it doesn't exist" saves a tool call. "Y is at location Z" is less valuable in the brief because the tool wrapper returns this on cache hit anyway - the model would have found it on its first tool call.

Prioritize: known-invalid items that agents commonly attempt, followed by known gotchas from the corrections layer.

---

## Pattern 8: Self-Healing Cache Design

The cache should be disposable. If it's lost, corrupted, or intentionally cleared, agents repopulate it through normal operation. No manual intervention, no import scripts, no data migration.

This property comes naturally from the lazy population pattern. Agents fetch and cache on miss. Delete the cache and every lookup becomes a miss, triggering a fresh fetch. The first few runs after a cache loss are slower. After that, the system returns to cached performance.

### Design implications

- Never store data in the cache that can't be re-derived from primary sources.
- The cache is a materialized optimization, not a source of truth.
- Backup is nice-to-have, not critical. The cost of cache loss is temporary performance degradation, not data loss.
- Corruption recovery is "delete and restart." No repair tools needed.

---

## Anti-Patterns to Avoid

### Stuffing full history into context

Every record from every prior run injected into the system message. Context window fills up. Model performance degrades. Token costs explode. The tool-layer interception pattern exists specifically to avoid this.

### Using semantic search for structured lookups

"Does endpoint X exist?" should be a key lookup, not a similarity query. Semantic search returns ranked results, which means the answer to a binary question becomes probabilistic. Use exact match for structured data, semantic search for unstructured data.

### Single-tier memory

One big store with one retrieval strategy for all knowledge types. This forces a compromise: either exact lookups are slow (because they go through an embedding pipeline) or fuzzy queries are poor (because the store only supports key match). Separate tiers with separate retrieval strategies.

### Preloading everything

Fetching all possible reference data at startup. This assumes you know what agents will need, adds startup latency, and caches data that may never be used. Lazy population is almost always the right default.

### Mutable reference data

Editing raw reference records when corrections are discovered. This mixes authoritative data with experiential findings and makes it impossible to distinguish "the spec says X" from "an agent once found that X doesn't work." Keep them in separate namespaces.

### Memory without expiry conditions

Cached knowledge without any mechanism for invalidation. This is worse than no cache at all - agents make confident decisions based on stale data and never discover the staleness. Every record needs a scope of validity, checked on read.

### Over-relying on the model to use memory

Injecting memory into the prompt and hoping the model applies it correctly. Models ignore injected context, especially long context, especially when it contradicts their parametric knowledge. If a memory-based decision is important enough to enforce, enforce it in the tool layer where the model can't override it.

---

## Choosing a Storage Backend

The patterns in this document are storage-agnostic. The requirements are:

- Key-value storage with hierarchical namespaces (or equivalent key prefixing).
- Persistent across process restarts.
- Read-after-write consistency (an agent that writes a record can immediately read it back).
- Metadata filtering for Tier 3 if semantic search is needed.

Suitable backends include: any embedded database (SQLite, DuckDB), any relational database (Postgres, MySQL) with a key-value access pattern, framework-integrated stores (LangGraph's BaseStore), or Redis for high-throughput scenarios.

The choice depends on your deployment model:

- Single process, async tasks: embedded database. Simplest option, no external dependencies.
- Multiple processes, same machine: embedded database with WAL mode, or a local database server.
- Multiple containers/machines: a shared database server (Postgres, Redis). Embedded databases don't support concurrent access from separate processes.

If your framework provides a built-in Store abstraction, prefer it over a standalone database. Integration is already done, and migration to a different backend is usually a one-line change.

---

## Evaluating Whether You Need This

Not every agent system needs a shared knowledge cache. The patterns here solve a specific problem: redundant work across agents or across runs, where the cost of re-derivation is significant.

Signs you need it:

- Agents repeatedly fail on the same invalid inputs across runs.
- Observability traces show duplicate tool calls for the same parameters.
- Agent tasks include expensive validation steps (API calls, subprocess execution) that produce deterministic results for the same inputs.
- Multiple agents work in the same domain and would benefit from each other's findings.

Signs you don't:

- Agents run independently on unrelated tasks with no overlapping knowledge.
- Tool calls are cheap and fast enough that caching overhead isn't worth it.
- Your knowledge domain changes so rapidly that cached results are stale before the next agent runs.
- You have a single agent with a single run pattern and no cross-run persistence needs.

Start with Tier 1 (result caching in the tool wrapper). Measure cache hit rate. If it's meaningful, add Tier 2 (reference data). Only add Tier 3 (semantic search) when you have a genuine fuzzy retrieval need that Tiers 1 and 2 don't cover.
