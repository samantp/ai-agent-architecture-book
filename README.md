# How Coding Agents Actually Work

## An Architecture Reference, Derived from Source

**Version 2** — reverse-engineered from the nine open-source codebases cloned in this workspace, written for principal engineers and architects. No LLM/RL mathematics; instead: the core algorithms, the real code, and the design decisions that separate a demo from a production agent.

---

## What this is

Tools like Cursor, Claude Code, Cline, and OpenHands look, from the outside, like a chat window bolted onto an LLM. What actually makes them work — turning a fallible text-completion model into something that can read a million-line codebase, edit files without corrupting them, run a shell safely, and recover when it gets confused — is a substantial pile of ordinary software engineering: event logs, state machines, permission systems, approximate string matching, OS sandboxing. That engineering is almost never written down. It lives scattered across the source of a dozen open-source projects, each of which reinvented large parts of it independently.

This booklet is an attempt to write it down once, in one place, grounded in real code rather than in blog-post generalities.

**The method:** nine open-source agent projects are cloned into sibling folders in this workspace (see the table below). Each chapter picks a concern — the agent loop, editing files, sandboxing, context management, multi-agent coordination — and works through how multiple independent implementations solved it, quoting actual source with file-and-line citations. Where the implementations agree, that convergence is treated as evidence of a real, load-bearing pattern. Where they disagree, the trade-off is made explicit rather than papered over.

**The intended reader** is someone who already knows how to build software — a principal engineer, architect, or technical lead — and wants to understand agent internals at the same depth they'd want for, say, a database engine or a compiler: the actual algorithms and data structures, not a product pitch. It deliberately stays out of the underlying LLM/RL theory; the assumption is that "the model" is a capable-but-unreliable black box, and the interesting engineering is everything built *around* it.

