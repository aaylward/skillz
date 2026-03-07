---
name: pull-request
description: Review changes, run tests, and open a well-documented pull request with commit message and PR description. Use this skill whenever the user wants to create a PR, open a pull request, push changes for review, or prepare code for merge.
---

# Pull Request Skill

Usage: `/pull-request`

Prepares and opens a pull request for the current branch. Runs tests, writes a commit message, and produces a clear PR description with diagrams where they help explain the change.

## Steps

### 1. Understand the Change

Infer the purpose from the diff, branch name, commit history, and conversation context. Only ask questions where the answer is genuinely ambiguous. Identify:

- **What problem does this solve?** What was broken, missing, or painful before?
- **Who is this for?** Be concrete — "API consumers paginating large result sets", not "users."
- **What is the expected behavior change?** What will be different after this lands?

This understanding anchors the commit message, PR description, and any analysis.

### 2. Triage: Determine PR Complexity

Classify the change to calibrate how much analysis is needed:

**Trivial** — dependency bumps, typo fixes, config changes, formatting:
- Run tests, commit, open PR. Skip analysis steps.

**Standard** — most feature work, bug fixes, refactors:
- Run tests, do a quick self-review for obvious issues, commit, open PR.

**Complex** — large changes, cross-cutting concerns, new architecture, security-sensitive:
- Run tests, do thorough self-review (complexity, blast radius, test coverage), commit, open PR.

Tell the user which level you've chosen. If they disagree, adjust.

### 3. Run Tests

Run the project's test suite before proceeding. Identify the build system(s) in use and run all relevant checks:

**npm / TypeScript:**
1. Typecheck — `tsc --noEmit` or the project's typecheck script
2. Lint — `eslint` or the project's lint script (check mode, not auto-fix)
3. Build — `npm run build`
4. Test — `npm test`

**Bazel:** `bazel build //...` and `bazel test //...` for affected packages.

**Other ecosystems:** Use the appropriate runner (`pytest`, `go test ./...`, `cargo test`, etc.).

If both Bazel and npm are present, run both. All checks must pass before proceeding. If any fail, report the specific failures and stop.

For **trivial** changes in large monorepos, use targeted test commands (affected packages only) rather than running the entire suite, unless the user asks for a full run.

### 4. Self-Review (Standard and Complex only)

Read through the diff with fresh eyes. This is not a full code review — it's a quick sanity check before sending to reviewers.

**Standard PRs — check for:**
- Obvious logic errors or missed edge cases
- Dead code, debug statements, or commented-out blocks left in
- Naming that would confuse a reviewer
- Missing tests for new code paths

**Complex PRs — also check for:**
- Complexity proportional to the problem being solved
- Blast radius: what modules/callers/services are affected? Summarize as Narrow/Moderate/Wide.
- Test coverage gaps for edge cases and failure modes
- Any security concerns (injection, auth, data exposure)

If you find issues, report them to the user and ask whether to fix them before opening the PR. Do not silently open a PR with known problems.

#### Validate Findings

Before reporting issues to the user, challenge each one:
- Is the code actually correct and you misread it?
- Is this a style preference, not a real problem?
- Is the failure scenario realistic?

Drop anything that doesn't survive the challenge. False positives waste time and erode trust.

### 5. Build the Testing Done Checklist

Based on the tests you ran in Step 3 and any self-review findings from Step 4, build the "Testing Done" checklist for the PR description. Include:

- Which test suites passed (unit, integration, e2e, typecheck, lint, build)
- Any manual verification you performed
- Specific edge cases or scenarios you tested

The checklist should reflect what was *actually done*, not aspirational testing. If a test category wasn't run, don't include it. This gives reviewers a clear picture of the author's confidence level and helps them decide what to verify themselves.

### 6. Write the Commit Message

Follow this format:

```
<type>(<scope>): <short imperative summary under 72 chars>

<body: explain WHY the change was made, not just WHAT changed.
Reference the problem, the approach, and any tradeoffs.
Keep lines under 72 chars.>

<footer: breaking changes, issue refs>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`

Present the commit message to the user for approval before committing.

### 7. Commit and Open the PR

After user approval of the commit message:

1. Stage and commit with the approved message.
2. Push the branch.
3. Ask the user: **draft or ready for review?**
4. Present the PR description for review before opening.
5. Open the PR.

**PR Description format — scale to complexity:**

#### Trivial PRs

```markdown
## Summary

<1-2 sentences: what changed and why>
```

That's it. No diagrams, no blast radius, no test plan for a dep bump or typo fix.

#### Standard PRs

```markdown
## Summary

<2-4 sentences: what changed, why, and who it affects>

## What Changed

<Bullet list of specific changes>

## Testing Done

<Checklist of what the author verified before opening the PR. Use checkboxes so reviewers can see what was covered and suggest additions.>

- [ ] <e.g., Ran unit tests locally — all passing>
- [ ] <e.g., Manually tested the new endpoint with valid and invalid inputs>
- [ ] <e.g., Verified edge case X behaves correctly>

## Diagram

<A Mermaid diagram if it helps explain the change — see diagram guidelines below. Omit if the change is self-explanatory from the summary and diff.>
```

#### Complex PRs

```markdown
## Summary

<2-4 sentences: what changed, why, and who it affects>

## What Changed

<Bullet list of specific changes>

## Testing Done

<Checklist of what the author verified before opening the PR.>

- [ ] <e.g., Ran full test suite — all passing>
- [ ] <e.g., Tested happy path and error cases manually>
- [ ] <e.g., Verified backward compatibility with existing callers>

## Blast Radius

<Narrow | Moderate | Wide> — <one sentence explanation>

## Diagrams

<One or more Mermaid diagrams — see diagram guidelines below>

## Discussion Points

<Tradeoffs where a reasonable engineer could have gone a different direction. For each, briefly describe the tradeoff and why this approach was chosen. Omit if no meaningful tradeoffs.>
```

Discussion Points example:
- **Polling vs. webhooks**: Chose polling for simplicity and to avoid requiring a public endpoint, at the cost of latency. Worth revisiting if latency becomes a concern.
- **Fail-closed on auth timeout**: Defaulting to deny to avoid accidental data exposure. Could be revisited if this causes availability issues.

### Diagram Guidelines

Diagrams are valuable when they clarify something that's hard to see from the diff alone — how execution flows through a system, how components interact, or what changed structurally. Even small changes can benefit from a diagram if the change has subtle effects on execution flow or system behavior.

**When to include a diagram:**
- The change affects how components or services interact
- There's a before/after difference in flow, state, or architecture worth showing
- The execution path is non-obvious and a picture would help a reviewer build context
- The change touches a part of a larger system and locating it visually helps

**When to skip:**
- Dependency bumps, config changes, typo fixes
- The change is completely self-explanatory from the diff
- A diagram would just restate the bullet points in "What Changed"

**Diagram quality:**
- Use the most appropriate Mermaid type: `sequenceDiagram`, `flowchart`, `stateDiagram`, `classDiagram`, etc.
- Use descriptive node labels — not `A`, `B`, `C`
- Include color styling (e.g., `style NodeName fill:#4a90d9,color:#fff`)
- In before/after diagrams, "before" appears first (above or left)
- Minimize line crossings — arrange nodes for clean directional flow
- For large changes, use one high-level overview plus focused detail diagrams rather than cramming everything into one

## Notes

- Never open a PR with auto-merge enabled
- Never open a PR with failing tests
- Link related issues with `Closes #123` or `Fixes #123` in the PR body
- The PR description should be readable by someone with no prior context
- Prefer compact and specific over long and vague
