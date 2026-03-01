---
name: code-review
description: Review a pull request by link. Checks out the branch, verifies clean state, fetches the diff, and produces a thorough structured code review.
---

# Code Review Skill

Usage: `/code-review <pr-link>`

Performs a deep, structured review of a pull request. Covers purpose, complexity, blast radius, TCO, test coverage, code quality, safety, and documentation. Produces actionable, specific findings.

## Steps

### 1. Parse the PR Link

Extract the repository and PR number from the provided link. Supported formats:
- `https://github.com/<owner>/<repo>/pull/<number>`
- `<owner>/<repo>#<number>`

If the link is missing or malformed, ask the user to provide a valid PR URL before continuing.

### 2. Fetch PR Metadata

Use `gh pr view <number> --repo <owner>/<repo> --json title,body,headRefName,baseRefName,author,additions,deletions,changedFiles` to retrieve:
- PR title and description
- Head branch name
- Base branch name
- Author
- Change size (additions, deletions, files changed)

If `gh` is not available or the user is not authenticated, ask the user to paste the PR description and diff manually.

### 3. Verify Local Git State

Before reviewing, ensure the local environment is in a clean and consistent state:

1. **Check for uncommitted changes** — run `git status --porcelain`. If there are any staged, unstaged, or untracked files (that are not gitignored):
   - Clearly list the affected files
   - Stop and tell the user: "You have uncommitted changes. Please commit or stash them before reviewing this PR, then run `/code-review` again."
   - Do not proceed until the user confirms their working tree is clean.

2. **Check out the PR branch** — run `gh pr checkout <number> --repo <owner>/<repo>` (or `git checkout <headRefName>` if already fetched). If the branch is already checked out, confirm it is up to date with `git fetch origin <headRefName> && git status`.

3. **Confirm the branch is current** — if the local branch is behind the remote, pull the latest: `git pull origin <headRefName>`.

After these checks, confirm to the user: "Checked out `<headRefName>` — working tree is clean and up to date."

### 4. Fetch the Diff

Retrieve the full PR diff:

```
gh pr diff <number> --repo <owner>/<repo>
```

Also retrieve the list of changed files and their change types (added, modified, deleted, renamed). Use this to understand the scope before diving into details.

### 5. Understand the Purpose

Read the PR title and description carefully. Identify:
- **What problem this solves** — what was broken, missing, or painful before?
- **Who this is for** — a specific user role, team, service, or workflow. Be concrete.
- **What the expected behavior change is** — what will be different after this lands?
- **Any stated caveats or known limitations**

If the PR description is missing or uninformative, note this as a finding: a PR without a clear purpose statement is harder to review correctly and harder to revert safely.

If the PR references issues or other PRs, read those for additional context before proceeding.

Use this understanding to anchor all subsequent analysis.

### 6. Analyze Complexity Appropriateness

Review the diff and ask: is the complexity of the solution proportional to the complexity of the problem?

Flag if:
- A simple problem was solved with unnecessarily complex abstractions
- A complex problem was addressed with an overly naive or fragile approach
- The code introduces new abstractions that could have been avoided
- The logic is hard to follow relative to what it is doing

Be specific. Quote the code. Suggest simplifications or call out where the complexity is justified.

### 7. Assess Blast Radius

Identify what the change touches and what could break:

- Which modules, packages, services, or systems are affected?
- Are there any callers or dependents not covered by the tests?
- Does the change touch shared infrastructure, public APIs, or data schemas?
- Is the change backward compatible? If not, what migration is required?
- What is the rollback story if this causes a production issue?

Summarize the blast radius as: **Narrow** (isolated, low risk), **Moderate** (some shared code, reviewable risk), or **Wide** (cross-cutting, requires careful rollout).

If Moderate or Wide, assess whether a narrower scoped change could accomplish the same purpose. If a smaller approach exists, describe it concretely.

### 8. Assess Total Cost of Ownership

Evaluate the long-term maintenance burden the change introduces or removes:

- **Complexity debt**: Does this add a new abstraction, pattern, or dependency that future engineers must understand? Is the cost justified?
- **New dependencies**: Flag any new external dependencies explicitly. Each is a long-term liability — security, upgrade, and build cost. Is it truly necessary?
- **Operational burden**: New infrastructure, jobs, queues, caches, or external services that need monitoring and scaling?
- **Testing burden**: Will this be easy or hard to test going forward? Does it make the suite slower or more brittle?
- **Onboarding cost**: Can a new team member safely modify this area, or does it require tribal knowledge?
- **Upgrade and migration exposure**: Does this lock in assumptions that will be expensive to change later?
- **Incident cost**: If this breaks in production, how hard will it be to diagnose? Does it fail loudly or silently?

