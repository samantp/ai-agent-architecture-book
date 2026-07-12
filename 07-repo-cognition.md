# Chapter 7 — Repository Cognition

How does an agent understand a million-line codebase through a 200k-token window? The answer that won — in every coding agent in this workspace — is *not* pre-built semantic indexes. It is **fast, iterative, on-demand search**, with syntax trees and language servers layered on for precision. This chapter examines that stack and explains the surprising absence of embeddings.

## 7.1 The grep-first doctrine

The doctrine appears verbatim in the tools and prompts:

- Codex system prompt: *"When searching for text or files, prefer using `rg` or `rg --files` … much faster than alternatives like `grep`."* (`open-interpreter@764a96e/codex-rs/core/gpt_5_codex_prompt.md` **[Verified]**)
- opencode ships `grep` and `glob` as first-class tools, with a description that encodes an escalation policy — `opencode@34e5809/packages/opencode/src/tool/grep.txt` **[Verified]**: *"Returns file paths and line numbers … If you need to identify/count matches, use the Bash tool with `rg` directly. Do NOT use `grep`. When you are doing an open-ended search that may require multiple rounds of globbing and grepping, use the Task tool instead."* — three tiers: structured search tool → raw ripgrep → **delegate to a subagent** so the intermediate hits burn a child's context window, not yours (Ch. 4.5).
- Cline bundles a ripgrep-based file walker with an IDE-index fallback (`cline@6309971/apps/vscode/src/services/search/file-search.ts:18-44`: `FileSearchSource = "host_index" | "ripgrep"` **[Verified]**).

Why iterated grep beats a semantic index for code: the queries agents actually issue are *symbol-shaped* (identifiers, error strings, config keys), for which exact/regex match has near-perfect precision; the loop makes search *conversational* (each result reformulates the next query — cheap with 50ms searches); and there is no index to build, stale-date, or store. The model itself is the ranking function.

