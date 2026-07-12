# Chapter 3 — Prompt Assembly

The prompt is not a string; it is a **render pipeline** run before every model call. This chapter follows what actually gets rendered, in what order, and why — from four codebases that made four different sets of choices.

## 3.1 The layers

Every system here assembles the request from the same layer stack, top (most static) to bottom (most volatile):

1. **Base system prompt** — identity, working style, output rules.
2. **Capability schemas** — tool definitions (native tool-calling JSON) or textual protocol descriptions.
3. **Environment preamble** — OS, cwd, project/AGENTS.md instructions, date.
4. **Durable user/project instructions** — rules files, skills, memory.
5. **Conversation history** — messages and tool results (already budgeted, Ch. 8).
6. **Ephemeral injections** — reminders, error-correction templates, plan-mode notices.

The ordering is not aesthetic — it is **cache discipline**. Providers cache prompt prefixes; every byte that changes invalidates everything after it. Static-first ordering is what makes iteration N+1 of a 50-iteration session cheap. This constraint shows up as concrete code below (§3.5).

## 3.2 Per-model system prompts (opencode)

The single most instructive fact in this chapter: opencode does not have *a* system prompt — it has one per model family, dispatched at runtime. `opencode@34e5809/packages/opencode/src/session/system.ts:27` **[Verified]**:

```typescript
export function provider(model: Provider.Model) {
  if (model.api.id.includes("muse-spark")) return [PROMPT_META]
  ...
  if (model.api.id.includes("gemini-"))  return [PROMPT_GEMINI]
  if (model.api.id.includes("claude"))   return [PROMPT_ANTHROPIC]
  if (model.api.id.toLowerCase().includes("kimi")) return [PROMPT_KIMI]
  return [PROMPT_DEFAULT]
}
```

with the prompts as checked-in text files: `opencode@34e5809/packages/opencode/src/session/prompt/anthropic.txt`, `gpt.txt`, `gemini.txt`, `codex.txt`, `kimi.txt`, `beast.txt`, `plan.txt`, `plan-reminder-anthropic.txt`, … **[Verified]** Two architectural lessons:

- **Model-agnosticism is a lie at the prompt layer.** The provider abstraction can unify the transport (Chapter 4), but behavioral instructions that make Claude agentic make GPT verbose, and vice versa. Systems that claim one prompt fits all models are leaving performance on the table; opencode budgeted for divergence up front.
- **Prompts are assets, not string literals.** Checked in as `.txt`, they get diffs, blame, and review like code. Codex does the same one step further — its prompts are versioned *per model generation* (`open-interpreter@764a96e/codex-rs/core/gpt_5_codex_prompt.md`, `gpt_5_1_prompt.md`, `gpt-5.1-codex-max_prompt.md`… **[Verified]**).

## 3.3 What production system prompts actually say (Codex)

Because Codex's prompts ship in-repo, we can quote a real one instead of speculating. From `open-interpreter@764a96e/codex-rs/core/gpt_5_codex_prompt.md` **[Verified]**, condensed:

> - When searching for text or files, prefer using `rg` or `rg --files` … much faster than alternatives
> - Try to use `apply_patch` for single file edits … Do not use apply_patch for changes that are auto-generated … or when scripting is more efficient
> - You may be in a dirty git worktree. NEVER revert existing changes you did not make …
> - **NEVER** use destructive commands like `git reset --hard` … unless specifically requested
> - Skip using the planning tool for straightforward tasks (roughly the easiest 25%). Do not make single-step plans.
> - If the user asks for a "review", default to a code review mindset: prioritise identifying bugs, risks, behavioural regressions …

Note what these are: not personality, but **policy compensating for known failure modes** (reverting user changes, destructive git commands, degenerate one-item plans, reviews that summarize instead of finding bugs). Production system prompts are accumulated incident postmortems. Two details deserve special attention:

- *"Do not waste tokens by re-reading files after calling `apply_patch` on them. The tool call will fail if it didn't work."* (`prompt_with_apply_patch_instructions.md:143`) — a prompt line that only works because the *tool contract* backs it (fail-closed edits, Chapter 5). Prompt and runtime are co-designed; neither stands alone.
- The apply-patch prompt embeds a **full BNF grammar** of the edit format (§ of `prompt_with_apply_patch_instructions.md:279+`). When a tool's payload is a DSL rather than JSON, the schema must be taught in prose.

## 3.4 The agent as configuration (SWE-agent)

SWE-agent pushes layering to its logical extreme: the entire behavioral surface is a YAML file. `SWE-agent@1132b3e/config/default.yaml` is 70 lines and defines the *whole agent* **[Verified]**:

