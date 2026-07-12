# Chapter 4 — The Tool Runtime

A tool runtime does five jobs: **define** (schema + description), **advertise** (render schemas into the request), **dispatch** (route a parsed call to a handler), **contain** (permissions, sandbox, cancellation), and **shape** (turn raw output into something safe to append to history). This chapter covers define/advertise/dispatch/shape; containment gets Chapters 6 and 10.

## 4.1 Defining a tool: the contract

opencode's tool interface is the most explicit statement of what a tool needs from its runtime — `opencode@34e5809/packages/opencode/src/tool/tool.ts` **[Verified]**:

```typescript
export type Context<M extends Metadata = Metadata> = {
  sessionID: SessionID
  messageID: MessageID
  agent: string
  abort: AbortSignal                       // cancellation, always
  messages: SessionV1.WithParts[]          // read-only view of history
  metadata(input: { title?; metadata? }): Effect<void>   // live progress → UI
  ask(input: PermissionV1.Request): Effect<void>          // permission gate
}
```

Note what is *in the contract*: an abort signal (every tool must be cancellable), a `metadata` channel (tools stream progress to the UI without polluting the model-visible result), and `ask` (tools request permission mid-execution — e.g. the edit tool asks per-file; Chapter 10). And note the typed argument-failure path:

```typescript
// tool.ts — the canonical "rewrite the input" error
export class InvalidArgumentsError ... {
  override get message() {
    return `The ${this.tool} tool was called with invalid arguments: ${this.detail}.
Please rewrite the input so it satisfies the expected schema.`
  }
}
```

