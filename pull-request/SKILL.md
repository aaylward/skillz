---
name: pull-request
description: Review changes, run tests, and open a well-documented pull request with commit message and PR description
---

# Pull Request Skill

A thorough pre-PR checklist and automated PR creation. Covers test coverage, complexity analysis, blast radius, documentation, and a polished PR with Mermaid diagrams.

## Steps

### 1. Clarify the Purpose

Infer the purpose from the diff, branch name, commit history, and conversation context. Only ask questions where the answer is genuinely ambiguous. The goal is to understand:

- **What problem does this change solve?** What was broken, missing, or painful before?
- **Who is this for?** Be concrete: "API consumers paginating large result sets" or "the nightly billing reconciliation job" — not "users" or "the system."
- **What is the business problem being solved?** Connect the technical change to a real outcome — reliability, speed, cost, correctness, developer productivity, etc.
- **How will this be used?** Who calls this code, when, and under what conditions?
- **What is the expected behavior change?** What will be different after this lands?
- **Are there any edge cases or caveats the reviewer should know?**

Use this understanding to inform all subsequent analysis, the commit message, and the PR description. The who and the business problem should be visible in the PR summary.

### 2. Run All Tests

Run the full test suite — not just tests related to the changed files. Projects often use multiple build systems simultaneously (e.g., Bazel alongside npm); identify and run all of them.

**For npm / TypeScript targets**, always run all four of these, not just tests:
1. **Typecheck** — e.g., `tsc --noEmit` or the project's typecheck script
2. **Formatter** — e.g., `prettier --check .` or `eslint --fix` (run the check variant, not auto-fix, so failures are visible)
3. **Build** — e.g., `npm run build`
4. **Test** — e.g., `npm test`

**For Bazel targets**, run the relevant `bazel build //...` and `bazel test //...` for affected packages. If both Bazel and npm are present, run both.

For other ecosystems use the appropriate runner (e.g., `pytest`, `go test ./...`, `cargo test`).

All checks must pass before proceeding. If any fail, report the specific failures and stop. Do not open a PR with failing checks.

### 3. Analyze Complexity Appropriateness

Review the diff and ask: is the complexity of the solution proportional to the complexity of the problem?

Flag if:
- A simple problem was solved with unnecessarily complex abstractions
- A complex problem was addressed with an overly naive or fragile approach
- The code introduces new abstractions that could have been avoided
- The logic is hard to follow relative to what it is doing

Be specific. Quote the code. Suggest simplifications or call out where the complexity is justified.

### 4. Assess Blast Radius

Identify what the change touches and what could break:

- Which modules, packages, services, or systems are affected?
- Are there any callers or dependents not covered by the tests?
- Does the change touch shared infrastructure, public APIs, or data schemas?
- Is the change backward compatible? If not, what migration is required?
- What is the rollback story if this causes a production issue?

Summarize the blast radius as: **Narrow** (isolated, low risk), **Moderate** (some shared code, reviewable risk), or **Wide** (cross-cutting, requires careful rollout).

If the blast radius is Moderate or Wide, actively analyze whether a smaller or more tightly scoped change could accomplish the purpose identified in step 1. Don't just ask — review the diff, understand what is strictly necessary to achieve the stated goal, and identify what could be deferred, split out, or removed. If a narrower approach exists, describe it concretely and recommend it. Let the user decide whether to proceed with the current scope or reduce it, but give them a real option to choose from.

### 5. Assess Total Cost of Ownership

For any design change or substantive code change, evaluate the long-term maintenance burden it introduces or removes:

- **Complexity debt**: Does this add a new abstraction, pattern, or dependency that future engineers must understand? Is that cost justified by what it enables?
- **New dependencies**: Be wary of any new external dependencies. Each one is a long-term liability — it must be kept up to date, may introduce security vulnerabilities, adds to build times, and can break without warning. A new dependency is only justified if the alternative (implementing the functionality yourself or using what's already available) is clearly worse. Flag new deps explicitly and ask whether they're truly necessary.
- **Operational burden**: Does this introduce new infrastructure, jobs, queues, caches, or external dependencies that need to be monitored, scaled, and kept alive?
- **Testing burden**: Will this be easy or hard to test going forward? Does it make the test suite slower, more brittle, or harder to reason about?
- **Onboarding cost**: Will a new team member be able to understand and safely modify this area of the codebase? Or does it require tribal knowledge to work with correctly?
- **Upgrade and migration exposure**: Does this make future upgrades (libraries, platforms, schemas) harder? Does it lock in assumptions that will be expensive to change?
- **Incident cost**: If this breaks in production, how hard will it be to diagnose and fix? Does it fail loudly and obviously, or silently and subtly?

Summarize whether the change increases, decreases, or is neutral on TCO, and briefly explain why. Flag any cases where a simpler alternative would have meaningfully lower long-term cost.

### 6. Verify Test Coverage

Check whether the change is adequately tested:

- Are there **unit tests** for the new or changed logic?
- If the change touches user-facing behavior or integrations, are there **end-to-end tests**?
- Are edge cases and failure modes tested?
- Do the tests test behavior, not implementation?

If coverage gaps exist, name them explicitly and ask the user whether to add tests before opening the PR.

### 7. Review Code Quality

Read code surrounding the changed lines — not just the diff. Understand caller context, adjacent logic, and upstream/downstream effects. Many bugs live at the boundary between changed and unchanged code.

First, verify **functional correctness**: does the code actually do what it claims to do? Trace through the logic against the stated purpose from step 1. Check that edge cases, boundary conditions, and error paths produce the right behavior — not just the happy path.

The change should be exemplary — it will be read and emulated by others. Check for:
- **Naming**: Are variables, functions, and types named clearly and consistently?
- **Structure**: Is the code organized logically? Are responsibilities well-separated?
- **Style**: Does it match the project's conventions (formatting, idioms, patterns)?
- **Readability**: Can a new engineer understand what this code does in 60 seconds?
- **No dead code**: Are there leftover debug statements, commented-out blocks, or unused imports?
- **No over-engineering**: Does every abstraction earn its existence?

Flag issues specifically. Don't nitpick unless it matters; do flag anything that would cause a reviewer to pause.

#### Human Readability

Code is read far more times than it is written, and debugging is harder than writing — so readability is not a nice-to-have, it is the most important quality of any change.

Specifically review:
- **Clarity of intent**: Does the code communicate *why* it does what it does, not just *what*? A reader should not have to reverse-engineer the author's reasoning.
- **Surprising logic**: Any non-obvious branching, ordering dependency, or side effect must be explained with a comment. If you had to think twice about a line, future readers will too.
- **Naming as documentation**: Names should be precise enough that a reader doesn't need to look at the implementation to understand what a function or variable does.
- **Cognitive load**: Is the reader asked to hold too many things in their head at once? Long functions, deeply nested conditions, and overloaded variables all increase the chance of misreading during a future debug session.

If any part of the change would slow down a reader — even slightly — flag it and suggest a clearer alternative.

#### Safety and Correctness

Review for classes of bugs that are easy to miss in review but expensive to debug in production:

- **Memory safety**: Use-after-free, double-free, buffer overflows, uninitialized reads, or leaked resources (handles, connections, allocations). In GC languages, check for unintentional retention that prevents collection.
- **Concurrency**: Shared mutable state accessed from multiple threads or coroutines without adequate synchronization. Check locks are held for the right scope — not too narrow (leaving gaps) or too wide (causing contention or deadlock).
- **Race conditions**: Time-of-check/time-of-use (TOCTOU) bugs, non-atomic read-modify-write sequences, or assumptions about ordering that only hold under sequential execution.
- **Cache coherency**: Stale reads from caches that aren't invalidated on write. Optimistic caching that doesn't account for concurrent writers or distributed state. TTL assumptions that break under load or clock skew.
- **Async/await pitfalls**: Unawaited promises, fire-and-forget that swallows errors, async functions called synchronously in contexts where the result is silently dropped.
- **Integer and numeric issues**: Overflow, underflow, truncation on cast, division by zero, floating-point equality comparisons.
- **Injection**: SQL injection, command injection, XSS, template injection, or any case where untrusted input reaches a sensitive sink without sanitization.
- **Auth and access control**: Missing or incorrect authentication checks, authorization bypasses, privilege escalation paths.
- **Data exposure**: Secrets, PII, or tokens logged, leaked in error messages, or included in client-facing responses.

Flag any of these specifically with the affected line and the failure scenario. These bugs often don't surface in tests but cause hard-to-reproduce production incidents.

#### Error Handling and Observability

- **Swallowed errors**: Empty or log-only catch blocks that discard failures silently. Errors that are caught, logged, and then execution continues as if nothing happened. Promises with no `.catch()` or missing `await`.
- **Unchecked return values**: Functions that return an error or success indicator that the caller ignores. In languages that return `(result, error)` tuples or `Result` types, both arms must be handled.
- **Silent success lies**: Functions that return success codes or resolve without error even when the underlying operation failed, making the failure invisible to callers.
- **Inappropriate log levels**: `DEBUG`/`INFO` for genuinely critical events that need to page someone; `ERROR`/`WARN` in hot paths where the condition is expected and frequent (log spam, alert fatigue, and performance overhead).
- **Missing observability**: New code paths, background jobs, or integrations with no metrics, traces, or structured log events — making future incidents blind. Important operations should be observable without attaching a debugger.
- **Retry and backoff issues**: Retrying non-idempotent operations that could cause duplicate side effects. Retries with no backoff or jitter causing thundering herd on a recovering dependency.
- **Timeout gaps**: External calls (network, DB, RPC) with no timeout set, or timeouts so long they don't bound failure recovery meaningfully.
- **Null/undefined propagation**: Values that can be null/undefined used without guards, especially after type assertions or casts that strip the compiler's null-safety.

### 8. Analyze Usability

For any change that touches user-facing behavior — APIs, CLI interfaces, UI, configuration, error messages, or developer-facing surfaces — evaluate whether it is usable, not just correct:

- **Discoverability**: Can users find the feature or understand that it exists? Are affordances clear?
- **Error messages**: Are failure messages actionable? Do they tell the user what went wrong *and* what to do about it?
- **Defaults**: Do defaults reflect the most common and safest use case?
- **Consistency**: Does the interface follow conventions already established in the product?
- **Progressive disclosure**: Do simple things remain simple while advanced options are still accessible?
- **Documentation and examples**: For API or CLI changes, is there enough context for a user to get started without reading source code?
- **Accessibility**: For UI changes, check keyboard navigability, screen reader compatibility, color contrast, and focus management.

Flag any usability issues and ask the user whether to address them before opening the PR.

### 9. Check Documentation

Identify any documentation that may need updating:

- **Inline comments**: Are complex or non-obvious sections explained?
- **Function/method docstrings**: Are public APIs documented?
- **README or markdown docs**: Does any user-facing or developer-facing documentation need updating?
- **Changelog**: If the project maintains one, does this change need an entry?

### 10. Write the Commit Message

Write a commit message following this format:

```
<type>(<scope>): <short imperative summary under 72 chars>

<body: explain WHY the change was made, not just WHAT changed.
Reference the problem, the approach, and any tradeoffs.
Keep lines under 72 chars. Use bullet points if helpful.>

<footer: breaking changes, issue refs, co-authors if relevant>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`

Present the commit message to the user for approval before committing.

### 11. Commit and Open the PR

After user approval of the commit message:

1. **Check the branch is current** — verify the branch is rebased on or recently merged with the base branch. If it's significantly behind, recommend rebasing and re-running tests before opening.
2. Stage and commit the changes with the approved message.
3. Push the branch.
4. **Link related issues** — if the work relates to a tracked issue, include `Closes #123` or `Fixes #123` in the PR body.
5. **Ask draft or ready** — ask the user whether to open as a draft PR (for early feedback) or ready for review.
6. Present the full PR description to the user for review before opening.
7. Open the PR with the following structure:

---

**PR Description format:**

```markdown
## Summary

<2-4 sentence summary of what changed and why>

## What Changed

<Bullet list of specific changes>

## How to Test

<Steps to verify the change works correctly>

## Blast Radius

<Narrow | Moderate | Wide> — <one sentence explanation>

## Before / After

<Before/after Mermaid diagram>

## Architecture / Flow

<Architecture or flow diagram using the most appropriate Mermaid type: sequenceDiagram, stateDiagram, classDiagram, flowchart, etc.>

## Discussion Points

<List any decisions where there were significant technical or UX tradeoffs — choices where a reasonable engineer could have gone a different direction. For each, briefly describe the tradeoff and why this approach was chosen. These are conversation starters, not blockers. Omit this section if there are no meaningful tradeoffs worth surfacing.>
```

Discussion Points example format:
- **[Technical] Polling vs. webhooks**: Chose polling for simplicity and to avoid requiring a public endpoint, at the cost of added latency. Worth discussing if latency becomes a concern.
- **[UX] Fail-closed vs. fail-open on auth error**: Defaulting to deny on auth service timeout to avoid accidental data exposure. Could be revisited if this causes user-visible availability issues.

---

Both Mermaid diagrams must:
- Use descriptive node labels
- Include color styling (e.g., `style NodeName fill:#4a90d9,color:#fff` or `%%{init: {'theme': 'base', 'themeVariables': {...}}}%%`)
- Be meaningful — not just decorative. One diagram showing before/after or structure, one showing flow or sequence.
- In before/after diagrams, the "before" state must always appear before (above or to the left of) the "after" state — never reversed.
- Minimize unnecessary line and arrow crossings. Arrange nodes so that flow is clean and directional; restructure layout rather than accept a tangled diagram.
- If the change touches many components, don't cram everything into one diagram. Instead, use one high-level overview diagram showing the broad structure or flow, then additional focused detail diagrams zooming into the specific subsystems or interactions that changed.

## Notes

- Never open a PR with auto-merge enabled — always leave merging as an explicit manual action for the author
- Never open a PR with failing tests
- Never skip the user clarification step — the purpose of the change must be explicit
- The PR description should be readable by someone with no prior context
- Prefer compact and specific over long and vague
- Diagrams are not required for every PR. Omit them when they would be trivially obvious or purely decorative. But even small changes can benefit from a diagram showing where in the system the change lives — locating a change within the broader architecture helps reviewers build context quickly
