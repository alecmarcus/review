---
name: resolve
description: "This skill should be used when the user asks to \"resolve PR comments\", \"resolve this PR\", \"handle review feedback\", \"resolve issue\", \"resolve ticket\", \"address PR comments\", or provides a PR/issue/ticket reference to resolve."
argument-hint: "[<url> | <number> | <branch> | current | <LINEAR-ID> | <comment-url>]"
context: fork
disable-model-invocation: false
allowed-tools: Bash, Read, Glob, Grep, Task, WebFetch, mcp__vestige__search, mcp__vestige__smart_ingest, mcp__vestige__codebase
---

# /review:resolve

Resolve:
- PR review threads
- PR comments
- GitHub issues
- Linear tickets
- etc.

For each item: research the claim, validate or fix, reply with reasoning and artifact citations, and resolve.

## Prime Directive: Fix, Don't Defer

**Your job is to FIX things, not create issues for later. Creating a GitHub issue is not resolving feedback — it is deferring it.** Every VALID or PARTIALLY_VALID item must result in an actual code change committed to the branch.

- A plan to fix something is not a fix.
- A note explaining what should change is not a fix.
- An issue tracking future work is not a fix.
- Only a committed code change is a fix.

The only acceptable reasons to not fix a VALID item are: (1) the fix requires human judgment on a genuinely ambiguous design tradeoff, or (2) the fix would require changes to code not in this PR's repository. Everything else gets fixed now.

**Line-by-line reading is mandatory.** Every verdict must be backed by having read — not skimmed, not grepped, not assumed — the full file, the full diff, and every relevant artifact in full. If you haven't Read it with the Read tool, you don't know what it says.

## Step 1: Parse Target

Resolve `$ARGUMENTS` into one or more targets:

| Input | Type | Resolution |
|-------|------|-----------|
| Empty or `current` | PR | Infer from current branch: `gh pr view --json number --jq '.number'` |
| Bare number (`42`) | PR | PR #42 in current repo |
| `issue 42` or `issue #42` | Issue | GitHub issue #42 |
| URL `github.com/.../pull/42` | PR | Extract owner/repo/number |
| URL `github.com/.../issues/42` | Issue | Extract owner/repo/number |
| URL `github.com/.../pull/42#discussion_r123` | Comment | Single review comment thread |
| `TEAM-123` (uppercase letters + hyphen + digits) | Linear | Linear ticket ID |
| URL `linear.app/...` | Linear | Linear ticket URL |
| Branch name | PR | Find PR for branch: `gh pr list --head <branch> --json number --jq '.[0].number'` |
| Multiple space-separated | Mixed | Parse each independently |

Store resolved targets. If resolution fails for any target, report the error for that target and continue with others.

## Step 2: Fetch

### 2.1 PR Threads, Comments & Reviews

For PR targets, fetch threads, PR-level comments, and formal reviews in one GraphQL request.

See `references/gh-graphql.md` for the exact query (`PRFetch`).

```bash
gh api graphql --input - << EOF
{"query":"<PR_FETCH_QUERY>","variables":{"owner":"$OWNER","repo":"$REPO","number":$PR_NUMBER}}
EOF
```

Parse the response to extract:

**Review threads** — filter to unresolved, non-outdated:
- Thread ID, file path, line range
- First comment body and author
- All replies

**PR-level comments** — filter to non-minimized:
- Comment ID (`id`), URL, body, author

**Reviews** — filter to `CHANGES_REQUESTED` or `COMMENTED` state:
- Review ID, body, author, state

Also fetch the PR diff for context:
```bash
gh pr diff $PR_NUMBER
```

### 2.2 GitHub Issues

For issue targets:
```bash
gh issue view <number> --json title,body,comments,labels,assignees,state
```

Extract the issue description and all comments as items to address.

### 2.3 Linear Tickets

For Linear targets, use the Linear MCP if available, or fall back to the Linear CLI/API:
```bash
linear issue view <ID>
```

Extract the ticket description, comments, and status.

