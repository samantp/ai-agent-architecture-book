# Chapter 10 — Human in the Loop

"Human in the loop" is usually presented as a UX topic. In these codebases it is three distinct *technical* subsystems: a **policy engine** deciding which actions need consent, a **steering channel** letting humans redirect a running loop, and an **undo mechanism** making agent mistakes cheap. Get these right and users grant more autonomy, not less — the paradox at the center of agent product design.

## 10.1 The policy engine: rules, not booleans

The naive design — a per-tool `requiresApproval` flag — collapses immediately: `bash` is sometimes `ls` and sometimes `rm -rf`, an edit inside the workspace differs from one in `~/.ssh`. Production systems converge on **pattern-matched rulesets evaluated per action**.

opencode's evaluator is the cleanest — `packages/opencode/src/permission/evaluate.ts:28` **[Verified]**:

```typescript
export function evaluate(permission: string, pattern: string,
                         ...rulesets: PermissionV1.Ruleset[]): PermissionV1.Rule {
  return rulesets
    .flat()
    .findLast(rule => Wildcard.match(permission, rule.permission)
                   && Wildcard.match(pattern, rule.pattern))
    ?? { action: "ask", permission: "*", pattern: "*" }   // default: ask
}
```

Five properties worth copying:

1. **Two-dimensional matching** — a *permission* (`"edit"`, `"bash"`, `"task"`) and a *pattern* (the file path, the command) both wildcard-matched; rules like `bash: "git *" → allow`, `edit: "**/.env" → deny` fall out naturally.
2. **Last match wins** (`findLast`) — later/more-specific rulesets override earlier ones; user config overrides defaults, session grants override both, with plain array concatenation as the composition operator.
3. **Fail-closed default**: no matching rule means `ask`, not `allow`.
4. **Session-approved overlays** — an "always allow this" answer appends a rule to a session-scoped ruleset (`opencode@34e5809/packages/opencode/src/permission/index.ts:67-74` shows `ask` evaluating against `ruleset, approved` **[Verified]**), so consent accumulates *within* a session without touching durable config.
5. **Permission requests originate inside tools** via `ctx.ask(...)` (Ch. 4.1) — the tool knows the real pattern (the resolved absolute path, the canonicalized command) far better than a pre-dispatch inspector can.

Subagents inherit a **derived, never-broader** ruleset (`deriveSubagentSessionPermission`, `opencode@34e5809/packages/opencode/src/agent/subagent-permissions.ts` **[Verified]**) — the confused-deputy guard: a child task cannot be a privilege-escalation vector. Plan mode (Ch. 9.1) is just a ruleset that denies writes — the same engine implements a product feature.

