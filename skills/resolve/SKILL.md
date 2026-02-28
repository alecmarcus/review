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

1. Read the file at <path> around lines <start>-<end> with surrounding context
2. Read the diff for this file from the PR
3. Search for relevant artifacts:
   - `.docs/specs/` — specifications that govern this code
   - `.docs/adrs/` — architectural decisions that apply
   - `.docs/standards/` — coding standards that apply
   - `.docs/lessons/` — known pitfalls or patterns
   - `CLAUDE.md` — project rules
   - Commit messages on this PR for rationale
   - Any local `.docs/` near the changed files
4. Search all available memories and memory tools for relevant context
5. Evaluate the comment's claim against the code AND the artifacts. Be receptive and objective.

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

- **Default to taking feedback seriously.** If a comment raises a legitimate concern, even if the current code "works," the verdict should be VALID unless artifacts explicitly justify the current approach.
- **Conservative invalidation.** Only mark INVALID when you can cite a specific artifact, standard, or architectural decision that directly contradicts the comment's claim.
- **PARTIALLY_VALID** when the comment identifies a real issue but proposes the wrong solution, or when only part of the feedback applies.
- **Always cite artifacts.** Every verdict must reference at least one artifact. If no artifact is relevant, note that as a provenance gap.
- **Don't use scope as invalidation reasoning.** Scope alone is NEVER enough reason to invalidate an otherwise valid claim. You should accept valid claims and create plans to address them, even if they seem like scope-creep. At most, you may create a follow up story or issue. 

## Step 4: Fix (Parallel Where File-Isolated)

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
4. Run any relevant tests after the fix:
   - Look for test files related to the changed code
   - Run the project's test command if identifiable
5. If tests fail, diagnose and fix — do not leave failing tests

### Constraints
- Only modify what the research report identifies — do not "improve" adjacent code
- Preserve existing code style and patterns
- If the fix requires changes beyond the identified scope, note what else needs changing but only fix what was identified
""")
```

### 4.3 Verify Fixes

After all fix agents complete, run the project's test suite if identifiable:
```bash
# Detect and run tests — check for common patterns
# cargo test, npm test, pytest, go test, etc.
```

If tests fail, this becomes a BLOCKING issue — do not proceed to resolving threads until tests pass.

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

<what was fixed and why>
<what was not changed and why>

Refs: <artifact citations>
```

### 5.2 Post GitHub Replies

Post replies using GraphQL. See `references/gh-graphql.md` for the `AddPullRequestReviewThreadReply` mutation.

### 5.3 Resolve Threads

After posting the reply, resolve the thread using the `ResolveReviewThread` mutation.

**Resolution rules:**
- VALID + fix committed + tests pass → resolve
- INVALID + reply posted → resolve
- PARTIALLY_VALID + fix committed + tests pass → resolve
- Any fix with failing tests → do NOT resolve, note in summary
- Research confidence LOW → do NOT auto-resolve, note for human review

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

**Is it out of scope?**
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

### Fixes
- <file>:<line> — <description> (commit <sha>)

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
