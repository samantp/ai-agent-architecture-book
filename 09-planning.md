# Chapter 9 — Planning and Task Decomposition

v1 of this book framed planning as a future "dedicated subsystem with DAG tracking." The production systems went a different way: **planning is implemented as tools and modes that shape the model's own behavior**, not as an external planner. What ships today is four mechanisms — plan modes, todo artifacts, subagent decomposition, and verification gates — each cheap, each grounded in this workspace.

## 9.1 Plan mode: a permission profile plus a prompt

opencode implements "planning" as an *agent mode* whose essence is **withheld write permissions plus a different system prompt** (`session/prompt/plan.txt`, `plan-mode.txt` **[Verified]**), with mode transitions mediated by tools the model itself calls:

- `plan-enter` — *"Use this tool to suggest switching to plan agent when the user's request would benefit from planning before implementation … This tool will ask the user if they want to switch."* (`tool/plan-enter.txt` **[Verified]**)
- `plan-exit` — presents the plan and requests approval to start implementing.

Note the design: the *model* proposes mode changes; the *user* approves them; the *permission system* enforces them (a plan-mode agent's ruleset denies edits — Ch. 10). There is no plan datastructure at all: the plan is prose in the conversation, and the mode boundary is what forces it to be produced *before* mutations begin. A `plan-reminder-anthropic.txt` injection nags the model to keep the plan updated **[Verified]** — behavioral maintenance, again by prompt.

Cline's Plan/Act split is the same idea hosted in the extension UI; SWE-agent bakes the plan into the *instance template* itself — the 5-step recipe (find → reproduce → fix → re-verify → edge cases) of `config/default.yaml` (Ch. 3.4) is a fixed plan the model instantiates rather than invents. For a constrained domain (bench-style bug fixing), a canned plan beats a generated one.

## 9.2 Todo lists: externalized working memory with a state machine

The todo tool looks trivial and is not. opencode's `todowrite` description (`tool/todowrite.txt` **[Verified]**) specifies a real protocol:

> - The task requires 3+ distinct steps … Use proactively
> - Mark `in_progress` (only ONE at a time) before working
> - Mark `completed` only after the required work is actually done, **including any required verification. Never based on intent.**
> - If blocked or partial, keep it `in_progress` and add a follow-up todo describing the blocker
> - Preserve user-provided commands verbatim (flags, args, order)

States: `pending → in_progress → completed | cancelled`, exactly one `in_progress`. What this actually is: **a loop-invariant maintained in context**. Because the todo list is re-rendered near the tail of every request (and survives compaction — plan/todo state is preserved by summarizers, Ch. 8.3), it functions as the model's program counter across dozens of iterations: after any digression, the single `in_progress` item tells the model where it was. The "never based on intent" rule exists because models mark work done when they *decide* to do it; coupling completion to verification is anti-hallucination engineering in prose.

Codex's plan tool guidance adds the calibration rules (`gpt_5_codex_prompt.md` **[Verified]**): *"Skip using the planning tool for straightforward tasks (roughly the easiest 25%). Do not make single-step plans. When you made a plan, update it after having performed one of the sub-tasks."* — thresholds against the twin failure modes of ceremonial planning (planning trivial work) and stale plans (write-once todo lists). Cline's focus-chain (`core/task/focus-chain/`) keeps the markdown-checklist variant of the same organ **[Verified structure]**.

The general principle: **externalize state the model must not lose, into an artifact the runtime re-injects**. The todo list is one instance; AGENTS.md instructions and memory files are the same pattern at session and installation scope.

## 9.3 Decomposition: subagents, not DAGs

Where v1 predicted dependency graphs, production systems decompose with **subagents** (Ch. 4.5): the parent describes a bounded sub-task; a child session with its own context window and a *narrower* permission/tool set executes it; only the final report re-enters the parent's context.

This is planning-relevant for a precise reason: **decomposition is context-budget management**. A parent that greps for 30 minutes accumulates 100k tokens of dead intermediate state; a parent that delegates gets back 2k tokens of conclusions. opencode's grep tool description literally routes open-ended searches to the task tool (Ch. 7.1); its subagent schema supports resumable children (`task_id`) and background execution with explicit anti-polling etiquette (`tool/task.ts` **[Verified]**).

What none of these systems has — still — is dependency-ordered parallel task scheduling *by the runtime*. Parallelism exists (background subagents, Codex's `agent_jobs` handlers and multi-agent session support — `tools/handlers/agent_jobs.rs`, `session/multi_agents.rs` **[Verified structure]**) but ordering is left to the model's judgment over a flat todo list. The pragmatic reading: for interactive coding, the model *is* the scheduler, and a DAG engine adds machinery exactly where flexibility is most valuable. (LangGraph serves the opposite regime — Ch. 12 — where the workflow is known and the machine should enforce it.)

## 9.4 Self-monitoring tools

Codex hands the model instruments alongside actuators — `core/src/tools/handlers/` **[Verified structure]** includes `get_context_remaining` (the model can query how much window it has left and plan its remaining moves accordingly), `wait_for_environment`, `sleep`, `request_user_input`, and `tool_search` (deferred tool loading — the registry-token-cost countermeasure of Ch. 4.4). Planning quality improves when the model can *sense* its budgets instead of being surprised by them; this pairs with the runtime-side budget reminders of Ch. 8.1.

## 9.5 Verification-gated completion

The strongest planning mechanism in this workspace is a gate at the *end*: SWE-agent's `review_on_submit_m` bundle (wired in `config/default.yaml` **[Verified]**) intercepts the model's `submit` and answers with a templated challenge:

> 1. If you made any changes … please run the reproduction script again …
> 2. Remove your reproduction script …
> 3. If you have modified any TEST files, please revert them … Use below `<diff>` to find the files you need to revert.
> 4. Run the submit command again to confirm.
>
> `<diff>{{diff}}</diff>`

The submit tool becomes a two-phase commit: first call returns a review checklist *with the actual diff embedded*; only the second call lands. Every one of Chapter 2's loops can host this pattern (Cline's terminal-tool check, opencode's plan-exit approval), and it generalizes to any agent with a completion action: **make "done" a claim the runtime disputes once, with evidence attached.** Cheap, model-agnostic, and it catches the classic end-of-task failures (leftover scratch files, modified tests, unverified fixes) at the moment the model is most prone to declare victory.

## 9.6 Design rules

1. Plan mode = prompt + permission profile + user-approved transitions. No planner component needed.
2. Todo list = re-injected loop invariant with a state machine; enforce "one in-progress" and "completed requires verification" in the tool description.
3. Calibrate planning ceremony (skip the easy 25%; no single-step plans; update after each sub-task).
4. Decompose for *context economics*: subagents return conclusions, not transcripts; give children narrower permissions and resumability.
5. Give the model gauges (context remaining) — plans improve when budgets are observable.
6. Gate completion with a runtime-generated review that embeds evidence (the diff).
