# Appendix — Source Map and v1 Corrections

All references use the pinned form `repo@hash/path:line` — hashes resolve via the **Pinned revisions** table in [README.md](README.md). Paths are relative to each repo root; prepend `/Users/pari/aiCodingAgents/<repo>/` to open locally, or build a permalink as `https://github.com/<upstream>/blob/<full-hash>/<path>#L<line>`.

## A.1 Load-bearing claims → source

### The agent loop (Ch. 2)
| Claim | Source |
|---|---|
| Cline canonical loop, reminders-contested stop, hook surface | `cline@6309971/sdk/packages/agents/src/agent-runtime.ts:570-756` (loop at 604; hooks 758-1216; approval 1269) |
| SWE-agent requery taxonomy, autosubmission salvage | `SWE-agent@1132b3e/sweagent/agent/agents.py:1006-1170` (forward 1006; handler 1062) |
| opencode reconciler loop; compaction/subtask as log tasks | `opencode@34e5809/packages/opencode/src/session/prompt.ts:1081-1190` |
| Codex task→turn loop; pending-input drain; mid-turn auto-compact | `open-interpreter@764a96e/codex-rs/core/src/tasks/regular.rs:37-89`; `open-interpreter@764a96e/codex-rs/core/src/session/turn.rs:144,227-371` |

### Prompt assembly (Ch. 3)
| Claim | Source |
|---|---|
| Per-model-family system prompts | `opencode@34e5809/packages/opencode/src/session/system.ts:27-41`; `opencode@34e5809/packages/opencode/src/session/prompt/*.txt` |
| Codex prompts shipped in-repo; policy content | `open-interpreter@764a96e/codex-rs/core/gpt_5_codex_prompt.md`; `prompt_with_apply_patch_instructions.md:132,143,279+` |
| Agent-as-YAML; instance recipe; pager env vars; submit review gate | `SWE-agent@1132b3e/config/default.yaml` (entire file) |
| Parser taxonomy (function calling ⇄ text protocols) | `SWE-agent@1132b3e/sweagent/tools/parsing.py:52-574` |
| Escalating error templates | `cline@6309971/apps/vscode/src/core/prompts/responses.ts` |
| Cache breakpoints; cache-aware eviction batching | `SWE-agent@1132b3e/sweagent/agent/history_processors.py:261 (CacheControl); :85-176 (LastNObservations, polling)` |

### Tools (Ch. 4)
| Claim | Source |
|---|---|
| Tool context contract (abort/ask/metadata); InvalidArgumentsError | `opencode@34e5809/packages/opencode/src/tool/tool.ts` |
| Truncation pages to disk w/ retrievable path; 2000 lines / 50KB | `opencode@34e5809/packages/opencode/src/tool/truncate.ts` |
| Parallelism via `RwLock<()>` read/write gating | `open-interpreter@764a96e/codex-rs/core/src/tools/parallel.rs:48,100-136` |
| Subagent tool: resume (`task_id`), background etiquette | `opencode@34e5809/packages/opencode/src/tool/task.ts` |
| Sequential dispatch + approval pipeline | `cline@6309971/sdk/packages/agents/src/agent-runtime.ts:1136-1368` |

### Editing (Ch. 5)
| Claim | Source |
|---|---|
| 9-replacer chain; uniqueness; proportionality guard; teaching errors | `opencode@34e5809/packages/opencode/src/tool/edit.ts:217,244-363,682-737` |
| apply_patch grammar; multi-`@@` addressing | `open-interpreter@764a96e/codex-rs/core/prompt_with_apply_patch_instructions.md:279+` |
| seek_sequence decreasing strictness; eof anchoring | `open-interpreter@764a96e/codex-rs/apply-patch/src/seek_sequence.rs` |
| Fail-closed contract, `is_exact` delta tracking | `open-interpreter@764a96e/codex-rs/apply-patch/src/lib.rs:184-211` |
| Strict uniqueness errors, no fuzz | `SWE-agent@1132b3e/tools/edit_anthropic/bin/str_replace_editor:523-537` |
| Lint-gated edit variants (evolutionary record) | `SWE-agent@1132b3e/tools/windowed_edit_*` |

