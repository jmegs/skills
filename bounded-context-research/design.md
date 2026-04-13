# Bounded Context Research Design

## Goal

Distill the unique research patterns from `smhanov/laconic` into a discoverable skill that changes agent behavior. The skill should teach a bounded-state research workflow, not document the Go library.

## Core Idea

Most research agents keep appending raw traces until the prompt gets bloated. This skill keeps working state compact on every loop and answers from that compact state instead of from a growing pile of search output.

## Trigger Surface

Use this skill when:
- the user asks an open-ended factual question that needs current web research
- the answer needs source grounding
- the answer requires synthesis across multiple searches or entities
- the right outcome may be "I could not verify that" instead of a guess

Do not use this skill when:
- the task is library or API documentation lookup
- the task is local codebase exploration
- the task is debugging or code review
- the user wants a video summarized
- the answer is already straightforward and does not need research

## Modes

### Scratchpad

Use `scratchpad` for single-focus questions that likely need only a few searches.

Working state:
- original question
- current knowledge summary, capped at one short paragraph or 5-8 bullets
- search history, capped at the last 3-5 queries
- iteration count

Loop:
1. Decide whether to answer or search.
2. If searching, run one focused query.
3. Rewrite the knowledge summary so it keeps only facts that help answer the question.
4. Discard the raw hits from working memory.
5. Repeat until grounded enough to answer or until the remaining uncertainty is itself the answer.

Escalation rule:
- if the summary starts holding multiple entities, conflicting sources, or too many open gaps, promote the run to `fact-notebook`

### Fact-Notebook

Use `fact-notebook` for multi-hop, multi-entity, or gap-driven research.

Working state:
- research goal
- key elements
- atomic facts with source URLs, capped to the 10-15 facts that matter most
- search queue, capped to the next 3-5 candidate queries
- visited queries, stored as short query labels only

Loop:
1. Rewrite the ask into a compact research goal.
2. Identify the key elements that must be resolved.
3. Generate a few targeted initial queries.
4. Extract atomic facts from snippets or fetched pages.
5. Deduplicate facts and track remaining gaps.
6. Stop early if the notebook already covers the goal.
7. Otherwise generate the next queries only for the unresolved gaps.

Boundedness rule:
- if the notebook grows past its cap, collapse settled facts into a short synthesis and drop facts that are no longer needed for the answer or citations

## Shared Rules

- Require evidence before answering.
- Keep runtime state in memory only and keep it small.
- Separate the roles mentally:
  - planner chooses the next move
  - extractor or synthesizer compresses evidence
  - finalizer writes the answer
- Prefer source-backed facts over smooth prose.
- Stop early when the current state is sufficient.
- Say plainly when evidence is incomplete or conflicting.
- Do not dump the full search trace unless the user asks for it.

## Storage Model

Runtime state is ephemeral and in-memory only. The skill should not write temp fact files, manifests, caches, or per-fact artifacts unless the user explicitly asks to save working notes.

The durable artifact is the answer.

## Answer Shape

Default answer shape:
1. Lead with the answer.
2. Add the minimum supporting facts needed for trust.
3. Include source URLs when they materially help.
4. State unresolved gaps directly.

## Evaluation Notes

The baseline runs for four prompts were stronger than expected on short, current, public facts. That means the value of this skill is not "the model can finally answer basic research questions." The value is disciplined state management, better behavior on longer research tasks, and consistency about grounding and uncertainty.

Future evals should emphasize:
- multi-hop questions
- multi-entity synthesis
- thin or conflicting evidence
- cases where raw-trace sprawl would normally hurt quality
