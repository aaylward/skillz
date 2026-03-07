---
name: code-review
description: Review a pull request by link or from the current branch. Checks out the branch, fetches the diff, and produces a structured code review scaled to the PR's size and risk. Use this skill whenever the user wants to review a PR, look at changes in a pull request, give feedback on a diff, or assess code quality of a branch.
---

# Code Review Skill

Usage: `/code-review <pr-link>`

Performs a structured review of a pull request, scaled to its size and risk. Small safe changes get a brief review; large risky changes get a thorough one.

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
gh pr view <number> --repo <owner>/<repo> --json title,body,headRefName,baseRefName,author,additions,deletions,changedFiles

# Diff
gh pr diff <number> --repo <owner>/<repo>
```

If `gh` is not available or the user is not authenticated, ask them to paste the PR description and diff manually.

### 3. Triage: Determine Review Depth

Before diving in, classify the PR to calibrate how deep to go. Use the metadata and a quick scan of the diff:

**Light review** — skim the diff, check for obvious issues, produce a short review:
- Fewer than ~50 lines changed
- Only touches docs, config, comments, or trivial formatting
- Dependency version bumps with no code changes

**Standard review** — read the diff carefully, check logic and tests, produce a full review:
- Moderate size (~50-500 lines changed)
- Touches application logic, tests, or build configuration
- Most PRs fall here

**Deep review** — read surrounding code beyond the diff, trace call chains, scrutinize edge cases:
- Large changes (500+ lines) or touches many files
- Modifies auth, security, data schemas, public APIs, or shared infrastructure
- Introduces new architecture patterns or significant refactors
- Changes concurrency, caching, or error handling in critical paths

Tell the user which depth you've chosen and why, e.g.: "This is a 12-line doc fix — doing a light review." If the user disagrees, adjust.

### 4. Read Context Without Switching Branches

Do NOT check out the PR branch — this disrupts the user's working tree. Instead, read surrounding code using:

```bash
# Read a specific file at the PR's branch
git fetch origin <headRefName>
git show origin/<headRefName>:<path/to/file>
```

For **light reviews**, the diff alone is usually sufficient — skip this step unless something looks off.

For **standard and deep reviews**, read surrounding context for any changed code where the diff alone doesn't make the logic clear — especially function signatures, callers, type definitions, and tests.

### 5. Analyze the Changes

What you check depends on the review depth. Work through the applicable items:

#### Always (all depths)
- **Purpose**: What problem does this solve? Is the approach reasonable?
- **Correctness**: Are there logic errors, off-by-ones, or missed edge cases?
- **Obvious safety issues**: Injection, hardcoded secrets, auth bypasses, data exposure

#### Standard + Deep
- **Complexity vs. problem**: Is the solution appropriately complex for the problem it solves?
- **Naming and readability**: Can a new engineer understand this quickly?
- **Error handling**: Swallowed errors, missing null checks, unawaited promises
- **Test coverage**: Are new code paths tested? Are edge cases covered? If tests exist in the diff, read them carefully.
- **Style consistency**: Does it match the project's existing patterns?

#### Deep only
- **Blast radius**: What modules/services/callers are affected? Is it backward compatible? Only surface this in the output if Moderate or Wide.
- **Concurrency and race conditions**: Shared mutable state, TOCTOU, non-atomic operations
- **TCO impact**: Does this add meaningful maintenance burden — new dependencies, abstractions, or operational complexity?
- **Rollback story**: If this breaks in production, how hard is it to revert?

### 6. Run Tests (only if requested)

Do NOT run the test suite automatically — it can be slow and the user may not want it. Instead, if you notice test files in the diff, mention them and ask:

> "This PR includes test changes. Want me to check out the branch and run the tests?"

If the user says yes, use a worktree to avoid disrupting their working tree:

```bash
git worktree add /tmp/pr-review-<number> origin/<headRefName>
cd /tmp/pr-review-<number>
# run tests...
git worktree remove /tmp/pr-review-<number>
```

### 7. Validate Findings

This step is mandatory — false positives waste the author's time and erode trust in the review. Every blocking and non-blocking issue must survive adversarial challenge before being included.

For each finding, spawn a subagent (using the Agent tool) with this role:

> You are an adversarial reviewer. Your job is to argue that the following claimed issue is wrong, overstated, or not worth raising. Consider: Is the code actually correct and the reviewer misread it? Is there surrounding context that makes it safe? Is this a style preference disguised as a correctness issue? Is the failure scenario realistic or purely theoretical? Make the strongest case you can that this should be downgraded or dropped.

Provide the subagent with the claimed issue, the relevant code, and sufficient surrounding context. Batch related findings into a single subagent call where they share context.

After reviewing the adversarial response, do one of:
- **Keep** — if the issue survives the challenge
- **Downgrade** — blocking to non-blocking, if the challenge reveals it's real but less severe
- **Drop** — if the challenge reveals the issue was wrong or too trivial to raise

For **light reviews** with only minor observations, you can do this challenge inline rather than spawning a subagent — but still do it.

### 8. Produce the Review

Scale the review format to the depth:

#### Light reviews
Keep it to a few sentences. No need for headers or formal structure unless there's actually an issue to flag.

```markdown
**Looks good** — <brief summary of what the change does and any minor observations>
```

Or if there's an issue:

```markdown
**One issue** — <the problem, with file:line reference and suggestion>