Schema-validation failure is not an exception to log — it is **a message to the model**, worded to elicit a corrected call. Every runtime here converges on this: validation errors are prompts (compare SWE-agent's `format_error_template`, Ch. 2.2).

Tool *descriptions* are checked-in text files, one per tool (`opencode@34e5809/packages/opencode/src/tool/edit.txt`, `grep.txt`, `task.txt`, … **[Verified]**), for the same reason system prompts are (Ch. 3.2): they are behavioral assets under review. Cline's SDK defines tools with variants per model family and host (`cline@6309971/sdk/packages/core/src/...`), and SWE-agent goes furthest: a tool *bundle* is a directory of executables plus YAML metadata **installed into the sandbox** (`SWE-agent@1132b3e/tools/edit_anthropic/bin/str_replace_editor` etc.), so "adding a tool" means shipping shell/Python programs onto the target box — the tool schema the model sees and the binary that runs are two views of one bundle **[Verified]**.

## 4.2 Advertising and dispatch

All four coding agents now default to native function calling (SWE-agent's `parse_function: function_calling` in `SWE-agent@1132b3e/config/default.yaml`; the others via provider SDKs), keeping the older textual protocols as fallbacks (the parser taxonomy of Ch. 3.4). Dispatch is a name→handler map everywhere; the differences are in what wraps the handler:

- **Cline**: `executeToolCalls` iterates calls *sequentially* (`agent-runtime.ts:1136-1176`), each passing through `prepareToolExecution` → `beforeTool` hooks → `requestToolApproval` → `executePreparedTool` → `afterTool` hooks **[Verified]**. Sequential-by-default is a correctness stance: tools mutate a shared filesystem.
- **Codex**: a `ToolRouter` + `ToolCallRuntime` with genuine parallelism, gated by an elegant primitive — a single `RwLock<()>`:

```rust
// codex-rs/core/src/tools/parallel.rs:100-136 [Verified]
let supports_parallel = self.router.tool_supports_parallel(&call);
let _guard = if supports_parallel {
    Either::Left(lock.read().await)    // read-safe tools share the lock
} else {
    Either::Right(lock.write().await)  // mutating tools get exclusivity
};
```

Read-only tools (file reads, greps) take the read lock and run concurrently; anything mutating takes the write lock and serializes against *everything*. Per-tool parallelism declared in the registry, enforced in one line. **[Verified]** This is the pattern to copy if you want parallel tool execution without a transaction system.

- **opencode**: tools are Effect programs; the session processor executes them within the streaming step, persisting a tool *part* whose state machine (`pending → running → completed/error`, with `metadata` updates streamed while running — `opencode@34e5809/packages/opencode/src/session/processor.ts:195` comment: *"Keep metadata streamed while running so failures retain progress detail"*) is itself the UI protocol **[Verified]**.

## 4.3 Shaping output: truncation is an interface, not an afterthought

The naive move — `output[:N] + "...truncated"` — destroys exactly the information the model may need. opencode's truncation module is the mature version — `opencode@34e5809/packages/opencode/src/tool/truncate.ts` **[Verified]**:

```typescript
export const MAX_LINES = 2000
export const MAX_BYTES = 50 * 1024
// Result: { content, truncated: false } |
//         { content, truncated: true, outputPath }   ← full output persisted
```

Oversized output is written to a **truncation directory** (7-day retention) and the model receives a head/tail *preview plus the path* — so the agent can `grep` or `read` the full output on demand. Truncation becomes a pointer swap: context holds a preview, disk holds the truth, and the existing file tools are the retrieval mechanism. Two further details: the preview direction is configurable (`head` for logs whose start matters, `tail` for build errors), and whether the hint advertises follow-up depends on the agent's own permissions (`hasTaskTool` check) — don't tell a tool-less subagent to go read a file it cannot read. **[Verified]**

SWE-agent attacks the same problem upstream (env vars killing pagers/progress bars, Ch. 3.4); Codex bounds output in its exec layer with `ExecCapturePolicy` (`open-interpreter@764a96e/codex-rs/core/src/exec.rs`) **[Verified structure]**. Do all three: suppress at the source, bound at capture, page via files.

## 4.4 Extension tools: MCP and skills

All three interactive agents are MCP clients: opencode (`opencode@34e5809/packages/opencode/src/mcp/`), Cline (`cline@6309971/apps/vscode/src/services/mcp/`), Codex (`open-interpreter@764a96e/codex-rs/codex-mcp/`, `open-interpreter@764a96e/codex-rs/core/src/mcp.rs`, with tool-approval templates in `mcp_tool_approval_templates.rs`) **[Verified structure]**. Architecturally MCP is the *define/advertise* half of the runtime outsourced over JSON-RPC: external servers contribute schemas at session start, and dispatch proxies to the server. The consequences that matter to an architect:

1. **Trust boundary.** An MCP tool executes outside your permission machinery unless you gate it explicitly — hence Codex's dedicated approval templates for MCP calls, and network approval as its own subsystem (`open-interpreter@764a96e/codex-rs/core/src/tools/network_approval.rs`).
2. **Context cost.** Every advertised tool schema is prompt tokens on *every* request. Large registries motivate deferred loading (schemas fetched on demand) — visible in this workspace as opencode's dynamic tool descriptions (`DynamicDescription` in `tool.ts`) and agent-scoped tool filtering (`opencode@34e5809/packages/opencode/src/session/prompt.ts:1061` filters tools per agent config **[Verified]**).
3. **Skills vs. tools.** A parallel extension mechanism appears in all three (opencode `opencode@34e5809/packages/opencode/src/skill/` + protected `"skill"` tool in compaction, Cline `cline@6309971/apps/vscode/src/core/context/instructions/user-instructions/skills.ts`, Codex `open-interpreter@764a96e/codex-rs/core/src/skills.rs`): a *skill* is instructions-as-data (markdown loaded into context on demand) rather than capability-as-code. Cheaper to author, no new attack surface, but consumes context instead of adding function.

## 4.5 The subagent as a tool

The most consequential "tool" in modern agents is the one that spawns another agent. opencode's `task` tool — `opencode@34e5809/packages/opencode/src/tool/task.ts` **[Verified]**:

```typescript
const BaseParameterFields = {
  description: ...,                    // 3-5 word label
  prompt: ...,                         // the sub-task
  subagent_type: ...,                  // which agent config to run
  task_id: Schema.optional(...),       // resume a previous subagent session!
  background: Schema.optional(Boolean) // async: return immediately, notify later
}
```

Three design decisions embedded in that schema:

- **Subagents are sessions.** The tool creates a child session running the same loop with a different agent config and a *derived permission set* (`deriveSubagentSessionPermission` — a child can never exceed its parent; Ch. 10). Passing `task_id` resumes the child with context intact — subagents are durable conversations, not fire-and-forget closures.
- **Background execution is a contract with the model.** The tool's static text spells out anti-patterns: *"DO NOT sleep, poll for progress, ask the task for status, or duplicate this task's work."* **[Verified]** When you add async capabilities, you must teach the model the concurrency discipline in the tool description itself — the schema is the API, the description is the semantics.
- **Isolation is the point.** A subagent burns its own context window and returns only its final message — the architectural answer to "this search will dump 50k tokens I don't want in the main session" (Ch. 8 and 11 return to this).

Cline mirrors this in `cline@6309971/apps/vscode/src/core/task/tools/subagent/` **[Verified structure]**.

## 4.6 Design checklist

1. Tool context = abort signal + progress channel + permission gate + session identity. If your `Tool` interface lacks any of these, retrofitting is painful.
2. Validation failures are model-facing prompts with a stable, instructive shape.
3. Sequential dispatch by default; parallelism only via declared read-safety (the `RwLock<()>` trick).
4. Truncate by paging to disk with a retrievable path — never by silent amputation.
5. Descriptions are reviewed assets; budget their token cost; filter the registry per agent/mode.
6. Gate MCP/external tools through the *same* permission machinery as builtins.
7. If you add a subagent tool, encode resumability (`task_id`) and concurrency etiquette from day one.
