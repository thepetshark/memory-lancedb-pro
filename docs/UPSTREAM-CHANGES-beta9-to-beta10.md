# memory-lancedb-pro: Upstream Changes beta.9 → beta.10

> **Prepared:** 2026-03-26  
> **Scope:** All upstream commits from our fork base (`aa0ec8d`, beta.5 era) through `upstream/master` (`5737614`, beta.10)  
> **Purpose:** Release notes for HomeAgent deployment — what changed, what matters to us, what to watch for  

---

## 🔴 Breaking / High Priority

### OpenClaw 2026.3+ Hook Migration (Critical for us)
- **All lifecycle hooks migrated** from deprecated `before_agent_start` to `before_prompt_build` with explicit priority ordering:
  - auto-recall = priority 10
  - invariants = priority 12  
  - derived focus = priority 15
- **Why it matters:** This is the fix for our 35-second session startup delays. The old `before_agent_start` hook was timing out on every new session. After this update + OpenClaw 2026.3.22+, session starts should be near-instant.
- **Requires:** OpenClaw ≥ 2026.3.22 (we're on 2026.3.24 ✅)
- **Action:** Run `openclaw doctor --fix` after deploying (done ✅)

---

## 🟠 New Features

### Recall Mode — Adaptive Intent Routing
- New `recallMode` config option with adaptive intent routing
- Analyzes query intent before retrieval — routes simple queries to fast BM25, complex semantic queries to full hybrid
- Reduces unnecessary embedding API calls for short/simple messages
- Config: `retrieval.recallMode: "adaptive"` (default remains `"hybrid"`)

### Memory Compaction — Progressive Summarization
- New `memory_compact` tool: progressively summarizes older stored memories to reduce LanceDB footprint
- Groups related memories by category/scope, condenses with LLM, stores summary as a new memory
- Useful when memory store grows large (>500 entries) and retrieval quality degrades
- Config: `compaction.enabled`, `compaction.maxAgedays`, `compaction.targetCategories`

### Session Compression — Adaptive Extraction Throttling
- New session-level compression: when a session produces too many auto-capture candidates, throttles extraction rate
- Prevents memory bloat from long active sessions generating hundreds of low-quality entries
- Auto-enabled; configurable via `autoCapture.throttle.*`

### Observable Retrieval Traces
- New `retrieval.trace: true` config option — logs full retrieval pipeline per query (candidates, scores, rerank results)
- Useful for debugging why certain memories aren't being recalled
- Output goes to stderr / OpenClaw plugin logs

### Batch Dedup
- Deduplication now runs in batch mode across the full candidate set before storing
- Previously checked one-by-one; batch mode catches more semantic duplicates with fewer API calls
- Similarity threshold lowered from 0.95 → 0.90 (catches more near-duplicates)

### Temporal Supersede Semantics
- `memory_update` on temporal categories (preferences, facts) now creates a new version instead of overwriting
- Old version is preserved but marked inactive (`excludeInactive: true` hides them from retrieval)
- `memory_store` with matching category auto-supersedes older entries
- New `memory_categories.ts` defines which categories are version-tracked

### Cross-Process File Lock + ClawTeam Scope
- File-level lock prevents concurrent writes when multiple agents share a LanceDB instance
- New `CLAWTEAM_MEMORY_SCOPE` env var — designates a shared team scope accessible to all agents
- `clawteam` scope propagates through all retrieval paths

### A-MAC Admission Control
- New admission control system: filters low-quality memories before storage
- Evaluates candidates against quality criteria (length, specificity, noise score)
- Stats tracked in `src/admission-stats.ts` — queryable via `memory_stats`
- Config: `admissionControl.enabled` (default: true)

### Azure OpenAI Embedding Provider
- New `azure-openai` embedding provider option
- Config: `embedding.provider: "azure-openai"` with `apiVersion`, `deploymentId`, `endpoint`

### DashScope (Alibaba Cloud) Rerank Provider
- New `dashscope` rerank provider for Alibaba Cloud users
- Config: `retrieval.rerank.provider: "dashscope"`

### TEI (Text Embeddings Inference) Rerank Provider  
- Self-hosted rerank via HuggingFace TEI server
- Config: `retrieval.rerank.provider: "tei"` with `endpoint`

### Auto-Recall Timeout Configurable
- New `autoRecallTimeoutMs` config option (previously hardcoded at 5000ms)
- Allows tuning recall timeout per deployment — useful on slower hardware
- Default: 5000ms

### Memory Governance Hardening
- New governance defaults: stricter category validation, mandatory scope on store
- `self_improvement_log` tool for structured error/learning/feature-request entries
- Governance maintenance scripts in `scripts/`

---

## 🟡 Fixes

### Scope Resolution (resolveToolContext)
- `memory_forget`, `memory_update`, `memory_stats`, `memory_list` were using incorrect agentId resolution
- Now use `resolveToolContext()` consistently — fixes "outside accessible scopes" errors
- **Our patch (`176c87f`) is now redundant** — upstream covers this fully

### Auto-Capture: Strip OpenClaw Envelope Metadata
- Auto-capture now strips `Conversation info (untrusted metadata):` and `Sender (untrusted metadata):` blocks before extraction
- Prevents noisy metadata from being captured as memories
- **Our partial patch covered this** — upstream implementation is more comprehensive

### Retrieval Correctness
- Fixed `vectorWeight`/`bm25Weight` not actually affecting fusion scoring (was always 50/50 regardless of config)
- Fixed cosine distance computation in vector search
- Fixed BM25-only post-processing pipeline to prevent stale tag matches
- Fixed `excludeInactive` not propagating to all retrieval search paths

### Reflection Injection Safety
- Reflection slices sanitized before prompt injection — prevents injecting corrupted/partial reflections
- Filter for false-positive reflection recall entries (stops reflection memories polluting regular recall)

### CJK Infinite Recursion Fix
- Fixed infinite recursion in `embedSingle()` for CJK text hitting embedding context limits
- Chunker now converges on CJK content correctly

### Preference Slot Dedup Fix
- Same-brand different-item preferences no longer deduped (e.g. "I prefer X over Y" and "I prefer X over Z" kept separately)
- Category string normalization fixed in preference-slot guard

### LanceDB Windows ESM Fix
- Uses `require()` for LanceDB to fix Windows ESM URL scheme errors
- No impact on our Linux deployment

### Non-Blocking Auto-Capture Hook
- `agent_end` auto-capture is now fire-and-forget — no longer blocks session teardown

### Bounded Before-Agent-Start Timeout
- Added timeout to `before_agent_start` auto-recall hook (previously unbounded)
- Moot after hook migration to `before_prompt_build`, but was important for beta.9 stability

### Retrieval Score Display
- Removed score/source suffixes from visible recall text injected into prompts
- Cleaner memory injection with no debug metadata leaking into context

### json5 Dependency
- `json5` now declared as a proper direct dependency in `package.json`
- **Our patch (`19dd7f1`) is now redundant**

### USER.md Boundary Handling
- Skip storing facts extracted exclusively from USER.md content
- Prevents USER.md (system-managed) from polluting auto-captured memories

---

## 🟢 Our Custom Patches — Status After This Update

| Patch | Status | Action |
|-------|--------|--------|
| Heartbeat/NO_REPLY skip | **Partial** — upstream covers `/HEARTBEAT/i` but not `NO_REPLY`, `health_check`, `system_check` | Keep `NO_REPLY` + health/system patterns only |
| Structural noise filter | **Not in upstream** | Keep as-is |
| `includeVectors` for backups | **Not in upstream** | Keep as-is |
| `resolveToolContext` scope fix | **Fully covered upstream** | Remove |
| json5 dependency | **Fully covered upstream** | Remove |
| Envelope metadata strip | **Covered upstream** (more comprehensive) | Remove |

---

## Requirements

- **Minimum:** OpenClaw 2026.3.22
- **Recommended:** OpenClaw 2026.3.24+ (we're on this ✅)
- Run `openclaw doctor --fix` after deploying