Otherwise this looks clean. <brief positive note if warranted>
```

#### Standard and Deep reviews

Use the structured format below. **Omit any section that has no findings** — do not include empty sections or "no issues found."

Always call out what was done well. Name specific things the author got right — a clean refactor, good naming, solid test coverage, a thoughtful approach. This reinforces good patterns and makes critical feedback land better.

```markdown
## Verdict

<Approve | Request Changes | Comment> — <N blocking, M non-blocking>

<One paragraph: overall assessment and the most important things to address.>

## What Was Done Well

<Specific callouts — clever approaches, clean structure, good naming, solid tests, etc.>

## Blocking Issues

<Must fix before merge. Include file:line and specific recommendation for each.>

## Non-Blocking Issues

<Worth addressing but not merge blockers. One sentence each with file:line.>

## Test Coverage

<Coverage gaps or test failures. Omit if adequate.>

<!-- DEEP REVIEWS ONLY — omit both sections below for light and standard reviews -->

## Blast Radius

<Moderate or Wide only — omit if Narrow>

## TCO

<Only if the change markedly increases maintenance burden>
```

## Verdict Criteria

The verdict is the most important part of the review — it's what the author acts on first.

- **Approve** — The change is correct, well-tested, and safe to merge as-is. Minor style nits don't block approval, but missing tests for new logic do.
- **Comment** — The change looks reasonable but has gaps that should be addressed. Use this when there are no outright bugs but the PR isn't fully ready (e.g., missing tests for new code paths, unclear intent, or non-trivial concerns that need the author's input).
- **Request Changes** — The change has correctness issues, security problems, or is fundamentally mis-approached.

**When to withhold Approve:**
- Security-sensitive changes (auth, access control, client identity, crypto, input validation) without corresponding tests should not be approved. Even if the fix looks correct, untested security code is a regression waiting to happen. Use "Comment" and explicitly call out that tests are needed before merging.
- New branching logic or conditional behavior without any test coverage.
- Changes where you cannot verify correctness from the diff alone and would need to run tests or read significant surrounding code.

The bar for "Approve" should be: "I am confident this is safe to merge right now." If you're not confident, use "Comment" — it's not a negative signal, it's an honest one.

## Guidelines

- Quote specific code in findings rather than describing it vaguely
- Distinguish clearly between blocking and non-blocking issues
- If the PR description is empty or unhelpful, flag it — a PR without context is a maintenance liability
- Do not suggest changes unrelated to the PR's stated purpose
- Never approve a PR with obviously failing logic, even if you didn't run tests
- After producing the review, offer to post it as a GitHub PR review comment using `gh pr review`