### 2.4 Optional: Update PR Description

If the PR title or description is unclear, vague, missing context, or has outdated checklist items, update it with a clearer summary based on the diff and commit history.

## Step 3: Research (Parallel Per Item)

For each unresolved comment/review/thread/issue/ticket, dispatch a **Task subagent** to research it. Dispatch all research agents in parallel.

Each research agent receives:

```
Task(subagent_type="general-purpose", prompt="""
## Research Task

You are researching a review comment to determine if it is valid.

### The Comment
Author: <author>
File: <path>
Lines: <start>-<end>
Comment: <body>
Thread replies: <any existing replies>

### Relevant Code
<Read the file at the specified lines, plus 50 lines of surrounding context>
<Read the diff for this file>

### Instructions

1. **Read the full file** at <path> using the Read tool — the entire file, not just the referenced lines. Understand the module context: imports, exports, class structure, type definitions, adjacent functions, error handling patterns. Then re-read lines <start>-<end> with at least 50 lines of surrounding context. **You may not render a verdict on code you have not fully read.**
2. **Read the full diff** for this file and all related files (imports, callers, test files, type definitions). Understand what changed AND what stayed the same. Read every hunk, not just the ones near the comment.
3. **Read git history** for the changed lines: `git log --follow -p -- <path>` or `git blame <path>`. Read the actual diffs in the history, not just commit messages. Understand what the code looked like before and what motivated the change.
4. **Read every relevant artifact in full** — not excerpted, not summarized, not inferred from titles:
   - `.docs/specs/` — read the entire spec, not just the section that seems relevant. Constraints appear in adjacent sections.
   - `.docs/adrs/` — read the full rationale, alternatives, and consequences sections.
   - `.docs/standards/` — read every applicable standard rule.
   - `.docs/lessons/` — read known pitfalls and patterns.
   - `CLAUDE.md` — read project rules in full.
   - PRD stories and acceptance criteria — read each criterion individually.
   - GitHub issues referenced — read the issue body AND all comments. Clarifications often appear in comments.
   - GitHub PR comments and review threads — read the full thread including all replies.
   - Commit messages on this PR — read the message AND `git show <sha>` to see the actual diff.
   - Any local `.docs/` near the changed files.
   **A Grep match is a starting point, not research.** After finding a match, Read the full file. A function name in search results does not mean you understand what it does.
5. **Search Vestige** specifically for past review findings on the same files, past bug fixes in the same module, and known anti-patterns: `mcp__vestige__search(query: "<project> <file-path> review findings bug fix anti-pattern")`. Include search results in your verdict.
6. **Interpret both literally and thematically** — the literal text is what the reviewer noticed; the thematic concern is what principle they're pointing at. A comment about "missing error handling" may thematically point at failure-mode design. A comment about "hardcoded value" may thematically point at configurability. Surface both levels.
7. **Evaluate against code AND artifacts.** When the thematic concern is valid but the literal suggestion is suboptimal, verdict = PARTIALLY_VALID. Be receptive and objective.
8. **Do not form a verdict before completing ALL reading steps above.** Every numbered step must be completed — with actual Read tool calls, not assumptions — before you write your verdict.

### Output

Return a structured verdict:

**File:** <path>
**Lines:** <range>
**Thread ID:** <id>
**Verdict:** VALID | INVALID | PARTIALLY_VALID
**Confidence:** HIGH | MEDIUM | LOW
**Reasoning:** <detailed explanation citing specific artifacts>
**Artifacts Cited:**
- <artifact path or reference>
**Suggested Fix:** <if VALID or PARTIALLY_VALID, describe what should change>
**Reply Draft:** <draft reply to post in the thread>
""")
```

### Research Rules