Summarize whether the change **increases**, **decreases**, or is **neutral** on TCO, and briefly explain why.

### 9. Verify Test Coverage

Check whether the change is adequately tested:

- Are there **unit tests** for the new or changed logic?
- If the change touches user-facing behavior or integrations, are there **end-to-end tests**?
- Are edge cases and failure modes tested?
- Do the tests test behavior, not implementation details?

Run the test suite on the checked-out branch to confirm tests pass:
- For npm/TypeScript: typecheck, lint, build, test
- For Bazel: `bazel test //...` for affected packages
- For other ecosystems: use the appropriate runner (`pytest`, `go test ./...`, `cargo test`, etc.)

If coverage gaps exist, name them explicitly. If tests fail, report the specific failures as a blocking finding.

### 10. Review Code Quality

Read code surrounding the changed lines — not just the diff. Understand caller context, adjacent logic, and upstream/downstream effects. Many bugs live at the boundary between changed and unchanged code.

The change will be read and emulated by others. Review for:

- **Naming**: Are variables, functions, and types named clearly and consistently?
- **Structure**: Is the code organized logically? Are responsibilities well-separated?
- **Style**: Does it match the project's conventions (formatting, idioms, patterns)?
- **Readability**: Can a new engineer understand what this code does in 60 seconds?
- **No dead code**: Leftover debug statements, commented-out blocks, or unused imports?
- **No over-engineering**: Does every abstraction earn its existence?

Flag issues specifically. Don't nitpick unless it matters; do flag anything that would cause a reviewer to pause.

#### Human Readability

Code is read far more times than it is written, and debugging is harder than writing — readability is the most important quality of any change.

Specifically review:
- **Clarity of intent**: Does the code communicate *why* it does what it does, not just *what*?
- **Surprising logic**: Any non-obvious branching, ordering dependency, or side effect must be explained with a comment. If you had to think twice about a line, future readers will too.
- **Naming as documentation**: Names should be precise enough that a reader doesn't need to look at the implementation to understand what a function or variable does.
- **Cognitive load**: Is the reader asked to hold too many things in their head at once? Long functions, deeply nested conditions, and overloaded variables all increase the chance of misreading.

#### Safety and Correctness

Review for classes of bugs that are easy to miss in review but expensive to debug in production:

- **Memory safety**: Use-after-free, double-free, buffer overflows, uninitialized reads, leaked resources.
- **Concurrency**: Shared mutable state without adequate synchronization. Locks held for the wrong scope.
- **Race conditions**: TOCTOU bugs, non-atomic read-modify-write, ordering assumptions that only hold under sequential execution.
- **Cache coherency**: Stale reads, invalidation gaps, TTL assumptions that break under load or clock skew.
- **Async/await pitfalls**: Unawaited promises, fire-and-forget that swallows errors, async functions called synchronously.
- **Integer and numeric issues**: Overflow, underflow, truncation on cast, division by zero, floating-point equality.
- **Injection**: SQL injection, command injection, XSS, template injection, or any case where untrusted input reaches a sensitive sink without sanitization.
- **Auth and access control**: Missing or incorrect authentication checks, authorization bypasses, privilege escalation paths.
- **Data exposure**: Secrets, PII, or tokens logged, leaked in error messages, or included in client-facing responses.

Flag any of these specifically with the affected line and the failure scenario.

#### Error Handling and Observability

- **Swallowed errors**: Empty or log-only catch blocks, promises with no `.catch()`, missing `await`.
- **Unchecked return values**: Error indicators or `Result` types where both arms are not handled.
- **Silent success lies**: Functions that return success even when the underlying operation failed.
- **Inappropriate log levels**: `DEBUG`/`INFO` for critical events; `ERROR`/`WARN` in hot paths for expected conditions.
- **Missing observability**: New code paths, jobs, or integrations with no metrics, traces, or structured log events.
- **Retry and backoff issues**: Retrying non-idempotent operations; retries with no backoff causing thundering herd.
- **Timeout gaps**: External calls with no timeout, or timeouts too long to bound failure recovery.
- **Null/undefined propagation**: Values used without guards after type assertions that strip null-safety.

### 11. Analyze Usability