The supporting cast matters as much as the search: honoring `.gitignore` plus agent-specific ignores (Cline's `cline@6309971/apps/vscode/src/core/ignore/` implements a `.clineignore` layer **[Verified structure]**), bounding result counts, and always returning `path:line` so follow-up reads can be *ranged* — every read tool here takes offset/limit, because reading whole files is how windows die (Ch. 8).

## 7.2 Syntax trees: skeletons and chunk localization

Two production uses of tree-sitter in this workspace, each solving a precise problem:

**Repo skeletons (SWE-agent `filemap`).** A ~40-line script installed in the sandbox — `SWE-agent@1132b3e/tools/filemap/bin/filemap` **[Verified]**:

```python
parser = get_parser("python")
tree = parser.parse(bytes(file_contents, "utf8"))
query = language.query("""(function_definition body: (_) @body)""")
# print the file, eliding function bodies over a threshold:
#   "def compute(a, b): ..."   ← signature survives, body replaced by ellipsis
```

One tree-sitter query turns a 2,000-line file into a 150-line map of signatures, classes, and imports — the highest-leverage tokens-per-understanding transform available. (LlamaIndex's `CodeSplitter` generalizes the same parse tree into retrieval chunks — Ch. 13.2.)

**Edit localization (OpenHands `chunk_localizer`).** `OpenHands@3949e1c/openhands/app_server/utils/chunk_localizer.py` **[Verified]** builds `Chunk` objects from a tree-sitter parse (`_create_chunks_from_tree_sitter`, `:80`) so that when an LLM produces a snippet meant to replace "somewhere in this file," the server can rank semantic units by similarity and map the snippet to real byte ranges — the same family of problem as Chapter 5's fuzzy matchers, attacked with an AST instead of line heuristics.

The correction from v1: current Cline contains **no tree-sitter integration** (v1 claimed `src/services/tree-sitter`); its bet is ripgrep + host-IDE services. Tree-sitter earns its place where there is no IDE to lean on.

## 7.3 LSP: compiler-grade cognition

The step past syntax is semantics — definitions, references, types, diagnostics — and the economical way to get it is the Language Server Protocol.

**opencode embeds a real LSP client in a terminal agent** — the strongest version of this idea in the workspace. `opencode@34e5809/packages/opencode/src/lsp/` (client, server lifecycle, per-language launch config, diagnostics) plus a model-facing tool — `opencode@34e5809/packages/opencode/src/tool/lsp.ts` **[Verified]**:

```typescript
// operations exposed to the model:
"goToDefinition" | "hover" | "workspaceSymbol"   // lsp.ts:12-14,84-89
```

The agent can ask "where is this defined?" and get the *compiler's* answer instead of grep's guess. Equally important is the passive half: diagnostics harvested after edits (`opencode@34e5809/packages/opencode/src/lsp/diagnostic.ts`) let the loop report "your edit introduced these type errors" without the model running a build — the fastest verification tier (Ch. 2, Ch. 5.5).

**Cline piggybacks on the IDE**: `cline@6309971/apps/vscode/src/integrations/diagnostics/` reads VS Code's diagnostic state (the red squiggles the user already has) **[Verified structure]** — zero extra processes, but the capability evaporates in CLI/host-less deployments, which is precisely why the SDK refactor has to treat diagnostics as a host capability (`runtime/host/`).

**Sandboxed agents** must ship language servers *into* the sandbox or do without — a real cost of the OpenHands topology (Ch. 6.3): cognition tools have to be baked into the sandbox image.

## 7.4 Freshness: the file you read is not the file on disk

An agent's context contains claims about file contents that age badly — the user edits a file mid-session, or another tool rewrites it. Cline maintains a `FileContextTracker` (`cline@6309971/apps/vscode/src/core/context/context-tracking/FileContextTracker.ts` **[Verified]**) recording when files enter context and watching for external modification, so the runtime can warn the model that its knowledge is stale rather than let it edit against a phantom copy. opencode's edit tool enforces the same invariant transactionally: **edits require a prior read**, and the file's modification time is checked against the read (`opencode@34e5809/packages/opencode/src/tool/edit.ts` reads-before-writes flow **[Verified]**). This is optimistic concurrency control, with the model as the transaction that must retry.

## 7.5 Where are the embeddings?

The most instructive *negative* result in this workspace: **none of the five coding agents builds a vector index of the repository.** No embedding calls in their repo-cognition paths; retrieval-by-embedding lives only in LlamaIndex (Ch. 13), a general framework.

Why, given that "RAG over your codebase" was the 2023–24 default pitch:

1. **Staleness.** Agents *mutate* the repo as they work; a vector index is stale after the first edit, and re-embedding on every write is cost without benefit when ripgrep is 50ms.
2. **Query shape.** "Where is `deriveSubagentSessionPermission` used?" is exact-match; embeddings *add* noise. Genuinely semantic queries ("where is auth handled?") are handled acceptably by iterated grep over naming conventions plus reading — the model bridges the semantic gap itself.
3. **The window grew.** With 200k-token contexts and file-skeleton tools, the marginal value of top-k chunk retrieval collapsed for interactive use.

Where embeddings *do* still earn a place: cross-repo/organizational search, docs and issue corpora, and retrieval over *conversation memory* — i.e., corpora that are large, mostly-static, and not greppable by symbol. Chapter 13 covers the machinery for when you are actually in that regime.

## 7.6 The cognition stack, assembled

Layered by cost and precision, with the escalation policy encoded in tool descriptions and agent habits:

```
L0  glob / file tree            name-shaped questions          ~10ms
L1  ripgrep                     symbol-shaped questions        ~50ms
L2  ranged read                 "show me this region"          ~free
L3  tree-sitter skeleton        "what's in this file/module"   ~100ms, no build
L4  LSP query                   defs/refs/types (exact)        needs server warm
L5  LSP diagnostics / build     "did my change break it"       passive / seconds
L6  subagent exploration        open-ended, multi-round        burns child context
```

A production agent needs L0–L2 on day one, earns most of its perceived intelligence at L3–L5, and keeps its context clean with L6.