### Execution & sandboxing (Ch. 6)
| Claim | Source |
|---|---|
| Persistent remote shell sessions; file RPCs | `SWE-agent@1132b3e/sweagent/environment/swe_env.py:8-17,197-263` |
| Seatbelt deny-default policy (Chrome-inspired) | `open-interpreter@764a96e/codex-rs/sandboxing/src/seatbelt_base_policy.sbpl` |
| Landlock helper-binary re-exec | `open-interpreter@764a96e/codex-rs/sandboxing/src/landlock.rs:23-65`; `open-interpreter@764a96e/codex-rs/linux-sandbox/` |
| Sandbox env markers; ExecRequest policy plumbing | `open-interpreter@764a96e/codex-rs/core/src/sandboxing/mod.rs:140-173` |
| Starlark exec policy; command canonicalization | `open-interpreter@764a96e/codex-rs/execpolicy/`; `open-interpreter@764a96e/codex-rs/core/src/command_canonicalization.rs` |
| Agent-in-sandbox; session keys; webhook event flow; tini; LRU | `OpenHands@3949e1c/openhands/app_server/sandbox/docker_sandbox_service.py:385-513` |
| Sandbox service variants (docker/process/remote) | `OpenHands@3949e1c/openhands/app_server/sandbox/*_sandbox_service.py` |

### Repo cognition (Ch. 7)
| Claim | Source |
|---|---|
| rg-first doctrine in prompt | `open-interpreter@764a96e/codex-rs/core/gpt_5_codex_prompt.md` |
| grep tool escalation policy (rg → Task tool) | `opencode@34e5809/packages/opencode/src/tool/grep.txt` |
| ripgrep walker + host-index fallback | `cline@6309971/apps/vscode/src/services/search/file-search.ts:18-44` |
| tree-sitter filemap (body elision) | `SWE-agent@1132b3e/tools/filemap/bin/filemap` |
| tree-sitter chunk localization | `OpenHands@3949e1c/openhands/app_server/utils/chunk_localizer.py:80+` |
| LSP tool (goToDefinition/hover/workspaceSymbol); diagnostics | `opencode@34e5809/packages/opencode/src/tool/lsp.ts:12-14,84-89`; `opencode@34e5809/packages/opencode/src/lsp/` |
| IDE diagnostics bridge | `cline@6309971/apps/vscode/src/integrations/diagnostics/` |
| File staleness tracking | `cline@6309971/apps/vscode/src/core/context/context-tracking/FileContextTracker.ts` |

### Context management (Ch. 8)
| Claim | Source |
|---|---|
| Prune constants (20k/40k, protected tools, 2-turn skip) | `opencode@34e5809/packages/opencode/src/session/compaction.ts:28-33,241-287` |
| Head/tail select; 25% clamp 2k–8k; turn splitting; summary-as-messages; replay | `opencode@34e5809/.../compaction.ts:80-128,188-239,289-319` |
| Trigger math; 0.5 ratios; strategy duality | `cline@6309971/sdk/packages/core/src/extensions/context/compaction.ts:82-232,160-190` |
| Atomic tool-pair removal; structure-preserving truncation | `cline@6309971/.../basic-compaction.ts:90-222` |
| Summarizer budget; separate summarizer model | `cline@6309971/.../agentic-compaction.ts:51-158` |
| Mid-turn compaction; token status; budget reminders | `open-interpreter@764a96e/codex-rs/core/src/session/turn.rs:306-371`; `open-interpreter@764a96e/codex-rs/core/src/session/rollout_budget.rs`, `token_budget.rs` |
| Observation elision + cache warning + polling | `SWE-agent@1132b3e/sweagent/agent/history_processors.py:85-176` |
| Interview eliding microcosm | `storm@fb951af/knowledge_storm/storm_wiki/modules/knowledge_curation.py:103-113` |

### Planning & HITL (Ch. 9–10)
| Claim | Source |
|---|---|
| Plan mode prompts + enter/exit tools | `opencode@34e5809/packages/opencode/src/session/prompt/plan*.txt`; `opencode@34e5809/packages/opencode/src/tool/plan-enter.txt`, `plan.ts` |
| Todo protocol ("never based on intent", one in-progress) | `opencode@34e5809/packages/opencode/src/tool/todowrite.txt` |
| Codex plan calibration; self-monitoring tools | `open-interpreter@764a96e/codex-rs/core/gpt_5_codex_prompt.md`; `open-interpreter@764a96e/codex-rs/core/src/tools/handlers/get_context_remaining.rs`, `tool_search*`, `request_user_input*` |
| Two-phase submit w/ embedded diff | `SWE-agent@1132b3e/config/default.yaml` (SUBMIT_REVIEW_MESSAGES); `SWE-agent@1132b3e/tools/review_on_submit_m/` |
| Wildcard ruleset evaluator; default ask; session grants | `opencode@34e5809/packages/opencode/src/permission/evaluate.ts:28-36`; `opencode@34e5809/packages/opencode/src/permission/index.ts:67-74` |
| Subagent permission narrowing | `opencode@34e5809/packages/opencode/src/agent/subagent-permissions.ts` |
| Shadow-git snapshots (`--git-dir`/`--work-tree` split) | `opencode@34e5809/packages/opencode/src/snapshot/index.ts:67-131`; `opencode@34e5809/packages/opencode/src/session/revert.ts` |
| Checkpoint restore | `cline@6309971/apps/vscode/src/core/controller/checkpoints/checkpointRestore.ts` |

