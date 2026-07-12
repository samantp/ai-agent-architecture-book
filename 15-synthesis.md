# Chapter 15 — Synthesis: The Decision Checklist and a Build Guide

## 15.1 The honest comparison

| Axis | Cline (SDK) | opencode | Codex / OI 1.0 | SWE-agent | OpenHands V1 |
|---|---|---|---|---|---|
| Loop state | in-memory array | **persisted message log** | session + queues | in-memory + trajectory file | server-side event store |
| Process topology | embedded library, many hosts | local client/server | protocol'd core, many clients | batch CLI | **agent inside sandbox** |
| Edit mechanism | search/replace + hooks | 9-replacer chain | `apply_patch` DSL + `seek_sequence` | strict `str_replace` (no fuzz) | SDK tools + AST chunk localizer |
| Execution containment | host + approvals (+ nascent subprocess sandbox) | host + permission rules | **seatbelt / Landlock / bwrap + execpolicy + net proxy** | Docker/remote via SWE-ReX | Docker container per session |
| Context policy | basic (deterministic) + agentic (LLM) compaction | prune (40k protect) + head/tail summarize | mid-turn auto-compact + budget reminders | observation elision + cache batching | (SDK-side) **[External]** |
| Permissions | approval hooks per host | wildcard rulesets, last-match-wins | typed approvals + Starlark policy | blocklist + sandbox | sandbox boundary + per-sandbox keys |
| Undo | checkpoint restore | shadow-git snapshots + log revert | per-turn diff tracking | git in container | container disposal |
| Distinctive lesson | the loop as embeddable, hook-extended library | event-sourcing pays for itself | security as depth-in-layers | agent-as-config; error taxonomy | orchestrate agents, don't host them |

Three cross-cutting verdicts that v1 missed or got backwards:

1. **The interesting state is not "memory," it's the *log*.** The systems that treat history as an append-only, flag-don't-delete log (opencode, Codex rollouts) get resumption, undo, multi-client, and compaction-as-a-view for free. v1's framing of memory as "RAM with eviction" undersold this: eviction is the easy half; *durability architecture* is the differentiator.
2. **Fuzzy edit application is a solved problem with a known algorithm** (ordered-leniency matching + uniqueness + proportionality guards, Ch. 5) — three independent implementations agree. Any new agent that ships a naive `str.replace` edit tool is ignoring settled engineering.
3. **Sandboxing matured from "Docker or nothing" to a layered policy stack** (command canonicalization → Starlark policy → OS mechanism → network proxy → typed approvals). v1's Docker-vs-native dichotomy is obsolete; Codex runs *natively* with stronger real-world containment than most container setups (which routinely mount the workspace and allow full egress anyway).

## 15.2 The twelve decisions

If you are building an agent, these are the decisions that shape everything downstream — each with the default this workspace's evidence supports:

1. **State substrate** — append-only persisted log with derived views. (Ch. 2.3, 8.6)
2. **Loop paradigm** — reconciler over the log if you chose (1); otherwise Cline's in-memory loop with host persistence. (Ch. 2)
3. **Topology** — embedded library for one host; client/server for many; agent-in-sandbox for untrusted/multi-tenant. (Ch. 1.3)
4. **Edit format** — search/replace + replacer chain for interactive agents; patch DSL if multi-file atomicity dominates. Fail closed, uniqueness required. (Ch. 5)
5. **Shell model** — persistent sessions from day one; output framing, timeout-as-result, ANSI stripping. (Ch. 6.1)
6. **Containment** — permission rulesets always; OS sandbox if you can afford per-platform work; container if the trigger isn't a developer at a keyboard. Egress control before syscall filtering. (Ch. 6, 10)
7. **Repo cognition ladder** — glob/grep/ranged-read first; tree-sitter skeletons; LSP if long-lived sessions; **no repo embeddings**. (Ch. 7)
8. **Context policy** — provider-usage-driven overflow detection; deterministic prune tier + summarize tier; atomic pair removal; batch mutations for cache. (Ch. 8)
9. **Planning surface** — plan mode = prompt + permissions; todo tool with verification-gated completion; subagents for context economics; two-phase submit. (Ch. 9)
10. **Permission engine** — (action, pattern) wildcard rules, last-match-wins, default ask, session grants, monotone narrowing for children. (Ch. 10.1)
11. **Steering & undo** — safe-point input draining; shadow-git snapshots transactional with log position. (Ch. 10.2–10.3)
12. **Multi-agent stance** — subagent trees by default; orchestrated peers (ledgers, selectors) only when the task is genuinely role-heterogeneous; graph runtime (LangGraph) when the *workflow* is the product. (Ch. 11, 12)

