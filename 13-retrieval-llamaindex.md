# Chapter 13 — LlamaIndex: Retrieval Infrastructure

Chapter 7 established the negative result: coding agents don't embed the repo — they grep it. LlamaIndex is what you reach for in the *other* regime: corpora that are large, mostly static, and not symbol-addressable — documentation sets, issue trackers, org-wide code search, conversation memory. This chapter extracts the four pieces of its machinery that matter architecturally, then returns to the boundary question: when does an agent actually need this?

## 13.1 The pipeline in one line

Every LlamaIndex deployment is the same dataflow:

```
documents → node_parser (chunking) → nodes(+metadata) → index (vector/keyword/graph/summary)
          → retriever → [postprocessors: rerank, filter] → response_synthesizer → answer
```

Each stage is a swappable component with a typed interface. The architectural value of studying it is not novelty — it is that each stage names a decision your agent's retrieval path must make *somewhere*, explicitly or by accident.

## 13.2 Chunking code with the parse tree

The chunker is where retrieval quality is won or lost, and `CodeSplitter` is the reference implementation of syntax-aware chunking — `llama_index@67514f6/llama-index-core/llama_index/core/node_parser/text/code.py:19` **[Verified]** (crediting SweepAI's chunking write-up in its docstring):

```python
# code.py:169 — recursive greedy packing over the tree-sitter AST
def _chunk_node(self, node, text_bytes, last_end=0) -> List[str]:
    new_chunks, current_chunk = [], ""
    for child in node.children:
        if child_size > max_size:                 # child too big on its own
            if current_chunk: new_chunks.append(current_chunk)
            new_chunks.extend(self._chunk_node(child, text_bytes, last_end))  # recurse INTO it
        elif fits(current_chunk + child_text):
            current_chunk += ...                  # greedy pack siblings
        else:
            new_chunks.append(current_chunk); current_chunk = child_text
    ...
```

The invariant: **chunk boundaries land on AST-node boundaries** — a function is never split mid-body unless the function alone exceeds the budget, in which case recursion descends to its children (statements, nested defs). Configuration is honest about the two cost models — `count_mode: "char" | "token"` with a real tokenizer for the latter (`code.py:19-58` **[Verified]**). Contrast the naive fixed-window splitter: half a function embedded with half of the next one retrieves as noise. SWE-agent's `filemap` (Ch. 7.2) and this splitter are the same tree, used in opposite directions — elide bodies for *overview*, bound subtrees for *retrieval*.

## 13.3 Routing: retrieval strategy as an LLM decision

Real corpora need multiple indexes (vector for semantics, keyword/BM25 for identifiers, summary index for "what is this repo about"). `RouterQueryEngine` — `llama_index@67514f6/llama-index-core/llama_index/core/query_engine/router_query_engine.py:95,160-164` **[Verified]** — makes index choice itself an LLM call: each sub-engine carries a natural-language description ("useful for finding exact class names…"), a selector prompts the model to pick one or several (`LLMSingleSelector` parses a JSON choice), and multi-selections are summarized together.

The pattern to recognize: this is Chapter 4's tool-dispatch loop in miniature — descriptions as affordances, an LLM as the router, validation on the choice. Coding agents do the same thing with *search tools* (grep vs. glob vs. LSP vs. subagent, Ch. 7.6) — they just let the main model route instead of a dedicated selector. A dedicated router earns its extra call when the sub-engines are expensive or numerous.

## 13.4 Response synthesis: fitting N chunks through one window

The `llama_index@67514f6/llama-index-core/llama_index/core/response_synthesizers/` package (**[Verified structure]**: `refine.py`, `compact_and_refine.py`, `tree_summarize.py`, `accumulate.py`, `simple_summarize.py`) is a catalog of algorithms for the terminal problem — retrieved content exceeds the window:

- **refine** — sequential fold: answer from chunk 1, then "given this answer and chunk 2, refine it," … Order-sensitive, never overflows, N calls.
- **compact** (+refine) — pack as many chunks per call as fit, then refine across packed batches: same guarantees, ~N/k calls. The default for good reason.
- **tree_summarize** — map-reduce: summarize leaves, then summarize the summaries, log-depth tree. Parallelizable, best for holistic "summarize all of X" queries.
- **accumulate** — answer per-chunk, return all answers (no fusion) — for when the caller will post-process.

These four shapes are worth memorizing because they recur *everywhere* context exceeds budget: Chapter 8's compaction is `refine` over conversation history; subagent report-backs (Ch. 9.3) are `tree_summarize` with agents as the map step; STORM's per-section writing (Ch. 14) is `compact` over an information table. LlamaIndex just names them.

## 13.5 When an agent actually needs this

A decision rule consistent with everything in this workspace:

| Corpus property | Right tool |
|---|---|
| Mutable during the session (the repo you're editing) | grep + ranged read + LSP (Ch. 7) — indexes go stale |
| Symbol-addressable (code identifiers, error strings) | grep/BM25 — exact match beats embeddings |
| Large + static + semantic (docs, wikis, papers, tickets) | **this chapter** — embed once, retrieve forever |
| Cross-session agent memory | hybrid: files-as-memory first; vector index when it outgrows grep |

And the integration pattern that preserves agency: expose retrieval to the agent **as a tool** (`query_docs(question) → synthesized answer + citations`) rather than stuffing top-k chunks into the prompt pre-emptively. Pre-emptive RAG spends window on guesses; tool-shaped RAG lets the model decide when external knowledge is needed — the same escalation discipline as Chapter 7's cognition ladder. LlamaIndex's own agent layer (`llama_index@67514f6/llama-index-core/llama_index/core/agent/` — ReAct and function-calling workflow agents **[Verified structure]**) closes the loop by wrapping any query engine as exactly such a tool.