**How to use it:** start with the [table of contents](#table-of-contents) below and either read start to end (Parts I–IV build on each other) or jump straight to whatever concern you're currently facing (e.g. "how do agents edit files without corrupting them" → Chapter 5). Chapter 15 is a standalone build guide if you're implementing an agent yourself. Every non-trivial claim carries a `repo@hash/path:line` citation you can open directly — see [Citation convention](#citation-convention-git-pinned) below for how to resolve one.

---

## What changed from v1

*(This section is a changelog against an earlier draft of this booklet and is only relevant if you read that draft. First-time readers can skip to [The reference codebases](#the-reference-codebases).)*

Version 1 described the right topics but grounded too many claims in memory of *older versions* of these projects rather than the code actually sitting in this folder. Version 2 was re-derived from the current clones, which changed several conclusions:

1. **The `open-interpreter/` folder is not the Python Open Interpreter.** It is Open Interpreter 1.0 — a Rust coding agent **forked from OpenAI's Codex CLI** (`open-interpreter@764a96e/codex-rs/*` crates). This is a gift: it means this workspace contains a production-grade Rust agent with the most sophisticated OS-level sandboxing of any project here. Chapter 6 is built on it.
2. **The `OpenHands/` clone is OpenHands V1 (the app shell), not the classic agent.** The agent loop lives in external PyPI packages (`openhands-sdk`, `openhands-agent-server`, `openhands-tools` — pinned at 1.35.0 in `pyproject.toml`). What *is* in this repo — the FastAPI app server and its Docker/process/remote sandbox orchestration — turns out to be the most instructive part anyway: the agent runs *inside* the sandbox and reports back over webhooks (Chapter 6).
3. **Cline has been re-architected** into a monorepo with a standalone agent SDK (`cline@6309971/sdk/packages/agents`, `cline@6309971/sdk/packages/core`). The famous `Task` class with `recursivelyMakeClineRequests` is gone; in its place is `AgentRuntime` — a clean, hook-based agent loop that is *better* teaching material (Chapter 2).
4. Every code snippet in v2 is quoted from a file in this workspace with a `path:line` pointer, or is explicitly labeled as pseudo-code or external knowledge.

## Epistemic conventions

Claims are tagged where it matters:

- **[Verified]** — quoted from or directly traced through source in this workspace; a `repo@hash/path:line` pointer is given.
- **[Inference]** — a conclusion reasoned from the code (e.g., from module structure), but not traced end-to-end.
- **[External]** — from documentation, papers, or general knowledge; not checkable in this workspace (e.g., the OpenHands SDK internals, closed-source agents).

### Citation convention (git-pinned)

Every source reference is pinned to an exact commit and written as:

```
<repo>@<short-hash>/<path>:<line[-range]>
e.g.  opencode@34e5809/packages/opencode/src/tool/edit.ts:682-737
```

- The short hash identifies the revision in the **pinned revisions** table below; references never resolve against a moving HEAD, so line numbers stay valid even after you `git pull` the clones (check out the pinned hash to follow one).
- **Shorthand references** (a bare filename like `turn.rs:227`, or a partial path) inherit the repo and revision of the nearest preceding fully-qualified reference in the same passage.
- To turn any reference into a browser permalink: `https://github.com/<upstream>/blob/<full-hash>/<path>#L<line>`.

### Pinned revisions

| Repo (folder) | Short | Full commit | Committed | Upstream |
|---|---|---|---|---|
| `cline/` | `6309971` | `63099710895e24593554b1e77ec7852f6f16c05c` | 2026-07-11 | `cline/cline` |
| `opencode/` | `34e5809` | `34e58090595d44e3e7cc37498f16753a98627456` | 2026-07-11 | `anomalyco/opencode` |
| `open-interpreter/` | `764a96e` | `764a96ee05853d5494d7e711eefecec57ab712ef` | 2026-07-06 | `OpenInterpreter/open-interpreter` |
| `SWE-agent/` | `1132b3e` | `1132b3e80a45487ce8423f75d0e180874bf84caa` | 2026-07-07 | `princeton-nlp/SWE-agent` |
| `OpenHands/` | `3949e1c` | `3949e1cc17d9443f1f4ef7d34d428baf065cd919` | 2026-07-11 | `All-Hands-AI/OpenHands` |
| `langgraph/` | `55ec2f2` | `55ec2f21939ce7755e6398c11b541de8926245ee` | 2026-07-10 | `langchain-ai/langgraph` |
| `autogen/` | `027ecf0` | `027ecf0a379bcc1d09956d46d12d44a3ad9cee14` | 2026-04-06 | `microsoft/autogen` |
| `llama_index/` | `67514f6` | `67514f63410c6d4c2344e7ca14b3ea7214d0bb84` | 2026-07-08 | `run-llama/llama_index` |
| `storm/` | `fb951af` | `fb951af7744dab086e34962e9bc6fe878e145f83` | 2025-09-30 | `stanford-oval/storm` |

All nine clones were clean (no local modifications) at these revisions when the book was derived.

## The reference codebases

| Repo (folder) | What it really is | Language | Role in this book |
|---|---|---|---|
| `cline/` | IDE-integrated agent, re-architected around an embeddable agent SDK | TypeScript | The distilled in-memory agent loop; hooks; compaction strategies |
| `opencode/` | Terminal-first client/server agent (SST) | TypeScript (Bun, Effect) | Event-sourced sessions; the edit-tool replacer chain; permissions; shadow-git snapshots |
| `open-interpreter/` | Open Interpreter 1.0 = fork of OpenAI **Codex CLI** | Rust | Turn/task architecture; `apply_patch` DSL; seatbelt/Landlock sandboxing; in-repo system prompts |
| `SWE-agent/` | Princeton's benchmark-oriented agent | Python | Config-as-agent (YAML); action parsing taxonomy; history processors; the ACI idea |
| `OpenHands/` | OpenHands V1 app server (agent in external SDK) | Python | Sandbox orchestration; agent-in-sandbox topology; webhook event flow |
| `langgraph/` | Durable graph runtime for agents | Python | Pregel/BSP superstep; checkpointing; interrupts |
| `autogen/` | Actor-model multi-agent framework (Microsoft) | Python | Runtime-as-message-bus; speaker selection; handoffs |
| `llama_index/` | Retrieval/indexing framework | Python | AST-aware chunking; routers; response synthesis |
| `storm/` | Stanford's research-article pipeline | Python (DSPy) | Multi-perspective research as a fixed pipeline; simulated interviews |

## Table of contents

**Part I — The anatomy of a coding agent**

1. [Anatomy and process topologies](01-anatomy.md) — the five subsystems; where each project draws its process boundaries and why
2. [The agent loop](02-agent-loop.md) — four real loops line-by-line: Cline's in-memory runtime, opencode's event-sourced reconciler, Codex's turn/task machine, SWE-agent's requery loop
3. [Prompt assembly](03-prompt-assembly.md) — layered prompts, per-model system prompts, templates-as-config, reminder injection, prompt-cache discipline
4. [The tool runtime](04-tool-runtime.md) — registries, schemas, dispatch, truncation, parallelism, MCP
5. [Editing files](05-editing-files.md) — the highest-stakes algorithm: fuzzy patch application in three independent implementations
6. [Execution and sandboxing](06-execution-sandboxing.md) — persistent shells; seatbelt/Landlock/bubblewrap; containers; the agent-in-sandbox topology
7. [Repository cognition](07-repo-cognition.md) — why grep beats embeddings; tree-sitter; LSP in and out of the IDE
8. [Context management](08-context-management.md) — overflow detection, pruning vs. summarization, cache-aware eviction, event-sourced history

**Part II — Behavior above the loop**

9. [Planning and task decomposition](09-planning.md) — plan modes, todo artifacts, subagents, verification-gated submission
10. [Human in the loop](10-human-in-the-loop.md) — permission rulesets, approvals, mid-turn steering, undo via shadow git
11. [Multi-agent systems](11-multi-agent.md) — AutoGen's actor runtime and speaker selection; handoffs; subagents vs. chat rooms

**Part III — The general-agent substrate**

12. [LangGraph: durable execution](12-langgraph.md) — the Pregel superstep, checkpointers, interrupt/resume semantics
13. [LlamaIndex: retrieval infrastructure](13-retrieval-llamaindex.md) — code-aware chunking, routing, response synthesis — and when coding agents actually need it
14. [STORM: research pipelines](14-storm.md) — personas, simulated interviews, pipeline-shaped agency

**Part IV — Synthesis**

15. [Comparative synthesis and a build guide](15-synthesis.md) — the design-decision checklist; a minimal viable agent in pseudo-code; corrections to v1's conclusions

[Appendix: source map](appendix-source-map.md) — every load-bearing claim mapped to file and line.

## How to read this

If you read only three chapters, read **2 (the loop)**, **5 (editing)**, and **8 (context management)** — they contain the algorithms that most differentiate real agents from naive tool-calling scripts. Chapter 15 stands alone as a decision guide if you are about to build one.
