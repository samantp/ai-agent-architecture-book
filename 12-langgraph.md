# Chapter 12 — LangGraph: Durable Execution for Agents

The coding agents of Part I hand control flow to the model and engineer guardrails around it. LangGraph represents the opposite bet: **when you know the workflow, encode it as a graph and let a runtime enforce it** — with database-backed checkpoints making execution pausable, resumable, replayable, and forkable. Its engine is a faithful implementation of a 2010-era distributed-systems idea: Google's Pregel, the bulk-synchronous-parallel (BSP) graph model.

## 12.1 The BSP superstep

A compiled graph runs as repeated **supersteps**. Each superstep: determine which nodes were triggered by the last step's channel updates; run them *in parallel* against a **snapshot** of state; collect their writes; apply all writes **atomically**; checkpoint. Nodes never see each other's mid-step writes — reads come from the previous superstep's world, exactly like Pregel vertices.

The engine loop is `PregelLoop.tick()` / `after_tick()` — `langgraph/libs/langgraph/langgraph/pregel/_loop.py:599-718` **[Verified]**, abridged:

```python
def tick(self) -> bool:
    if self.step > self.stop:                       # recursion_limit
        self.status = "out_of_steps"; return False
    self.tasks = prepare_next_tasks(                 # plan phase (_algo.py)
        self.checkpoint, self.checkpoint_pending_writes,
        self.nodes, self.channels, ...,
        trigger_to_nodes=self.trigger_to_nodes,
        updated_channels=self.updated_channels)
    if not self.tasks: self.status = "done"; return False
    if not self.is_replaying and self.checkpoint_pending_writes:
        self._reapply_writes_to_succeeded_nodes(self.tasks)   # crash recovery (§12.3)
    if self.interrupt_before and should_interrupt(...):
        raise GraphInterrupt()
    return True                                      # caller executes tasks in parallel

def after_tick(self) -> None:
    writes = [w for t in self.tasks.values() for w in t.writes]
    self.updated_channels = apply_writes(            # atomic apply
        self.checkpoint, self.channels, self.tasks.values(), ...)
    ...
    self._put_checkpoint({"source": "loop"})         # durability boundary
```

State is not one dict but a set of **channels**, each with an update discipline (last-value overwrite, appending reducers à la `Annotated[list, operator.add]`, etc.) and a version counter; `trigger_to_nodes` maps channel updates to the nodes they wake. Two consequences an architect should register: **parallel fan-out is safe by construction** (writers can't race — writes merge through channel reducers at the barrier), and **a "conditional edge" is just a routing function evaluated at the barrier** — control flow lives in data, which is why it checkpoints cleanly.

## 12.2 Checkpointers: the persistence interface

Durability is one small interface — `libs/checkpoint/langgraph/checkpoint/base/__init__.py:176-300` **[Verified]**: `BaseCheckpointSaver.get_tuple(config)`, `.put(config, checkpoint, metadata, new_versions)`, `.put_writes(config, writes, task_id)`, `.list(...)` — keyed by `thread_id` (a conversation/execution identity) and `checkpoint_id` (a specific superstep). Implementations: in-memory, SQLite, Postgres (`libs/checkpoint-*`).

Because a checkpoint lands after *every* superstep, the thread's history is a **persistent chain of full states**, which yields, uniformly:

- **Resume** — re-invoke with the same `thread_id`; the loop continues from the last checkpoint.
- **Time travel** — fetch any historical `checkpoint_id`, inspect it, *fork* execution from it with modified state (`update_state`).
- **Replay/debug** — the `is_replaying` flag distinguishes re-execution over known writes from fresh work (`_loop.py:715-716`).

## 12.3 Pending writes: sub-step crash granularity

The subtle mechanism worth knowing: `put_writes` persists *individual task results within an unfinished superstep*. If 3 parallel nodes run and one crashes, the two successes are already durable; on retry, `_reapply_writes_to_succeeded_nodes` (`_loop.py:662-664` **[Verified]**) restores their writes and only the failed node re-executes. BSP's atomic barrier, without BSP's "redo the whole round" recovery cost. If you build your own graph runtime, this is the feature you will discover you need in production week one.

## 12.4 Interrupts: human-in-the-loop as an exception you resume

LangGraph's approval mechanism is dynamic interruption from *inside* a node — `libs/langgraph/langgraph/types.py:811` **[Verified]**:

```python
def node(state):
    answer = interrupt("what is your age?")     # 1st execution: raises GraphInterrupt
    return {"human_value": answer}              # resumed execution: returns the Command value
...
graph.stream({"foo": "abc"}, config)            # → {'__interrupt__': (Interrupt(value=...),)}
graph.stream(Command(resume="42"), config)      # continue, days later if you like
```

The documented semantics carry two sharp edges (docstring, `types.py:824-831` **[Verified]**): resumption **re-executes the node from its start** (everything before `interrupt()` runs again — side effects before an interrupt must be idempotent), and multiple interrupts in one node are matched to resume values **by order** — so don't reorder them across deploys while runs are parked. Static variants (`interrupt_before=[node]` at compile time) trade expressiveness for predictability. Comparing Chapter 10: opencode/Cline ask *within a running process* (the tool awaits an answer); LangGraph *parks the whole execution in a database row* — approval flows that survive process death and take days are the payoff, node re-execution is the price.

`Send` (`types.py:664`) completes the control-flow vocabulary: dynamic fan-out of N task-parameterized node invocations in one superstep (map-reduce over items discovered at runtime), their writes merging at the barrier; `Command` (`types.py:759`) lets a node both update state and `goto` a target, subsuming conditional edges for imperative tastes.

## 12.5 Where this fits against Part I

The prebuilt `create_react_agent` (`libs/prebuilt` **[Verified structure]**) is exactly Chapter 2's loop — model node ⇄ tool node with a conditional edge — re-expressed as a two-node graph, which makes the comparison precise:

- What LangGraph *adds*: free durability/time-travel per step, enforced workflow structure, safe parallel branches, park-and-resume approvals.
- What it *doesn't solve*: everything inside the model node — prompt assembly, context management, edit-tool robustness, permissions — i.e., all of Chapters 3–10. A LangGraph coding agent still needs that entire stack; the graph replaces only the `while` loop and the persistence layer.
- The trade: coding agents keep the plan in the *model* (flexible, cheap to change, hard to audit); LangGraph keeps it in the *graph* (auditable, enforceable, rigid). Use the graph where the process is the product — pipelines, compliance-gated flows, long-running approval chains; use the model-driven loop where adaptability is the product — interactive engineering work. Hybrids are natural: a LangGraph pipeline whose "implement" node hosts a full model-driven coding agent.