- **Interpret thematically.** Ask "what design principle is the reviewer pointing at?" — if the thematic concern is valid, reflect it in the verdict even if the literal suggestion isn't the right fix.
- **Trace the full provenance chain.** Reconstruct why the code exists: what requirement drove it, what decision chose this approach, what trade-offs were considered. If the chain is broken (code exists without documented rationale), that itself is a finding worth noting.
- **Read before judging.** Do not form a verdict before completing ALL research steps above. Every numbered step must be completed — with actual Read tool calls, not assumptions — before you write your verdict. Premature verdicts cause confirmation bias — you'll unconsciously seek evidence that supports your initial impression and ignore evidence that contradicts it.
- **Default to taking feedback seriously.** If a comment raises a legitimate concern, even if the current code "works," the verdict should be VALID unless artifacts explicitly justify the current approach. **Do not rationalize.** If you find yourself explaining why the code is "fine actually" or "works in practice," stop and re-examine — you may be defending the code instead of evaluating the feedback.
- **Conservative invalidation.** Only mark INVALID when you can cite a specific artifact section, standard rule, or architectural decision that **directly and specifically** contradicts the reviewer's claim. The cited artifact must address the exact concern raised — a general architectural document that "implies" the current approach is acceptable does not count. You must be able to point to a specific line or section that says the opposite of what the reviewer claims. If you cannot, the verdict is VALID or PARTIALLY_VALID.
- **PARTIALLY_VALID** when the comment identifies a real issue but proposes the wrong solution, or when only part of the feedback applies.
- **Always cite artifacts.** Every verdict must reference at least one artifact. If no artifact is relevant, note that as a provenance gap.
- **Don't use scope as invalidation reasoning.** Scope alone is NEVER enough reason to invalidate an otherwise valid claim. If a comment raises a valid concern about code changed in this PR, fix it. If the concern is valid but about code NOT changed in this PR, fix what you can and create a follow-up issue only for the truly unreachable remainder.

## Step 4: Fix (Parallel Where File-Isolated)

### Commit Strategy: One Fix, One Commit

Each fix MUST be its own atomic, revertible commit. Do NOT batch multiple fixes into a single commit.

- Each commit addresses exactly one review thread or comment
- Each commit must be independently revertible via `git revert <sha>` without breaking other fixes
- Commit message format:
  ```
  fix(review): <short description>

  Addresses <thread-id or comment-url>.
  <one-line rationale>
  ```
- If a fix touches multiple files for the same thread, all go in one commit
- If multiple threads affect the same lines in a file, each thread still gets its own commit (applied sequentially)

For threads with VALID or PARTIALLY_VALID verdicts, group fixes by file to determine what can run in parallel:

- Fixes to different files → dispatch in parallel
- Fixes to the same file → dispatch sequentially to avoid conflicts

### 4.1 Select Fix Agent

For each fix, check `.claude/agents/` for project-specific agents. Select the most applicable one based on the domain of the fix (e.g., `security-reviewer` for security fixes, `frontend` for UI fixes, `data` for model changes). Fall back to `general-purpose` if no specialist fits.

### 4.2 Dispatch Fix Agent

```
Task(subagent_type="<selected-agent>", prompt="""
## Fix Task

Apply a fix based on review feedback.

### Research Report
<full research verdict from step 3>

### Project Context
<CLAUDE.md contents>

### Relevant Artifacts
<artifacts cited in research>

### Instructions

1. Read the current state of <file> at the relevant lines
2. Implement the fix described in the research report
3. Follow all project standards from CLAUDE.md and .docs/standards/
4. **Commit this fix as its own atomic commit** — stage only the files you changed, not unrelated files:
   ```bash
   git add <only-the-files-you-changed>
   git commit -m "fix(review): <short description>

   Addresses <thread-id or comment-url>.
   <one-line rationale>"
   ```
   Do NOT leave changes uncommitted. This commit must contain only changes for this specific thread — one thread, one commit.
5. Run any relevant tests after committing:
   - Look for test files related to the changed code
   - Run the project's test command if identifiable
6. If tests fail, fix the issue and amend YOUR commit (`git commit --amend`) — do not create a separate "fix tests" commit for this thread

### Constraints
- **You MUST edit files.** If you complete this task without making a code change, you have failed. A VALID verdict means the code needs to change — your job is to change it, not to explain what should change.
- Only modify what the research report identifies — do not "improve" adjacent code
- Preserve existing code style and patterns
- If the fix requires changes beyond the identified scope, implement the core fix AND note what else needs changing
""")
```

