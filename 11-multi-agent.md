# Chapter 11 — Multi-Agent Systems

Two philosophies of multi-agent coexist in this workspace. The coding agents use **hierarchical subagents** — parent delegates a bounded task, child reports back (Ch. 4.5, 9.3). AutoGen builds **peer societies** — agents as actors exchanging messages, with coordination itself delegated to an LLM. This chapter examines AutoGen's machinery bottom-up, then puts the two philosophies side by side, because choosing between them is the actual architectural decision.

## 11.1 The substrate: an actor runtime, not a chat loop

AutoGen's core has no conversations in it at all. `autogen-core` is a message runtime: `SingleThreadedAgentRuntime` (`autogen@027ecf0/python/packages/autogen-core/src/autogen_core/_single_threaded_agent_runtime.py:149` **[Verified]**) processes an asyncio queue of typed envelopes:

```python
self._message_queue: Queue[PublishMessageEnvelope | SendMessageEnvelope
                           | ResponseMessageEnvelope] = Queue()      # :257
async def send_message(self, message, recipient, ...) -> Any:        # :332  RPC (future-backed)
async def publish_message(self, message, topic_id, ...) -> None:     # :387  pub/sub fan-out
```

Agents are addressed by `(type, key)` and **instantiated lazily** when a message first arrives; subscriptions map topic types to agent types. The same programming model scales out — `autogen@027ecf0/protos/agent_worker.proto` and `cloudevent.proto` **[Verified]** define a gRPC worker protocol with CloudEvents envelopes, so a "team" can span processes and languages (the `dotnet/` tree implements the same runtime for .NET). The layering claim: *conversations are a library feature* (`autogen-agentchat`) *built on a distributed-actor substrate* — which is why AutoGen teams can be paused, distributed, and observed like any message-driven system.

## 11.2 Coordination by LLM: the selector loop

`SelectorGroupChat` implements "who speaks next?" as an LLM call with defenses. From `_selector_group_chat.py:152-300` **[Verified]**, the full policy stack in order:

1. **Deterministic override first**: an optional `selector_func` picks the speaker programmatically (return `None` to fall through) — hard-code the transitions you know, delegate the rest.
2. **Candidate narrowing**: `candidate_func` or the `allow_repeated_speaker` rule prunes the choice set.
3. **Prompted selection**: roles rendered one-per-line from participant *descriptions*, then the selector prompt (`{roles}`, `{participants}`, `{history}`) — model-family aware (system message for OpenAI-family, user message otherwise, `:241-245`).
4. **A validation-and-retry loop** treating the LLM as an unreliable parser of its own output:

```python
# :248-300 (abridged)
while num_attempts < max_attempts:
    response = await self._model_client.create(select_speaker_messages)
    mentions = self._mentioned_agents(response.content, self._participant_names)
    if len(mentions) == 0:
        feedback = f"No valid name was mentioned. Please select from: {participants}."
    elif len(mentions) > 1:
        feedback = "Expected exactly one name to be mentioned. ..."
    elif agent_name == self._previous_speaker and not allow_repeated:
        feedback = "Repeated speaker is not allowed, ..."
    else:
        return agent_name                    # valid
    select_speaker_messages.append(UserMessage(content=feedback, ...))
```

The same requery-with-feedback shape as SWE-agent's format errors (Ch. 2.2) — a structural constant of LLM systems: **every place an LLM output feeds a machine, wrap it in validate → explain → retry with a budget.**

Two cheaper siblings share the manager interface: `RoundRobinGroupChat` (fixed rotation — no LLM), and `SwarmGroupChatManager`, where the next speaker is dictated by the last `HandoffMessage` (`_swarm_group_chat.py:52-62` **[Verified]**) — handoffs emitted by agents as tool calls, i.e., *routing decided by the workers instead of a supervisor* (the OpenAI Swarm pattern).

## 11.3 Termination as a first-class, composable object

Group chats do not decide when to stop; **conditions** do — `autogen@027ecf0/python/packages/autogen-agentchat/src/autogen_agentchat/conditions/_terminations.py` **[Verified]** ships `StopMessageTermination`, `MaxMessageTermination`, `TextMentionTermination`, `TokenUsageTermination`, `TimeoutTermination`, `HandoffTermination`, `ExternalTermination`, `FunctionalTermination` — combinable with `|` and `&`. Externalizing the stop rule from agent logic is the design to copy: coding agents hard-code their stop conditions in the loop (Ch. 2); a multi-agent system cannot, because no single agent sees the whole picture — so it must be policy *over the event stream*.

## 11.4 The orchestrator pattern: Magentic-One's two ledgers

`MagenticOneGroupChat` is AutoGen's answer to open-ended tasks, and its orchestrator prompts encode the most explicit anti-stall machinery in this workspace (`autogen@027ecf0/python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/_magentic_one/` **[Verified]**):

- A **task ledger** — facts gathered, plan, team roster — maintained by dedicated prompts (`_get_task_ledger_facts_prompt`, `_plan_prompt`, with *update* variants for re-planning — `_magentic_one_orchestrator.py:107-119`).
- A **progress ledger** evaluated every round as structured JSON (`_prompts.py:114-118` **[Verified]**):

```python
is_request_satisfied:   bool     # done?
is_in_loop:             bool     # are we repeating ourselves?
is_progress_being_made: bool     # net forward motion?
next_speaker:           str
instruction_or_question: str     # concrete directive for that speaker
```

- A **stall counter** (`max_stalls`, `_n_stalls` — `:72-99`): consecutive rounds with loop-detected/no-progress increment it; exceeding the budget triggers *re-planning* — the task ledger is rewritten with lessons learned, and the team restarts on the new plan.

This is the supervisor design distilled: the orchestrator never does object-level work; it *senses* (progress ledger), *directs* (per-round instruction to one worker), and *re-plans on stall*. Compare Ch. 2's single-agent loops, which handle the same failure (spinning without progress) with iteration caps and duplicate-detection — the ledger is that machinery made explicit and LLM-legible.

## 11.5 Subagent trees vs. peer societies

| | Subagent tree (opencode/Cline/Codex) | Peer society (AutoGen) |
|---|---|---|
| Topology | parent→child call tree | arbitrary graph over a message bus |
| Coordination | parent's own reasoning | selector LLM / handoffs / orchestrator |
| Context | child gets a *fresh, narrow* window; returns a summary | shared transcript (per-team), grows monotonically |
| Permissions | derived, monotonically narrower (Ch. 10.1) | per-agent tool sets; no inherent narrowing |
| Failure containment | child failure = one bad tool result | one agent's confusion enters the shared transcript |
| Determinism | high (parent controls) | low-to-medium (LLM routing) |
| Cost | 1 extra session per delegation | selector/orchestrator calls **every round** |

The coding agents' choice of trees is not conservatism — it follows from their constraints: the filesystem is a shared mutable resource (concurrent peer writers would need the transaction system nobody has built — Ch. 4.2's `RwLock` is the honest state of the art), context economics reward summarize-and-return over shared transcripts, and security demands monotone permission narrowing. Peer societies earn their overhead when the *task itself* is heterogeneous and role-shaped (research + browse + code + review with genuine back-and-forth), when no single agent should hold the full context, or when components are owned by different teams and meet only at the message bus — which is also the regime where AutoGen's distributed runtime stops being over-engineering.

The synthesis position (visible in Codex's `agent_jobs`/`multi_agents` modules and Magentic-One alike): **a strong single-agent loop is the unit; multi-agent is an orchestration layer above it, added when the task demands it — not the default substrate.**
