# Chapter 2 — The Agent Loop

Four production loops, four different answers to the same three questions: *where does loop state live?* *how does the loop decide it is done?* *what happens when something goes wrong mid-iteration?*

## 2.1 Cline: the distilled in-memory loop

Cline's SDK contains the cleanest statement of the canonical loop. `AgentRuntime.execute()` — `cline@6309971/sdk/packages/agents/src/agent-runtime.ts:570` **[Verified]**, abridged to its skeleton:

```typescript
// agent-runtime.ts:604 (abridged; error/event plumbing elided)
while (this.config.maxIterations === undefined ||
       this.state.iteration < this.config.maxIterations) {
    this.throwIfAborted();
    this.state.iteration += 1;

    const { message, finishReason } = await this.generateAssistantMessage();
    const toolCalls = message.content.filter(p => p.type === "tool-call");
    this.state.messages.push(message);

    if (toolCalls.length === 0) {
        // Model produced plain text. Before accepting that as "done",
        // give configured reminders a chance to push it back to work.
        const reminders = this.getCompletionReminderMessages();
        if (reminders.length > 0) {
            for (const r of reminders) await this.addUserReminderMessage(r);
            continue;
        }
        return this.finishRun("completed", message);
    }

    const toolMessages = await this.executeToolCalls(toolCalls);
    this.state.messages.push(...toolMessages);

    // Some tools (attempt_completion-style) terminate the run themselves.
    const terminal = this.findCompletingToolMessage(toolCalls, toolMessages);
    if (terminal) return this.finishRun("completed", message, textFromToolMessage(terminal));
}
throw new Error(`Agent runtime exceeded maxIterations (...)`);
```

Design points worth stealing:

- **Two distinct stop signals.** "No tool calls" (implicit stop) and "a completion tool ran" (explicit stop). Implicit stop is *contested*: the `getCompletionReminderMessages()` hook lets the host inject "you're not done — check your todo list" messages and `continue`, converting a premature stop into another iteration. Explicit completion tools give the model a way to return a structured final answer.
- **Everything observable is an event.** Every state change emits (`run-started`, `turn-started`, `message-added`, `assistant-message`, `turn-finished`, `run-finished`/`run-failed`) with a full state snapshot attached — the UI protocol falls out of the loop for free.
- **Hooks are the extension surface**: `beforeRun/afterRun`, `beforeModel/afterModel`, `beforeTool/afterTool` (`agent-runtime.ts:758-1216`). Approval is just a privileged hook (`requestToolApproval`, line 1269). This is the same shape as Claude Code's hooks and opencode's plugin bus — a convergent API.
- **Failure classification at the end** (`agent-runtime.ts:717-752`): one `catch` block maps any error to `aborted` (user/controlled stop) vs `failed`, runs `afterRun` hooks either way, and returns a structured `AgentRunResult` rather than throwing to the host.

State lives in `this.state.messages` — a mutable array. Persistence, resumption, and multi-client viewing are explicitly the host's problem.

## 2.2 SWE-agent: the requery loop and an error taxonomy

SWE-agent's loop (Python, synchronous) is `step()` around `forward_with_handling()` — `SWE-agent@1132b3e/sweagent/agent/agents.py:1006-1170` **[Verified]**. Its contribution is not the loop shape but the most explicit **error taxonomy** in this workspace. `forward()` raises; each exception class gets a distinct policy:

```python
# agents.py:1106 (abridged)
n_format_fails = 0
while n_format_fails < self.max_requeries:
    try:
        return self.forward(history)
    # --- errors that requery the model with a templated correction ---
    except FormatError as e:            # unparseable thought/action
        n_format_fails += 1
        history = handle_error_with_retry(e, self.tools.config.format_error_template, ...)
    except _BlockedActionError as e:    # command on the blocklist
        n_format_fails += 1
        history = handle_error_with_retry(e, ...blocklist_error_template, ...)
    except BashIncorrectSyntaxError as e:  # shellcheck rejected the command
        n_format_fails += 1
        history = handle_error_with_retry(e, self.templates.shell_check_error_template, ...)
    except ContentPolicyViolationError:
        n_format_fails += 1             # just resample
    # --- errors that end the attempt but salvage work ---
    except _TotalExecutionTimeExceeded:
        return handle_error_with_autosubmission("exit_total_execution_time", ...)
```