### 4.3 Verify Fixes

After all fix agents complete:

1. **Verify atomic commits:** Run `git log --oneline` and confirm each fix produced its own commit. If any fix agent failed to commit, stage its changes and commit now with the proper format. If any agent batched multiple fixes into one commit, note this as a process failure.

2. **Run the full test suite:**
   ```bash
   # Detect and run tests — check for common patterns
   # cargo test, npm test, pytest, go test, etc.
   ```

3. If tests fail, use `git log --oneline` to identify which fix commit likely broke them. Fix the issue by amending the responsible commit or adding a targeted follow-up. Do NOT proceed to resolving threads until tests pass.

### 4.4 Save Fix Learnings

After each fix subagent completes, if the fix revealed a non-obvious pattern (e.g., a subtle dependency, a migration requirement, a test ordering issue, a surprising API behavior), save it to Vestige immediately via `remember_pattern`. Don't defer to Step 7 — fixes often surface the most actionable learnings, and delaying risks losing them.

```
mcp__vestige__codebase:
  action: "remember_pattern"
  codebase: "<project>"
  name: "<pattern name>"
  description: "<what the fix revealed and how to avoid the issue in future>"
  files: [<affected files>]
```

## Step 5: Reply + Resolve

For each researched thread, post a reply and optionally resolve:

### 5.1 Compose Reply

Based on the verdict:

**VALID (fixed):**
```
Fixed in <commit-sha>.

<reasoning summary>

Refs: <artifact citations>
```

**INVALID:**
```
After investigation, this is working as intended.

<detailed reasoning>

Refs: <artifact citations>
```

**PARTIALLY_VALID (fixed):**
```
Partially addressed in <commit-sha>.

**Fixed:** <what was fixed and why>
**Remaining:** <what was not changed, why, and the concrete next step — a follow-up commit on this PR, not "consider doing X later">

Refs: <artifact citations>
```

### 5.2 Post GitHub Replies

Post replies using GraphQL. See `references/gh-graphql.md` for the `AddPullRequestReviewThreadReply` mutation.

### 5.3 Resolve Threads

After posting the reply, resolve the thread using the `ResolveReviewThread` mutation.

**Resolution rules:**
- VALID + fix committed + tests pass → resolve
- INVALID + HIGH confidence + reply posted → resolve
- INVALID + MEDIUM or LOW confidence + reply posted → do NOT resolve, leave for human review
- PARTIALLY_VALID + fix committed + tests pass → resolve
- Any fix with failing tests → do NOT resolve, note in summary
- Research confidence LOW on any verdict → do NOT auto-resolve, note for human review

### 5.4 GitHub Issues

For issue targets, post a comment with findings and optionally close:
```bash
gh issue comment <number> --body "<findings>"
```

Only close if the issue is fully addressed. If partially addressed, note remaining items.

### 5.5 Non-Thread PR Comments

Process each non-minimized PR-level comment (these are PR body comments, not inline thread comments — they cannot be replied to with a thread reply).

For each comment, classify and act:

**Is it a summary of previously resolved threads?**
```bash
gh api graphql --input - << EOF
{"query":"mutation MinimizeComment(\$subjectId: ID!, \$classifier: ReportedContentClassifiers!) { minimizeComment(input: { subjectId: \$subjectId, classifier: \$classifier }) { minimizedComment { isMinimized minimizedReason } } }","variables":{"subjectId":"$COMMENT_ID","classifier":"RESOLVED"}}
EOF
```

**Is it outdated and not actionable?** (references code that no longer exists, mentions a stale concern already addressed)
```bash
gh api graphql --input - << EOF
{"query":"mutation MinimizeComment(\$subjectId: ID!, \$classifier: ReportedContentClassifiers!) { minimizeComment(input: { subjectId: \$subjectId, classifier: \$classifier }) { minimizedComment { isMinimized minimizedReason } } }","variables":{"subjectId":"$COMMENT_ID","classifier":"OUTDATED"}}
EOF
```