## 15.3 A minimal viable agent (annotated pseudo-code)

~90 lines that inherit the load-bearing decisions. Everything here maps to a verified mechanism from Part I (annotations point to chapters).

```python
# ---------- state: append-only log; views are derived (Ch 2.3, 8.6) ----------
class Log:
    def append(self, entry): db.insert(entry); bus.publish(entry)   # UI = subscriber
    def view(self):  # filter: drop pruned parts, cut at last summary + tail
        return render_for_model(db.entries, skip_flagged=True)

# ---------- permissions: rules, not booleans (Ch 10.1) ----------
def evaluate(action, pattern, *rulesets):
    rule = last_match(rulesets, action, pattern)      # wildcard, last wins
    return rule or Rule("ask")                        # fail closed

# ---------- edit tool: ordered leniency (Ch 5) ----------
REPLACERS = [exact, line_trimmed, block_anchor, whitespace_norm, escape_norm]
def apply_edit(content, old, new):
    for r in REPLACERS:
        for candidate in r(content, old):
            if count(content, candidate) == 1 and proportionate(candidate, old):
                return content.replace(candidate, new)
    raise ToolError("No unique match. Re-read the file and provide more context.")
                                                      # errors teach (Ch 4.1)

# ---------- context: prune then summarize (Ch 8) ----------
def enforce_budget(log, model):
    if tokens(log.view()) < threshold(model): return
    reclaimed = flag_old_tool_outputs(log, protect_tokens=40_000,
                                      protected=("skill", "todo"))
    if reclaimed < 20_000:                            # don't churn cache for crumbs
        head, tail = split_keeping_recent(log, budget=clamp(0.25*window, 2k, 8k))
        summary = summarizer_llm(handoff_prompt(head)) # state-of-work, not narrative
        log.append(summary_pair(summary))              # compaction is DATA in the log
        log.append(replay(last_real_user_msg(head)))   # resume the task, not its memory

# ---------- the loop: reconciler with typed error policy (Ch 2) ----------
def run(session):
    requeries = 0
    while True:
        view = session.log.view()
        last = view.last_assistant()
        if last and last.finished and not last.tool_calls: return last  # done
        enforce_budget(session.log, session.model)
        try:
            msg = llm(system_prompt(session.model.family)   # per-family asset (Ch 3.2)
                      + reminders(session),                  # tail injections only
                      tools=schemas(session.agent), history=view)
        except MalformedOutput as e:
            if (requeries := requeries + 1) > MAX: return salvage(session)  # (Ch 2.2)
            session.log.append(correction_template(e)); continue
        session.log.append(msg)
        for call in msg.tool_calls:                    # sequential by default (Ch 4.2)
            rule = evaluate(call.action, resolve_pattern(call), *session.rulesets)
            if rule.action == "deny": result = denied(rule)
            elif rule.action == "ask" and not await ui.approve(call, evidence(call)):
                result = denied_by_user()
            else:
                snapshot = shadow_git.commit(session.worktree)   # undo (Ch 10.3)
                result = dispatch(call, ctx=Context(abort, ask, progress))
                result = truncate_paged(result, max=50_000)      # page to disk (Ch 4.3)
            session.log.append(tool_result(call, result, snapshot))
```

What is deliberately *absent* — multi-agent orchestration, vector retrieval, a graph engine — matches what the production systems also omit from their cores. Add them as layers when the task demands (Ch. 11–13), not before.

## 15.4 On closed-source agents

v1 devoted a section to inferring Claude Code's internals. v2 declines: with five open production codebases in hand, inference about closed ones adds risk without insight — and the open systems visibly converge (per-model prompts, replacer-chain edits, ruleset permissions, two-tier compaction, subagent trees), so the closed ones are unlikely to be exotic. Where a closed agent's *observable* behavior matters (e.g., Codex's published prompts, which are in this repo), it is cited as such. **[Method note, not a claim]**

## 15.5 Closing argument

The engineering that makes coding agents work is conventional software architecture applied without sentimentality to an unreliable component: event sourcing for state, BSP for parallelism, optimistic concurrency for file freshness, rulesets for authorization, approximate string matching for edits, cache-aware data layout for cost. The model did not change what good systems look like; it changed *where the unreliability lives* — and every chapter of this book is some classical technique repositioned around that fact. Read the nine codebases with that lens and they stop looking like AI projects and start looking like what they are: distributed systems whose least reliable node happens to be brilliant.