Three ideas here generalize:

1. **Validate before executing.** A bash action is run through syntax checking *before* it touches the environment; a failure becomes a conversational correction ("your command had a syntax error: …"), not a wasted execution. Cheap static checks in front of expensive/dangerous dynamic ones.
2. **Budget the corrections.** `max_requeries` bounds the "model emits garbage → we scold it → it emits the same garbage" loop, the classic runaway failure of naive agents.
3. **Salvage on hard failure.** `handle_error_with_autosubmission` extracts whatever patch exists in the environment and submits it as the result with an explicit exit status — for a benchmark agent, a partial diff scores better than nothing, and the exit status keeps the books honest. The general lesson: *terminal error paths should still produce a well-formed artifact*.

Note also `forward()` attaches the partially-built step to the exception (`e.step = step`, `agents.py:1059`) so error handlers can render correction templates with the model's actual output — exceptions as data carriers, not just control flow.

## 2.3 opencode: the loop as a state reconciler

opencode inverts the Cline design. The loop holds **no in-memory conversation state at all**. Each iteration re-reads the persisted message log and decides what the session needs next — `opencode@34e5809/packages/opencode/src/session/prompt.ts:1081` **[Verified]**, abridged:

```typescript
// prompt.ts:1088 (abridged)
while (true) {
    // 1. Rebuild the world from the store (compacted messages filtered out)
    let msgs = yield* MessageV2.filterCompactedEffect(sessionID)
    const { user: lastUser, assistant: lastAssistant, finished: lastFinished, tasks } =
        MessageV2.latest(msgs)

    // 2. Exit condition: last assistant message finished, without tool calls,
    //    and it came after the last user message
    if (lastAssistant?.finish && !["tool-calls"].includes(lastAssistant.finish)
        && !hasToolCalls && lastUser.id < lastAssistant.id) break

    // 3. Work scheduling: pending tasks encoded IN the message log
    const task = tasks.pop()
    if (task?.type === "subtask")    { yield* handleSubtask({...}); continue }
    if (task?.type === "compaction") {
        const result = yield* compaction.process({...})
        if (result === "stop") break; else continue
    }
    // 4. Overflow check → schedule compaction as a task, then loop again
    if (lastFinished && (yield* compaction.isOverflow({ tokens: lastFinished.tokens, model }))) {
        yield* compaction.create({ sessionID, auto: true }); continue
    }
    // 5. Otherwise: run one LLM step (stream, execute tools, persist parts)
    ...
}
```

The consequences are worth spelling out:

- **Crash recovery is free.** Kill the process anywhere; on restart the reconciler reads the log and continues — exactly the Kubernetes-controller pattern (desired state: "last user message answered"; observed state: the log).
- **Compaction and subagent runs are not special control flow** — they are *messages in the log* (a user message with a `compaction` part; an assistant summary with `summary: true`). The loop discovers them like any other work item. This keeps exactly one writer (the loop) and one source of truth (the log). Chapter 8 details the compaction algorithm.
- **Multi-client UIs subscribe to the same log.** The TUI, web UI, and `opencode run` CLI render one session because the session *is* the log.
- The cost: subtle exit conditions. Step 2 above must distinguish "assistant finished for real" from "provider said `stop` but the message contains tool calls anyway" — a real providers-lie-sometimes workaround, handled with an orphaned-interrupted-tool check (`prompt.ts:1100-1130`).

Cline persists *after* state changes; opencode persists *as* the state change. If you need one sentence to compare loop architectures, that is the sentence.

## 2.4 Codex: turns, tasks, and mid-turn steering

Codex (the `open-interpreter@764a96e/codex-rs` clone) splits the loop into two levels. A **task** loops over **turns**; a turn loops over **sampling requests**. `open-interpreter@764a96e/codex-rs/core/src/tasks/regular.rs:73` **[Verified]**:

```rust
loop {
    let last_agent_message = run_turn(sess, ctx, ..., next_input, ...).await?;
    if !sess.input_queue.has_pending_input(&sess.active_turn).await {
        return Ok(last_agent_message);
    }
    next_input = Vec::new();   // queued user input becomes the next turn's input
}
```

And inside `run_turn` (`open-interpreter@764a96e/codex-rs/core/src/session/turn.rs:227` **[Verified]**, heavily abridged):

```rust
loop {
    // user messages typed while the model was running
    let pending_input = sess.input_queue.get_pending_input(&sess.active_turn).await;
    if run_hooks_and_record_inputs(&sess, &turn_context, &pending_input).await { break; }

    // periodic context injections, recorded as history items
    rollout_budget::maybe_record_reminder(...);      // budget warnings
    time_reminder::maybe_record_current_time_reminder(...);

    let input: Vec<ResponseItem> = sess.clone_history().await
        .for_prompt(&turn_context.model_info.input_modalities);
    let result = run_sampling_request(..., input, cancellation_token.child_token()).await;

    // post-sampling token accounting drives auto-compaction *mid-turn*
    let token_status = context_window_token_status(...).await;
    if needs_follow_up && token_limit_reached {
        run_auto_compact(..., CompactionReason::ContextLimit, CompactionPhase::MidTurn).await?;
        continue;
    }
    if !needs_follow_up { ...; break; }
}
```

Distinctive decisions:

- **Mid-turn user input is queued, not injected.** The user can keep typing; input drains at a *safe point* (top of the loop), so the model never sees a half-delivered instruction and tool executions are never interleaved with new directives. Compare Cline, which aborts and restarts, and opencode, which enqueues prompts into the log.
- **Compaction is a first-class phase inside the turn** (`CompactionPhase::MidTurn`), not a between-turns maintenance job — necessary because a single agentic turn (dozens of tool calls) can overflow the window on its own. The comment at `turn.rs:347` is refreshingly candid: *"as long as compaction works well in getting us way below the token limit, we shouldn't worry about being in an infinite loop."*
- **Cancellation is hierarchical**: `cancellation_token.child_token()` hands each sampling request a child of the turn's token, so aborting a turn cancels its in-flight request and tool executions transitively — the structured-concurrency answer to the zombie-process problem.
- **Turn-scoped bookkeeping** — `turn_diff_tracker.rs` accumulates the net file diff of a whole turn for review UIs; `turn.rs` traces per-turn token status (`auto_compact_scope_tokens`, `tokens_until_compaction`) — is only possible because "turn" is a reified object, not an emergent property of the message array.

## 2.5 Synthesis: choosing a loop architecture

| | Cline SDK | SWE-agent | opencode | Codex |
|---|---|---|---|---|
| Loop state | in-memory array | in-memory + trajectory file | **persisted message log** | session history + queues |
| Stop: implicit | no tool calls (contested by reminders) | — (parser demands an action every step) | finish reason + no tool calls | `!needs_follow_up` |
| Stop: explicit | completion tool | `submit` tool (+ review gate, Ch. 9) | — | — |
| Hard budgets | `maxIterations` | `max_requeries`, total exec time | per-agent `steps` | token budgets + turn limits |
| Mid-run user input | abort + new run | n/a (batch) | enqueued into log | **drained at safe points** |
| Malformed output | error message → retry | **typed exceptions → templated requery** | provider-quirk patches | sampling-level retries |
| Crash recovery | host's job | rerun instance | **free (reconciler)** | rollout file reconstruction |

Rules of thumb that fall out:

1. **Pick your state substrate first** — everything else (resumability, multi-client, subagents, undo) follows from whether history is an in-memory array or a persisted log.
2. **Treat "model stopped without finishing" as a first-class case.** All four systems have machinery for it (completion reminders, requery templates, orphaned-tool checks). Your agent will need it on day one.
3. **Classify errors into "requery," "retry," "salvage," and "abort" buckets** explicitly, each with its own budget. SWE-agent's exception taxonomy is the reference implementation.
4. **Make loop progress observable as events** with enough payload that a UI needs no other channel.