For any change that touches user-facing behavior — APIs, CLI interfaces, UI, configuration, error messages, or developer-facing surfaces — evaluate whether it is usable, not just correct:

- **Discoverability**: Can users find the feature or understand that it exists? Are affordances clear?
- **Error messages**: Are failure messages actionable? Do they tell the user what went wrong *and* what to do about it? Vague or technical error messages are bugs.
- **Defaults**: Do defaults reflect the most common and safest use case? A bad default is worse than no default.
- **Consistency**: Does the interface follow conventions already established in the product (naming, flag format, response shape, etc.)? Inconsistency increases learning cost.
- **Progressive disclosure**: Does the change expose complexity appropriately — simple things are simple, advanced things are possible?
- **Documentation and examples**: For API or CLI changes, is there enough context for a user to get started without reading source code?
- **Accessibility**: For UI changes, check for keyboard navigability, screen reader compatibility, color contrast, and focus management.

Flag usability issues like any other finding — they affect real users and are often cheaper to fix before merge than after.

### 12. Check Documentation

Identify documentation that may need updating:

- **Inline comments**: Are complex or non-obvious sections explained?
- **Function/method docstrings**: Are public APIs documented?
- **README or markdown docs**: Does user-facing or developer-facing documentation need updating?
- **Changelog**: If the project maintains one, does this change need an entry?

### 13. Validate Findings

Before finalizing, stress-test every issue (blocking and non-blocking). For each candidate finding, spawn a subagent (using the Agent tool) with this role:

> You are an adversarial reviewer. Your job is to argue that the following claimed issue is wrong, overstated, or not worth raising. Consider: Is the code actually correct and the reviewer misread it? Is there surrounding context that makes it safe? Is this a style preference disguised as a correctness issue? Is the failure scenario realistic or purely theoretical? Make the strongest case you can that this should be downgraded or dropped.

Provide the subagent with the claimed issue, the relevant code, and sufficient surrounding context. Batch related issues into a single subagent call where they share context.

After reviewing the adversarial response, do one of:
- **Keep at current severity** — if the issue survives the challenge.
- **Downgrade** — blocking to non-blocking, if the challenge reveals it's real but less severe.
- **Drop** — if the challenge reveals the issue was wrong or too trivial to raise.

This step is mandatory. It prevents false positives that waste the author's time and erode trust in the review.

### 14. Produce the Review

Output a structured review using the format below. Be specific throughout — quote code, cite line numbers, and give actionable recommendations. **Omit any section that has no findings** — do not include empty sections or write "looks good" / "no issues found."

**Always call out what was done well.** Even when a PR has significant issues, a good review names the things the author got right — a clever approach, a well-named abstraction, solid test coverage, a clean refactor. This is not politeness; it reinforces good patterns, helps the author calibrate what to keep, and makes the critical feedback land better. If the PR is mostly good, say so clearly and specifically.

---

**Review format:**

```markdown
## Verdict

<Approve | Request Changes | Comment> — <N blocking, M non-blocking>

<One paragraph summary of the overall assessment and the most important things the author should address.>

## Blast Radius

<Narrow | Moderate | Wide> — <one sentence explanation>

## What Was Done Well

<Specific callouts of things the author got right — clever approaches, clean structure, good naming, solid test coverage, thoughtful error handling, etc. Be concrete. This section should be non-empty unless the PR is trivially small.>

## Blocking Issues

<Issues that must be addressed before merging. Include file path, line reference, and specific recommendation for each. Omit this section if there are none.>

## Non-Blocking Issues

<Issues worth addressing but not merge blockers. Keep each to one sentence with file path and line reference. Omit this section if there are none.>

## Usability

<Usability issues — error messages, defaults, discoverability, consistency, accessibility. Omit if none or no user-facing surface.>

## Test Coverage

<Coverage gaps or test failures only. Omit if tests pass and coverage is adequate.>

## TCO

<Only include this section if the change markedly increases long-term maintenance burden. Briefly explain why.>

## Documentation

<Doc gaps identified. Omit if none.>
```

---

## Notes

- Never approve a PR with failing tests
- Quote specific code in findings rather than describing it vaguely
- Distinguish clearly between blocking and non-blocking issues
- If the PR description is empty or unhelpful, flag it — a PR without context is a maintenance liability
- Do not suggest changes unrelated to the PR's stated purpose
- After producing the review, offer to post it as a GitHub PR review comment using `gh pr review <number> --repo <owner>/<repo> --comment --body "..."`