**Is it genuinely out of scope?** (STRICT definition: the comment is about functionality in files NOT changed in this PR AND not directly related to the PR's changes. If the PR touched the code being discussed, it is in scope — period. If the comment raises a valid concern about changed code but the fix seems large, it is still in scope — fix it or fix the portion you can.)
```bash
gh api graphql --input - << EOF
{"query":"mutation MinimizeComment(\$subjectId: ID!, \$classifier: ReportedContentClassifiers!) { minimizeComment(input: { subjectId: \$subjectId, classifier: \$classifier }) { minimizedComment { isMinimized minimizedReason } } }","variables":{"subjectId":"$COMMENT_ID","classifier":"OFF_TOPIC"}}
EOF
```
Then create a GitHub issue:
```bash
gh issue create --title "<specific title>" --body "<body>"
```
The issue body must include:
- Link to the original comment URL
- Link to the PR
- What specific code/area it concerns (with file paths and line references)
- Acceptance criteria for addressing it
- References to any relevant artifacts (specs, ADRs, standards)

**Otherwise (actionable, in-scope feedback):**
1. Research and resolve as with threads (steps 3–4)
2. Post a new PR comment with the response. Start the body with a link to the original:
   ```
   > [Original comment](<original-comment-url>)

   <response body>
   ```
3. Minimize the original comment:
   ```bash
   gh api graphql --input - << EOF
   {"query":"mutation MinimizeComment(\$subjectId: ID!, \$classifier: ReportedContentClassifiers!) { minimizeComment(input: { subjectId: \$subjectId, classifier: \$classifier }) { minimizedComment { isMinimized minimizedReason } } }","variables":{"subjectId":"$COMMENT_ID","classifier":"RESOLVED"}}
   EOF
   ```

### 5.6 Reviews

For each `CHANGES_REQUESTED` or `COMMENTED` review:

1. Dismiss the review as stale:
   - Message: `"Addressed — see inline thread replies and comment responses."`
   - Use `DismissReview` mutation from `references/gh-graphql.md`

2. Request re-review from the same reviewer:
   - Look up the reviewer's node ID using the `GetUserId` query
   - Use the `RequestReviews` mutation

### 5.7 Linear Tickets

For Linear targets, post a comment and update status via the Linear MCP or CLI.

## Step 6: Summary

After all items are processed, output a summary:

```
## Resolve Summary: PR #<number>

### Threads
**Processed:** <total> | **Fixed:** <count> | **Invalidated:** <count> | **Partial:** <count> | **Skipped:** <count>

### Comments
**Processed:** <total> | **Minimized (resolved):** <count> | **Minimized (outdated):** <count> | **Minimized (off-topic):** <count> | **Responded + minimized:** <count>

### Reviews
**Dismissed:** <count> | **Re-review requested:** <count>

---

### Fixes (each is its own atomic commit)
- `<sha>` <file>:<line> — <description>

### Invalidations
- <file>:<line> — <reasoning summary>

### Issues Created
- <issue-url> — <title>

### Requires Human Review
- <file>:<line> — <why this needs human judgment>

### Test Results
<pass/fail summary>
```

If any threads remain unresolved, explain why and suggest next steps.

## Step 7: Memory

Save resolution learnings to vestige:

```
mcp__vestige__smart_ingest:
  content: "Resolved <N> threads on PR #<number> (<title>). Fixed: <summary>. Invalidated: <summary>. Patterns: <any recurring feedback patterns>."
  tags: ["resolve", "<project>"]
  node_type: "event"
```

If resolution revealed common feedback patterns (e.g., "reviewers keep flagging missing error handling in this module"), save as a pattern:

```
mcp__vestige__codebase:
  action: "remember_pattern"
  codebase: "<project>"
  name: "<pattern name>"
  description: "<what keeps happening and how to prevent it>"
  files: [<affected files>]
```