### Frameworks (Ch. 11–14)
| Claim | Source |
|---|---|
| Actor runtime queue/envelopes; lazy agents; gRPC/CloudEvents | `autogen@027ecf0/python/packages/autogen-core/src/autogen_core/_single_threaded_agent_runtime.py:149-526`; `autogen@027ecf0/protos/` |
| Speaker selection stack + retry feedback | `autogen@027ecf0/python/packages/autogen-agentchat/.../_selector_group_chat.py:152-300` |
| Swarm handoff routing | `autogen@027ecf0/python/packages/autogen-agentchat/.../_swarm_group_chat.py:52-62` |
| Magentic-One ledgers, stall counter, re-planning | `autogen@027ecf0/python/packages/autogen-agentchat/.../_magentic_one/_magentic_one_orchestrator.py:72-119`; `_prompts.py:79-118` |
| Composable termination conditions | `autogen@027ecf0/python/packages/autogen-agentchat/src/autogen_agentchat/conditions/_terminations.py` |
| Pregel tick/after_tick; pending-writes replay | `langgraph@55ec2f2/libs/langgraph/langgraph/pregel/_loop.py:599-718` |
| Checkpointer interface | `langgraph@55ec2f2/libs/checkpoint/langgraph/checkpoint/base/__init__.py:176-300` |
| interrupt/Command semantics (node re-execution) | `langgraph@55ec2f2/libs/langgraph/langgraph/types.py:535-885` |
| AST chunking (recursive greedy pack) | `llama_index@67514f6/llama-index-core/llama_index/core/node_parser/text/code.py:19-260` |
| Router engine; synthesizer family | `llama_index@67514f6/.../query_engine/router_query_engine.py:95-164`; `.../response_synthesizers/` |
| Personas from related articles | `storm@fb951af/knowledge_storm/storm_wiki/modules/persona_generator.py:48-118` |
| Interview loop; sentinel termination; asker/knower split | `storm@fb951af/.../knowledge_curation.py:25-245` |

## A.2 Corrections to v1

1. **Open Interpreter chapter (v1 Ch. 21)** described the legacy Python REPL architecture. The clone is the Rust rewrite forked from OpenAI Codex; v1's content did not describe any code in this workspace. Replaced by Codex-grounded material throughout Part I.
2. **Cline paths**: v1 cited `cline/apps/vscode/src/core/task/index.ts`, `assistant-message/diff.ts`, `services/tree-sitter/`, and the `recursivelyMakeClineRequests` loop — none exist in this clone. The engine now lives in `cline@6309971/sdk/packages/agents` (`AgentRuntime`); no tree-sitter integration exists in current Cline.
3. **OpenHands**: v1 cited `OpenHands/openhands/events/action/action.py`, a pub/sub `EventStream` loop, and an in-repo agent controller. This clone contains none of that — the agent is the external `openhands-sdk`; the repo is the app/orchestration server. v1's "PTY server inside the container" description is superseded by the agent-server-in-container + webhook architecture. (v1's `chunk_localizer` citation was correct and is retained.)
4. **opencode**: v1 described `packages/core/src/pty`, `packages/protocol`, and SQLite persistence in `packages/core/src/database` — the real modules are `packages/opencode/src/*` (session, tool, permission, snapshot, lsp) with an event-sourced message store; the "headless daemon" framing survives but the mechanism (log reconciliation) is materially different and more interesting.
5. **v1's Ch. 7 claim that "no major open-source agent enforces structured planning"** understated reality: todo state machines with verification-gated completion (opencode), calibrated plan-tool policy (Codex), and two-phase submit (SWE-agent) all ship today; what remains absent is runtime DAG scheduling.
6. **v1's Claude Code chapter**: speculation removed (see Ch. 15.4).
7. **v1's "Docker vs. native" security dichotomy** replaced by the layered model of Ch. 6 (policy → OS sandbox → network proxy → approvals), which is what the strongest current implementation actually does.
