# Chapter 8 — Context Management

The context window is the agent's RAM, and it leaks: every tool result, every file read, every compiler dump stays in history forever unless something evicts it. Context management is the discipline of **detecting pressure, choosing victims, and preserving invariants** — and it is where long-horizon agents are won or lost. The three TypeScript/Rust agents here converge on a two-tier scheme (cheap deterministic pruning + expensive LLM summarization); the details differ instructively.

## 8.1 Detecting pressure

- **opencode** checks after every finished assistant message: `compaction.isOverflow({tokens, model})` consults the model's window, configured output reserve, and usable share (`opencode@34e5809/packages/opencode/src/session/overflow.ts`, called from the loop at `opencode@34e5809/packages/opencode/src/session/prompt.ts:1161-1168` **[Verified]**). Token counts come from the provider's own usage reports where possible; estimation falls back to `Token.estimate(JSON.stringify(modelMessages))` (`opencode@34e5809/packages/opencode/src/session/compaction.ts:180-186` **[Verified]**) — a chars/4-style heuristic. The lesson: **you need a cheap, always-available estimator plus provider truth when it arrives**; both will be a few percent wrong, so all thresholds must carry margin.
- **Codex** computes a full `token_status` after *every sampling request, inside the turn*: active context tokens, an auto-compact scope limit, `tokens_until_compaction` (`open-interpreter@764a96e/codex-rs/core/src/session/turn.rs:306-345`, `open-interpreter@764a96e/codex-rs/core/src/session/context_window.rs`, `open-interpreter@764a96e/codex-rs/core/src/session/token_budget.rs` **[Verified]**). Crossing the limit triggers compaction **mid-turn** (`CompactionPhase::MidTurn`) — necessary because one agentic turn with dozens of tool calls can overflow a window on its own. It also *warns the model* as budget dwindles via reminder items (`open-interpreter@764a96e/codex-rs/core/src/session/rollout_budget.rs`, Ch. 3.5), so the model can wrap up rather than get truncated mid-thought.
- **Cline** triggers on `reserveTokens` (headroom in absolute tokens) or `thresholdRatio` (fraction of max input), whichever is configured — `cline@6309971/sdk/packages/core/src/extensions/context/compaction.ts:193-232` **[Verified]** — with guardrails like `MIN_CONTEXT_DERIVED_INPUT_RATIO = 0.5` (never trust a derived input limit below half the window) and a `budget-projection/` module that *projects* whether the next request will fit before sending it.

## 8.2 Tier 1 — deterministic pruning (cheap, lossy, targeted)

The first response to pressure should not be an LLM call. It should be deleting things whose value has demonstrably expired — and in a coding agent, that means **old tool outputs**: the 400-line file read from 40 turns ago is almost worthless, while the assistant's *decisions* about it remain load-bearing.

**opencode `prune`** — `opencode@34e5809/packages/opencode/src/session/compaction.ts:241-287` **[Verified]**, walking backwards through history:

```typescript
export const PRUNE_MINIMUM = 20_000   // don't bother for < 20k tokens reclaimed
export const PRUNE_PROTECT = 40_000   // most recent 40k tokens of tool output are safe
const PRUNE_PROTECTED_TOOLS = ["skill"]

// walk messages newest→oldest, skipping the 2 most recent user turns entirely;
// stop at any prior summary boundary; accumulate completed tool outputs;
// once the running total exceeds PRUNE_PROTECT, mark older outputs
//   part.state.time.compacted = Date.now()
// …but only commit if the total reclaimable exceeds PRUNE_MINIMUM.
```

The eviction *doesn't rewrite history* — parts are flagged, and the prompt renderer filters flagged content (`MessageV2.filterCompactedEffect`, Ch. 2.3). Three transferable rules: **recency protection is measured in tokens, not turns** (40k of protection ≈ however many turns that spans); **some content is never prunable** (`skill` instructions — deleting the instructions the agent is currently following is self-sabotage); **don't churn for crumbs** (20k minimum reclaim, because every history mutation is also a cache invalidation — §8.5).

**Cline "basic" compaction** — `basic-compaction.ts` **[Verified]** — is the same idea with a stricter invariant made explicit: candidates are **atomic pairs** (`collectAtomicRemovalCandidates`, `:190`) — a tool *call* and its *result* are removed together or not at all, because orphaned halves violate provider API contracts (an Anthropic `tool_use` block without its `tool_result` is a hard 400 error). Truncation of what remains preserves message *structure*, truncating text blocks within a message rather than deleting blocks (`truncateMessageToTokens`, `:90-112`).

## 8.3 Tier 2 — summarization (expensive, lossy differently)

When pruning isn't enough, all three systems compress the *head* of the conversation into a summary and keep a verbatim *tail*. The interesting engineering is the head/tail split and the state representation.

**opencode** — `opencode@34e5809/packages/opencode/src/session/compaction.ts` **[Verified]**:

- Tail budget: `preserve_recent_tokens` config, defaulting to `clamp(25% of usable window, 2_000, 8_000)` (`preserveRecentBudget`, `:80-85`).
- `select()` (`:188-239`) walks the last `tail_turns` (default 2) user-turns newest-first, accepting whole turns while they fit the budget; when a turn doesn't fit, `splitTurn` (`:105-128`) finds the latest *message boundary inside the turn* that fits — turns are the preferred unit, messages the fallback. Head = everything before the kept tail.
- **The summary is itself a message pair in the log**: a user message carrying a `compaction` part, answered by an assistant message flagged `summary: true` (`completedCompactions`, `:62-78`). Rendering for the model simply *filters* to post-compaction messages + summary. History is never destroyed — the full log remains for the UI, for audit, and for re-compaction from scratch. This is the event-sourcing dividend (Ch. 2.3).
- Auto-compaction on overflow additionally *replays* the interrupted user request after the summary (`processCompaction` `:289-319` finds the last real user message and re-enqueues it) so the model resumes the actual task, not just the memory of it.

**Cline "agentic"** — `agentic-compaction.ts:99` **[Verified]** — same head/tail scheme (`preserveRecentTokens`), with two production wrinkles: the summarizer can be a *different, cheaper model* (`resolveSummarizerConfig`; separate provider allowed), and the summary request itself gets an input budget computed against the summarizer's own limits (`buildAgenticSummaryInputBudget`, `:51`) — compressing an oversized history can overflow the *compressor*, so the input to summarization is itself truncated first (`summarizeToolResults` pre-shrinks tool outputs — `compaction.ts:128-158`).

**What goes in the summary prompt** (convergent across opencode's `buildPrompt` import from core and Cline's summarizer): state-of-work (files touched, decisions made, current failures, next steps) — not narrative recap. The summary is a *handoff document to yourself*.

## 8.4 The benchmark-era alternative: observation elision

SWE-agent's `LastNObservations` (`SWE-agent@1132b3e/sweagent/agent/history_processors.py:85-176` **[Verified]**) predates giant windows and is still the right tool for fixed-recipe agents: keep only the last N tool *observations* verbatim, replace older ones with `"Old environment output: (57 lines omitted)"`, never elide the first observation (it's the task statement), honor per-message `keep_output`/`remove_output` tags. Thoughts and actions survive; only environment output is elided — the same "decisions outlive data" insight as §8.2, discovered in 2023.

Its docstring contains the sharpest sentence in this chapter's literature **[Verified]**: *"Note that using this history processor will break prompt caching (as the history of every query will change every time)"* — mitigated by the `polling` parameter (evict every k steps, keeping between `n` and `n+polling` observations) to batch invalidations.

And in miniature, STORM does the same inside a single research interview (`storm@fb951af/knowledge_storm/storm_wiki/modules/knowledge_curation.py:103-113` **[Verified]**): interview turns beyond the last 4 render as *"Expert: Omit the answer here due to space limit"* — questions kept, answers elided, 2,500-word clamp. Context management is fractal: any loop that accumulates text needs an eviction policy, even a sub-loop inside one pipeline stage.

## 8.5 The cache constraint (the rule that shapes everything)

Prompt caching bills only *appended* tokens cheaply; any mutation of earlier history re-bills everything after it. So every mechanism in this chapter is in tension with cache economics, and the mature systems all encode the same response:

1. **Batch evictions** (SWE-agent `polling`; opencode `PRUNE_MINIMUM`) — one big invalidation beats many small ones.
2. **Prefer append-only signaling** — reminders and warnings go at the *tail* (Codex records budget reminders as new items rather than editing old ones).
3. **When you must rewrite, rewrite big** — summarization resets the entire prefix anyway, so it's the natural moment to also prune, reorder, and re-anchor cache breakpoints (SWE-agent's `CacheControlHistoryProcessor` pins `cache_control` marks to the last messages — `history_processors.py:261` **[Verified]**).

Budget rule of thumb visible in these systems: aim compaction to land you well below ~50% of the window (Cline's `LONG_CONVERSATION_TARGET_RATIO = 0.5` **[Verified]**) — compacting to 90% just schedules the next compaction.

## 8.6 Durable memory beyond the window

Above the window sits state that must outlive it: opencode's full message log (SQLite) with flags rather than deletions; Codex's **rollout files** — append-only JSONL per session enabling exact reconstruction (`open-interpreter@764a96e/codex-rs/core/src/session/rollout_reconstruction.rs`, truncation handling in `thread_rollout_truncation.rs` **[Verified structure]**) plus a state DB; Cline delegating persistence to hosts; and cross-session artifacts — plan/todo files (Ch. 9), AGENTS.md-style instruction files, memory directories — which are *the agent writing to its own future context* through the filesystem. The window is a cache over durable state, not the state itself; systems that treat it that way (opencode, Codex) get resumption, audit, and multi-client for free, and their compaction becomes a *view* concern rather than a destructive operation.

## 8.7 Checklist

1. Two tiers: deterministic pruning first (tool outputs, token-measured recency protection, protected classes), LLM summarization second.
2. Remove tool call/result **pairs atomically**; never orphan a provider-side block.
3. Head/tail with a sized tail (≈25% of window, clamped); split oversized turns at message boundaries.
4. Represent compaction as *data in the log*, not destruction of it.
5. Compute the summarizer's own input budget; pre-shrink before compressing.
6. Batch every history mutation; you are spending cache money.
7. Warn the model before the wall (budget reminders); replay the interrupted request after compaction.
8. Compact mid-turn if your turns can outgrow the window on their own.