```yaml
agent:
  templates:
    system_template: |-
      You are a helpful assistant that can interact with a computer to solve tasks.
    instance_template: |-
      <uploaded_files>{{working_dir}}</uploaded_files>
      I've uploaded a python code repository ... Consider the following PR description:
      <pr_description>{{problem_statement}}</pr_description>
      ...
      1. As a first step, it might be a good idea to find and read code relevant ...
      2. Create a script to reproduce the error and execute it ...
      3. Edit the sourcecode of the repo to resolve the issue
      4. Rerun your reproduce script and confirm that the error is fixed!
      5. Think about edgecases ...
    next_step_template: |-
      OBSERVATION:
      {{observation}}
    next_step_no_output_template: |-
      Your command ran successfully and did not produce any output.
  tools:
    env_variables: { PAGER: cat, GIT_PAGER: cat, PIP_PROGRESS_BAR: 'off', TQDM_DISABLE: '1', ... }
    bundles: [ {path: tools/registry}, {path: tools/edit_anthropic}, {path: tools/review_on_submit_m} ]
    parse_function: { type: function_calling }
  history_processors:
    - type: cache_control
      last_n_messages: 2
```

Observations a designer should take away:

- **The famous system prompt is one sentence.** The work happens in the *instance template* — a per-task user message with an explicit 5-step recipe (find → reproduce → fix → re-verify → edge cases). Research finding embedded in config: recipes near the task beat personas at the top.
- **`next_step_no_output_template` exists** because models interpret empty tool output as failure. Every mature runtime grows this ("Command ran successfully with no output" appears nearly verbatim in Cline and Codex too).
- **Environment variables are prompt engineering.** `PAGER=cat`, `TQDM_DISABLE=1` — killing pagers and progress bars is context-window hygiene at the source, cheaper than truncating downstream.
- Swap `parse_function` from `function_calling` to `thought_action` and the same agent runs on models without native tool calling — the parser taxonomy in `SWE-agent@1132b3e/sweagent/tools/parsing.py:52-574` (**[Verified]**: `ActionParser`, `ThoughtActionParser` for ```-fenced actions, `XMLThoughtActionParser`, `FunctionCallingParser`, `JsonParser`, …) is a complete catalog of every tool-call transport the field has used. The protocol is a pluggable serialization detail, not the architecture.

## 3.5 Ephemeral injections and cache discipline

**Reminders.** Systems inject synthetic messages the user never typed: opencode's `SessionReminders.apply` runs in the loop right before each model call (`opencode@34e5809/packages/opencode/src/session/prompt.ts:1180`) and its `opencode@34e5809/packages/opencode/src/session/reminders.ts` module manages them **[Verified]**; Codex records time reminders and rollout-budget warnings as history items at safe points (`open-interpreter@764a96e/codex-rs/core/src/session/time_reminder.rs`, `open-interpreter@764a96e/codex-rs/core/src/session/rollout_budget.rs`) **[Verified]**; Cline's completion reminders convert premature stops into continued work (Ch. 2.1). Design rule visible in all three: **injections are appended near the tail, marked as system-origin, and never claim to be the user** — models treat "the environment reports X" differently from fabricated user speech, and cache prefixes survive because nothing upstream changed.

**Error templates as prompts.** SWE-agent's requery templates (Ch. 2.2) and Cline's escalating error responses (`cline@6309971/apps/vscode/src/core/prompts/responses.ts` **[Verified]** — after three consecutive failures of a file write, the template hard-bans the tool and dictates an alternative strategy) are prompt assembly on the failure path. Treat correction messages as templates with state (failure counts, context usage), not ad-hoc strings.

**Cache control, explicitly.** SWE-agent ships a history processor whose only job is cache annotation — `SWE-agent@1132b3e/sweagent/agent/history_processors.py:261` `CacheControlHistoryProcessor` sets Anthropic `cache_control` breakpoints on the last N messages **[Verified]** — and its eviction processor documents the interaction: `LastNObservations` has a `polling` parameter that batches evictions ("keep between `n` and `n+polling` observations") because *every* eviction rewrites history and invalidates the cache (`history_processors.py:117-122` **[Verified]**). This is the clearest statement in the workspace of a rule every agent must obey: **context edits are cache invalidations; batch them.**

## 3.6 Assembly order, concretely

Pulling it together, the per-iteration render pipeline of a modern coding agent looks like:

```
render_request(session, model):
    sys  = base_prompt_for(model_family)            # checked-in asset (§3.2)
         + environment_block(os, cwd, date)         # changes rarely
         + project_instructions(AGENTS.md, rules)   # changes rarely
    tools = schemas_for(enabled_tools, model)       # stable within a session
    hist  = history.for_prompt(modalities)          # already budgeted (Ch. 8)
    hist += pending_reminders()                     # tail injections only
    mark_cache_breakpoints(sys, tools, hist[-N:])   # provider-specific
    return (sys, tools, hist)
```

Anything that varies per-iteration belongs at the tail; anything that varies per-session belongs after the per-install constants; and every layer should be a reviewable artifact, not a string concatenated in code.
