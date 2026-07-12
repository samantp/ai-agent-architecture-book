# Chapter 14 — STORM: Research as a Pipeline

STORM (Stanford) writes Wikipedia-grade articles from nothing but a topic. It earns its chapter for one architectural idea the coding agents don't exhibit: **agency as a fixed pipeline of narrow LLM roles**, where the "agent loop" is replaced by staged dataflow and the LLM never controls what happens next. It is the strongest counter-example to Part I's model-driven loops — and its techniques transfer directly to the "research before you code" phase of engineering agents.

## 14.1 The pipeline

`storm@fb951af/knowledge_storm/storm_wiki/engine.py` **[Verified structure]** wires four stages, each an isolated module with typed inputs/outputs:

```
topic → KnowledgeCuration (research) → InformationTable
      → OutlineGeneration            → hierarchical outline
      → ArticleGeneration            → sectioned draft w/ citations
      → ArticlePolish                → lead section, dedup → article
```

Each stage can run, be cached, and be evaluated independently (the runner exposes per-stage toggles). Nothing downstream can reorder or skip upstream work — determinism the model-driven loops of Chapter 2 explicitly trade away. Note also the implementation substrate: STORM is built on **DSPy** — prompts are typed `dspy.Signature` classes (docstring = instructions, fields = I/O schema) composed into `dspy.Module`s **[Verified]** — prompts as *programs* rather than strings, one step more disciplined than opencode's prompts-as-assets (Ch. 3.2).

## 14.2 Perspective discovery: fan-out before depth

STORM's research quality comes from asking *different questions*, not more questions. `persona_generator.py` **[Verified]**: `FindRelatedTopic` (`:48`) pulls related Wikipedia articles and their tables of contents, `GenPersona` (`:56`) distills them into K editor personas ("performance engineer," "historian of the standard," …), and a default `"Basic fact writer"` is always appended (`StormPersonaGenerator`, `:114`). Grounding personas in *related articles' structure* — rather than asking the model to brainstorm freely — imports the accumulated editorial judgment of adjacent topics. Each persona then researches in parallel.

## 14.3 The simulated interview

The core loop — two LLMs interviewing each other — `storm@fb951af/knowledge_storm/storm_wiki/modules/knowledge_curation.py:25-81` **[Verified]**:

```python
# ConvSimulator.forward (abridged)
for _ in range(self.max_turn):
    user_utterance = self.wiki_writer(topic, persona, dlg_history).question
    if user_utterance == "": break
    if user_utterance.startswith("Thank you so much for your help!"):
        break                                  # protocol sentinel, not a parser
    expert_output = self.topic_expert(topic, question=user_utterance, ...)
    dlg_history.append(DialogueTurn(agent_utterance=expert_output.answer,
                                    user_utterance=user_utterance,
                                    search_queries=expert_output.queries,
                                    search_results=expert_output.searched_results))
```

The asymmetry is the design: the **WikiWriter** (persona'd questioner) has no retrieval — only curiosity shaped by the dialogue so far; the **TopicExpert** has no persona — it decomposes each question into search queries (`QuestionToQuery`), retrieves, and answers *only from retrieved text* with citations (`AnswerQuestion`, `:167-245`). Separating the asker from the knower forces every fact through retrieval — hallucination control by role design rather than by instruction. Termination is a literal string sentinel the questioner is prompted to emit — crude, effective, and honest about what protocol termination over natural language is.

Two production details inside this loop, both context management in miniature (cf. Ch. 8.4): interview history beyond the last 4 turns renders as *"Expert: Omit the answer here due to space limit"* (questions kept, answers elided — `knowledge_curation.py:103-113` **[Verified]**), and the rendered conversation is clamped to 2,500 words.

## 14.4 Grounded synthesis

Downstream stages maintain the citation chain: curation yields an `InformationTable` (turns + sources); outline generation drafts from the model's prior, then *refines against the interviews* (structure informed by what research actually found); article generation writes **section by section**, each section retrieving its own support from the table (per-section embedding search) so no single call needs the full corpus — `tree_summarize`/`compact` shapes again (Ch. 13.4); polish adds the lead and dedupes. Every claim in the output traces to a retrieved source; the pipeline's *shape* enforces what a "cite your sources" instruction merely requests. (`storm@fb951af/knowledge_storm/collaborative_storm/` extends this to a human-in-the-roundtable variant — **[Verified structure]**.)

## 14.5 What transfers to coding agents

1. **Fan-out-then-synthesize beats deep-first search** for unfamiliar territory. "Understand this codebase" as K parallel perspective-subagents (security, data-flow, build-system, API-surface) whose reports merge into one brief is STORM's curation stage wearing Chapter 9.3's subagent machinery.
2. **Separate the asker from the knower** wherever grounding matters: a reviewer subagent that can only *query* (grep/LSP/docs) and must cite `file:line` for every claim is the TopicExpert pattern applied to code review.
3. **Artifact-first research**: STORM's outline+table is the research artifact that makes the writing stage cheap and auditable. The coding equivalent — a `RESEARCH.md` with cited findings written *before* implementation (Ch. 9's plan-mode output) — turns exploration from vibes into a reviewable deliverable.
4. **Pipeline where the process is known, loop where it isn't** — the same verdict as Chapter 12, reached from the opposite direction: STORM is what "workflow-shaped agency" looks like when built from prompts instead of graphs; LangGraph is what it looks like when built from checkpoints. A production research feature would sensibly be STORM's stages hosted on LangGraph's durability.