Cline routes every dispatch through an approval hook (`requestToolApproval` — `agent-runtime.ts:1269`; policy logic in `cline@6309971/sdk/packages/core/src/runtime/tools/tool-approval.ts` **[Verified structure]**), which composes with hosts: the VS Code extension renders approval UI, a CI host auto-answers from config. Codex factors the problem furthest: canonicalize the command first (`command_canonicalization.rs` — so `python3 -c "..."` can't dodge a `python` rule), evaluate against Starlark exec policy (Ch. 6.2), then route residual risk to *typed* approval flows — command approval, **network approval as its own category** (`open-interpreter@764a96e/codex-rs/core/src/tools/network_approval.rs`), MCP-tool approval templates, and generic elicitation (`elicitation.rs`, `open-interpreter@764a96e/codex-rs/core/src/tools/handlers/request_user_input.rs`) **[Verified structure]**. Escalations carry evidence (the sandbox denial that prompted them), which is what makes the sandboxed-by-default posture usable.

Design consequence across all three: **an approval is a *grant* added to state, not a branch taken once** — which is why "always allow `npm test`" is expressible and why audit logs of grants matter more than logs of prompts.

## 10.2 Steering: interrupts without derailment

The second channel is redirection mid-run — rarer to get right than approval. Three models, in increasing sophistication:

- **Abort-and-restart (Cline)**: the runtime is abortable at every await (`throwIfAborted`, AbortController end-to-end — Ch. 2.1); user input starts a new run over updated history. Simple; loses in-flight work.
- **Queue into the log (opencode)**: new user messages append to the session log; the reconciler loop naturally picks them up on its next iteration (Ch. 2.3). No special interrupt machinery — steering is just more state to reconcile.
- **Queue with safe-point draining (Codex)**: typed-while-running input lands in `input_queue`, drained *only* at the top of the turn loop (`open-interpreter@764a96e/codex-rs/core/src/session/turn.rs:227-235`), and an outer task loop turns leftover queued input into the next turn (`open-interpreter@764a96e/codex-rs/core/src/tasks/regular.rs:84-87`) **[Verified]**. The model never observes a half-delivered instruction; tool sequences are never sliced by a mid-execution injection; and cancellation, when actually requested, propagates through hierarchical tokens (child per sampling request/tool call).

The safe-point pattern is the one to copy for any agent whose turns have internal structure: define the points where new human input may enter, and make everything between them atomic.

## 10.3 Undo: shadow git

Approval prevents *anticipated* damage; undo makes *unanticipated* damage cheap — and cheap undo is what lets users approve more. The convergent implementation is a **shadow git repository**: real git plumbing, pointed away from the user's `.git`.

opencode — `packages/opencode/src/snapshot/index.ts:67-81` **[Verified]**:

```typescript
gitdir: path.join(Global.Path.data, "snapshot", ctx.project.id, Hash.fast(ctx.worktree))
const args = (cmd) => ["--git-dir", state.gitdir, "--work-tree", state.worktree, ...cmd]
```

`--git-dir` lives under opencode's data directory (keyed by project + worktree hash); `--work-tree` is the user's actual tree. Snapshots are commits in this parallel history: created around mutating operations, diffable (the UI's per-turn file diffs), and restorable per-file or wholesale — without ever touching the user's own git state (their index, stashes, and hooks stay pristine; the code even consults the *user's* `.git` only to respect ignore rules, `opencode@34e5809/packages/opencode/src/snapshot/index.ts:102-131` **[Verified]**). `opencode@34e5809/packages/opencode/src/session/revert.ts` wires this to the conversation: reverting to a message restores the corresponding snapshot *and* the log position — filesystem state and conversation state roll back together **[Verified structure]**. Cline's checkpoint system (`cline@6309971/apps/vscode/src/core/controller/checkpoints/checkpointRestore.ts` **[Verified structure]**) exposes the same contract: restore workspace, conversation, or both.

That pairing is the non-obvious invariant: **restoring files without rewinding the conversation leaves the model believing its edits exist**; every subsequent action compounds the confusion. Undo must be transactional across the two stores. (This is also why Codex tracks per-turn diffs — `turn_diff_tracker.rs` — the reviewable unit of undo is the turn.)

## 10.4 The escalation ladder

Assembled, the mature systems present a coherent autonomy dial rather than a binary:

```
deny  <  ask (evidence attached)  <  session grant  <  durable rule  <  sandboxed autonomy
```

with three supporting guarantees making the upper rungs tolerable: scoped containment (Ch. 6), transactional undo (§10.3), and full observability (every event streamed to the UI — Ch. 2.1, 4.2). The product insight embedded in all three codebases: **invest in undo and evidence, and the approval prompt count can drop an order of magnitude without losing safety** — the inverse of the naive "more safety = more prompts" assumption.

## 10.5 Checklist

1. Permissions = wildcard rules over (action, pattern), last-match-wins, default `ask`, composable rulesets (defaults < user < session grants).
2. Ask from *inside* the tool with resolved arguments; canonicalize commands before matching.
3. Subagent permissions are derived and monotonically narrower.
4. Type your approvals: command, network egress, external/MCP tools, generic elicitation — different evidence, different UI.
5. Steering enters at defined safe points; in-between work is atomic.
6. Undo = shadow git, transactional across filesystem *and* conversation.
7. Stream everything; an invisible agent earns no autonomy.
