---
name: human-review
description: Explain a pull request to a human reviewer. Surfaces what changed, why it matters, which component boundaries are crossed, and what a reviewer should pay attention to. Use this skill when the user wants to understand a PR before reviewing it, get oriented in an unfamiliar diff, or prepare to give thoughtful feedback.
---

# Human Review Skill

Usage: `/human-review <pr-link>`

Explains a pull request to a human reviewer — not to approve or reject it, but to help the reviewer build a clear mental model before they dive into the diff. Surfaces context, intent, component impact, and the things most worth a reviewer's attention.

## Steps

### 1. Parse the PR Link

Extract the repository and PR number from the provided link. Supported formats:
- `https://github.com/<owner>/<repo>/pull/<number>`
- `<owner>/<repo>#<number>`

If the link is missing or malformed, ask the user to provide a valid PR URL.

### 2. Fetch PR Metadata and Diff

Use `gh` to retrieve everything needed in parallel:

```bash
# Metadata
gh pr view <number> --repo <owner>/<repo> --json title,body,headRefName,baseRefName,author,additions,deletions,changedFiles,comments,reviews

# Diff
gh pr diff <number> --repo <owner>/<repo>
```

If `gh` is not available or the user is not authenticated, ask them to paste the PR description and diff manually.

### 3. Understand the Change

Read the PR title, description, and diff to reconstruct what the author was trying to accomplish. Do not assume the PR description is accurate or complete — cross-reference it with the actual diff.

Identify:
- **The problem being solved**: What was broken, missing, or painful before this change?
- **The approach**: How did the author solve it? What alternatives might exist?
- **Who is affected**: Which callers, consumers, services, or users will notice this change?

If the PR description is empty, vague, or contradicts the diff, note that explicitly — it's a signal the reviewer should ask the author for clarification before approving.

### 4. Map Component Boundaries

Identify which parts of the system this PR touches and how they interact. For each changed area, describe:

- **What layer it belongs to** (e.g., API surface, data model, business logic, infrastructure, UI)
- **What it depends on** (upstream callers, data sources, config, external services)
- **What depends on it** (downstream consumers, other modules, public interfaces)
- **Whether the boundary is crossed** — i.e., does this change affect contracts between components, not just internals?

Boundary crossings are the highest-risk parts of any PR. Flag them clearly.

For unfamiliar code, read surrounding context:

```bash
git fetch origin <headRefName>
git show origin/<headRefName>:<path/to/file>
```

### 5. Assess the Blast Radius

Estimate how far the impact of this change reaches:

- **Narrow** — change is contained within a single component, module, or file. Other parts of the system are unaffected even if something goes wrong.
- **Moderate** — change affects a shared utility, a data contract, or a component with several callers. A bug here could surface in multiple places.
- **Wide** — change touches a foundational layer, a public API, a shared schema, or something that runs in every request path. A bug here could be widespread.

State the blast radius clearly and explain the reasoning. This tells the reviewer how carefully to read the diff.

### 6. Surface What Needs a Reviewer's Attention

This is the most important step. Based on your analysis, identify the 2–5 things a reviewer should actually think hard about. These are not bugs or issues — they are judgment calls, tradeoffs, or areas where the reviewer's domain knowledge matters most.

Good examples:
- "The new `parseConfig` function is called in the hot path — worth checking whether the added complexity could affect latency."
- "The data migration runs inline during startup. Worth asking the author: what happens if it fails halfway through?"
- "This changes the shape of the `UserProfile` type. Any consumers not in this repo may need updates."
- "The PR replaces the polling approach with webhooks — the author has a comment explaining why, but worth asking whether the new failure mode (missing webhook delivery) is acceptable."

Avoid surfacing things that are obviously fine or that a reviewer would notice in 10 seconds of reading. Focus on the things that require context or domain knowledge to evaluate.

### 7. Produce the Explanation

Always include all sections. Scale the detail to the size and complexity of the PR — a 5-line fix doesn't need a multi-paragraph component map, but a cross-cutting refactor does.

```markdown
## What This PR Does

<2–4 sentences: the problem being solved, the approach taken, and who it affects.
Write this as if explaining to a colleague who hasn't read the PR — concrete and specific.>

## Component Map

<Which parts of the system are touched, what layer they belong to, and how they connect.
Call out any component boundaries being crossed. Use a Mermaid diagram if it helps.>

## Blast Radius

<Narrow | Moderate | Wide> — <one sentence explaining why>

<If Moderate or Wide: which specific callers, consumers, or services are affected, and how.>

## What to Pay Attention To

<2–5 specific things worth a reviewer's careful consideration. These are judgment calls and
tradeoffs — not bugs. Frame each as a question or observation the reviewer should sit with.>

## Context Gaps

<Anything the PR description leaves unclear, or places where the diff diverges from the stated intent.
If the PR description is missing or misleading, say so here. Omit this section if the PR is well-documented.>
```

### Diagram Guidelines

Include a Mermaid diagram in the **Component Map** section when the PR touches more than one component or crosses a meaningful boundary. Diagrams help reviewers orient quickly.

- Use `flowchart` for data flow or call chains
- Use `sequenceDiagram` for request/response interactions
- Use `classDiagram` for type or schema changes
- Label nodes with real names from the codebase, not abstractions like `A` or `B`
- Show which nodes are new, changed, or unchanged (use color or labels)
- Keep it focused — show the parts relevant to this PR, not the entire system

Example color conventions:
- Changed: `style NodeName fill:#f5a623,color:#000`
- New: `style NodeName fill:#7ed321,color:#000`
- Unchanged (for context): `style NodeName fill:#9b9b9b,color:#fff`

## Guidelines

- This skill explains, it does not judge. Do not approve, reject, or issue verdicts.
- Write for someone who is smart but unfamiliar with this part of the codebase.
- Prefer concrete specifics over vague generalities — name actual files, functions, and types.
- If you genuinely cannot determine the intent of a change from the diff and description, say so rather than guessing.
- Do not summarize every changed file — focus on the parts that matter for understanding the change.
- After producing the explanation, offer to follow up with `/robot-review` if the user wants a full automated code review.
