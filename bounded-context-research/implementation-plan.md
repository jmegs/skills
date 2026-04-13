# Bounded Context Research Implementation Plan

## Goal

Create a new `bounded-context-research` skill in an isolated directory that turns the best ideas from `laconic` into a reusable research workflow.

## Architecture

Version 1 stays self-contained. The main artifact is `SKILL.md`, supported by a small eval set and baseline workspace outputs. Runtime state stays in memory only.

## Files

- `bounded-context-research/design.md`
- `bounded-context-research/implementation-plan.md`
- `bounded-context-research/SKILL.md`
- `bounded-context-research/evals/evals.json`
- `bounded-context-research-workspace/iteration-0/`

## Tasks

### 1. Capture the approved design

- Write the design doc.
- Keep the skill focused on discoverability rather than provenance.
- Preserve the in-memory-only runtime model.

### 2. Run the RED phase

- Run baseline research prompts without the skill.
- Save the outputs in an isolated workspace.
- Record what the baseline does well and where the skill still adds value.

### 3. Draft the skill

- Use trigger-focused frontmatter.
- Keep the body self-contained.
- Teach two modes: `scratchpad` and `fact-notebook`.
- Emphasize grounding, compact state, bounded state caps, early stopping, explicit uncertainty, and a promotion rule from `scratchpad` to `fact-notebook`.

### 4. Seed evals

- Save the initial eval set to `evals/evals.json`.
- Include prompts that cover:
  - simple grounded current facts
  - multi-entity synthesis
  - policy nuance
  - insufficient evidence

### 5. Prepare for iteration

- Keep the workspace structure isolated under `bounded-context-research-workspace/`.
- If later iterations are needed, add `iteration-1/`, `iteration-2/`, and so on.
- If the eval set is not discriminating enough, add a harder multi-hop prompt before revising the skill.

## Current Status

The design is approved.

The initial baseline outputs are saved locally.

The next meaningful iteration after this file is to run with-skill evals and tighten the wording if the skill under-triggers, over-triggers, or encourages too much trace dumping.
