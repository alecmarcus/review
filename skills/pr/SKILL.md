---
name: pr
description: "This skill should be used when the user asks to \"review this PR\", \"review PR\", \"start a review\", \"code review\", or provides a PR reference for review."
argument-hint: "[<url> | <number> | <branch> | current | <pr-name>]"
context: fork
disable-model-invocation: false
allowed-tools: Bash, Read, Glob, Grep, Task, WebFetch, mcp__vestige__search, mcp__vestige__smart_ingest, mcp__vestige__codebase
---

# /review:pr

Comprehensive code review with provenance tracing. Gathers all genesis context (specs, ADRs, PRDs, loom logs, stories), dispatches the project's preferred review agents in parallel, and produces a thorough review.

## Prime Directive: Thoroughness Over Speed

**Every claim, finding, and verdict MUST be backed by line-by-line reading of the actual code and full reading of every referenced artifact.** No skimming. No assuming. No grepping for a term and claiming to have done research. No summarizing an artifact from its title.

- **Code**: Use the Read tool on every modified file — the entire file, not just changed lines. Read every line of the diff. Read surrounding context (imports, callers, tests, adjacent functions). If you haven't Read it, you can't make claims about it.
- **Artifacts**: Read specs, ADRs, PRDs, stories, acceptance criteria, GitHub issues, and GitHub comments **in full**. Every section, not just the one that seems relevant. Adjacent sections often contain the constraint that matters most.
- **Commits**: Read the actual diff of every commit referenced via `git show <sha>`. Do not cite a commit hash without having read the changes it introduced line by line.
- **No shortcuts**: A `Grep` hit is a starting point, not research. After finding a match, Read the full file and understand the context. A function name match does not mean you understand what the function does, what calls it, or what it depends on.

Agents that skip reading steps or produce findings based on assumptions will generate false positives and miss real issues. **Read first, judge second, always.**

## Step 1: Parse Target

Resolve `$ARGUMENTS` into a PR reference:

| Input | Resolution |
|-------|-----------|
| Empty or `current` | Infer from current branch: `gh pr view --json number,url --jq '.number'` |
| Bare number (`42`) | PR #42 in current repo |
| URL (`https://github.com/.../pull/42`) | Extract owner/repo/number from URL |
| Branch name (`feat/auth`) | Find PR for that branch: `gh pr list --head <branch> --json number --jq '.[0].number'` |
| Text (PR title search) | `gh pr list --search "<text>" --json number,title --jq '.[0].number'` |

If resolution fails, report the error and stop. Do not guess.

Store the resolved PR number as `$PR_NUMBER` and repo as `$REPO` (default: current repo) for all subsequent steps.

## Step 2: Gather Genesis Context

Before reviewing any code, collect all provenance and context that informed the PR. This is critical — reviews without genesis context produce shallow findings.

### 2.1 PR Metadata

```bash
gh pr view $PR_NUMBER --json title,body,labels,milestone,assignees,baseRefName,headRefName,commits,files,additions,deletions,changedFiles,reviews,comments,reviewRequests
```

Extract:
- **Title and description** — the stated intent
- **Linked issues** — parse `Closes #N`, `Fixes #N`, `Resolves #N` from body
- **Labels and milestone** — categorization context
- **Base and head branches** — merge target and source

### 2.2 Diff and Commits

```bash
gh pr diff $PR_NUMBER
```

```bash
gh pr view $PR_NUMBER --json commits --jq '.commits[].messageHeadline'
```

For large PRs (>3000 diff lines), note the size — step 4 will handle batching.

Count diff lines:
```bash
gh pr diff $PR_NUMBER | wc -l
```

### 2.3 Project Artifacts

Search the codebase for project context. Not all of these will exist — collect what's available:

1. **CLAUDE.md** — read root `CLAUDE.md` for project rules, architecture, agent roster
2. **`.docs/`** — specs, ADRs, standards, architecture, planning sessions:
   ```
   .docs/specs/
   .docs/adrs/
   .docs/standards/
   .docs/architecture.md
   .docs/sketch.md
   .docs/planning-sessions/
   ```
3. **Local `.docs/`** — check if any changed files have a local `.docs/` directory nearby (feature-level documentation)
4. **PRD stories** — search for story references in commit messages and PR description:
   - Check `.loom/prd.json` or any `prd.json` found via `Glob("**/prd.json")`
   - Extract stories referenced by ID patterns like `PREFIX-NNN`
5. **Loom logs** — if the project uses Loom:
   - `.loom/history.log`
   - `.loom/logs/`
   - Worktree paths: `.claude-worktrees/*/loom/`
6. **External references** — parse PR description and commits for:
   - Linear ticket IDs (`TEAM-NNN`)
   - GitHub issue references (`#NNN`)
   - URLs to external specs or designs

### 2.4 Vestige Memory

Query vestige for project-specific context:

```
mcp__vestige__search: query="<project-name> architecture patterns decisions"
mcp__vestige__codebase: action="get_context", codebase="<project-name>"
```

### 2.5 Assemble Context Bundle

Compile all gathered context into a structured summary for review agents:

```
GENESIS CONTEXT:
- PR: #<number> "<title>"
- Intent: <description summary>
- Linked issues: <list>
- Stories: <referenced PRD stories with acceptance criteria>
- Specs: <relevant spec sections>
- ADRs: <relevant architectural decisions>
- Standards: <applicable coding standards>
- Architecture: <relevant architecture context>
- Loom: <iteration history if applicable>
```

This bundle is passed to every review agent in step 4.

## Step 3: Determine Review Agents

### 3.1 Read Default Roster

Read `CLAUDE.md` and find the "Agent Reviews" section. Parse the listed agents as the base roster. If no such section exists, use this default set:

- black-hat
- red-hat
- white-hat
- security-reviewer
- cryptographer
- bug-catcher
- chronicler
- alignment-reviewer
- api-design-reviewer
- simplifier

### 3.2 Content-Based Adjustment

Analyze the PR diff and metadata to add or remove agents:

| Condition | Action |
|-----------|--------|
| PR touches crypto, MLS, encryption, key management, signatures | Ensure `cryptographer` is included |
| PR touches UI views, components, CSS, HTML, SwiftUI, Compose | Add `frontend-reviewer` |
| PR touches public API surfaces, protocols, interfaces | Ensure `api-design-reviewer` is included |
| PR includes test files | Add `test-quality-reviewer` |
| PR adds or updates dependencies (Cargo.toml, package.json, etc.) | Add `dependency-safety-reviewer` |
| PR touches performance-sensitive paths (hot loops, DB queries, caching) | Add `performance-optimizer` |
| PR is purely documentation or config | Keep all agents — even docs/config PRs benefit from full review coverage |
| PR touches data models, persistence, migrations | Add `architecture-reviewer` |

### 3.3 Load Agent Definitions

For each agent in the final roster, check if `.claude/agents/<agent-name>.md` exists in the project. If it does, read its content to use as the agent's system prompt. If not, the agent will use its built-in definition.

Also scan `.claude/agents/` for any project-specific agents not in the default roster that may be relevant.

## Step 4: Dispatch Review Agents

### 4.1 Prepare Diff Payload

Check the diff size from step 2.2:

- **Small/medium PR** (<=3000 diff lines): Pass the full diff to each agent
- **Large PR** (>3000 diff lines): Split commits into waves of ~10 commits each. Each wave gets the same genesis context but only its subset of the diff. Agents review each wave and their findings are merged in step 5.

### 4.2 Parallel Dispatch

For each agent in the roster, dispatch a Task subagent **in parallel**:

```
Task(subagent_type="<agent-type>", prompt="""
You are reviewing PR #<number>: "<title>"

## Your Role
Review this PR from the perspective of: <agent-name>
<agent-system-prompt if available>

## Project Context
<CLAUDE.md contents>

## Genesis Context
<context bundle from step 2.5>

## Diff
<full diff or wave subset>

## Instructions
1. **Read every modified file in full using the Read tool.** Not just the changed lines — the entire file. Understand imports, exports, class structure, type definitions, adjacent functions, error handling patterns. Then re-read the diff hunks with that full context. If a file is too large, read it in sections, but cover every section. **You may not produce any finding about a file you have not fully read.**
2. **Read every genesis context artifact in full.** This means:
   - Every spec document referenced or relevant to the changed code — read the entire spec, not just the section that seems relevant. Constraints often appear in adjacent sections, preambles, or appendices.
   - Every ADR — read the full rationale, alternatives considered, and consequences sections.
   - Every PRD story and its acceptance criteria — read each criterion individually and trace it to implementing code.
   - Every referenced GitHub issue — read the issue body AND all comments. Context and clarifications often appear in comments, not the original body.
   - Every referenced GitHub PR comment or review thread — read the full thread including all replies.
   - Every commit on this PR — read the commit message AND the actual diff via `git show <sha>`. Do not cite a commit without having read its changes.
   - `CLAUDE.md` and all `.docs/` artifacts — read in full, not excerpted.
3. **No skimming, no shortcuts, no assumptions.** A `Grep` match is a starting point, not research. After finding a match, use Read to understand the full file context. A function name appearing in a search result does not mean you understand what it does, what calls it, or what it depends on. **If you haven't read it line by line, you don't know what it says.**
4. **Cite specific artifact, section, and line range** for every finding — e.g., `spec.md:45-52`, `ADR-003:rationale`, `story PROJ-42 acceptance criterion 3`. Not "see spec" or "per the ADR." Vague citations are as bad as no citations.
5. **Trace provenance for every significant diff hunk** — identify which requirement, decision, or story drove each change. Follow the chain: requirement → design decision → implementation. Flag untraceable changes (code that exists without documented rationale).
6. **Consider findings literally AND thematically** — a database call in a hot path is literally "a query" but thematically a caching/performance concern. A missing nil check is literally "no guard" but thematically a failure-mode design gap. Surface both levels.
7. **Classify findings as binary: DO or LEARN.** DO = must fix (bugs, correctness errors, spec violations, provenance gaps, unmet acceptance criteria). LEARN = worth remembering (patterns, conventions, non-blocking observations). **Bias hard toward DO. If in doubt, it's a DO.** Do not rationalize away a concern — if it looks like a problem, classify it as DO and explain why. The bar for LEARN is high: you must be confident the issue has zero correctness, security, or spec-compliance impact.
8. **Do not manufacture findings** — but also **do not manufacture excuses**. If you find yourself explaining why something "is fine actually" or "works in practice," stop and re-examine. Your job is to surface real problems, not to defend the code.
9. **Search Vestige** for patterns, known issues, and past review findings relevant to the files being reviewed: `mcp__vestige__search(query: "<project> <file-area> patterns issues review")`. Previous review cycles may have flagged the same file or pattern — build on that context instead of starting from zero.
10. **Evaluate commit quality** — review the PR's commit history (`git log --oneline` for the PR range). Flag non-atomic commits that bundle unrelated changes, oversized commits that should be split, missing or unhelpful commit messages, and commits that aren't independently revertible. Each logical change should be its own commit.

## Output Format
Return your findings as a structured list:

### Findings
- **[DO]** <file>:<line> — <description>
  - Artifact: <spec/ADR/standard reference>
  - Reasoning: <why this matters>
- **[LEARN]** <description> — <why this matters for future work>

### Summary
<1-2 sentence domain-specific summary>

### Provenance Gaps
<list of changes without documented rationale, or "None">
""")
```

Dispatch all agents in a single message with parallel Task calls.

### 4.3 Collect Results

Wait for all agents to complete. Each returns structured findings.

## Step 5: Synthesize Review

### 5.1 Aggregate Findings

Collect all agent reports and:

1. **Deduplicate** — if multiple agents flagged the same line/issue, merge into one finding. Preserve the **union of all citations** when merging — if agent A cited `spec.md:45` and agent B cited `ADR-003:rationale`, the merged finding includes both. Note which agents agreed.
2. **Group by classification** — DO items first, then LEARN items
3. **Group by file** — within each classification, organize by file path
4. **Verify provenance** — every finding should cite a specific artifact section. If an agent finding lacks a citation, search the genesis context yourself to find the supporting reference. **Default: keep as DO.** Only weaken to LEARN if no artifact reference exists anywhere AND the finding has zero correctness, security, or spec-compliance impact. If you're uncertain, it stays as DO.
5. **Diff coverage check** — verify every changed file in the PR was reviewed by at least one agent. If any file was missed, flag it as a gap and note which agents should have covered it.
6. **LEARN → DO reclassification** — review every LEARN finding and ask: "Would a senior reviewer expect this to block merge?" If yes, reclassify as DO. Common misclassifications: unmet acceptance criteria marked as LEARN, spec deviations marked as LEARN, missing error handling in user-facing paths marked as LEARN, security concerns marked as LEARN.

### 5.2 Check Implementation Coverage

Perform a methodical requirement-by-requirement trace against the PR:

1. **For each requirement** (from PRD stories, specs, acceptance criteria), locate the implementing code — cite specific `file:line-range`.
2. **Evaluate completeness** — is the requirement fully satisfied or only partially? Note partial implementations explicitly.
3. **Check for subtle drift** — does the implementation do what the requirement says, or something close-but-different? Drift is harder to catch than omission and more dangerous.
4. **Flag untraceable changes** — diff hunks that don't map to any requirement. These may be legitimate (refactoring, cleanup) or scope creep.
5. **Flag uncovered requirements** — requirements with no implementing code in the diff.

Flag untraceable changes, uncovered requirements, and subtle drift as DO findings.

### 5.2b Save Mid-Synthesis Learnings

Before formatting the final review, save notable patterns and provenance gaps to Vestige immediately — don't wait for Step 6. If the review reveals a recurring anti-pattern or a provenance gap that keeps appearing, save it now so future reviews benefit even if this review is interrupted.

```
mcp__vestige__smart_ingest:
  content: "REVIEW PATTERN: <pattern description>. Seen in PR #<number> affecting <files>."
  tags: ["review-pattern", "<project>"]
  node_type: "pattern"
```

### 5.3 Format Review

Compose the final review in this structure:

```markdown
## Review: PR #<number> — <title>

**Agents:** <list of agents that reviewed>
**Genesis:** <list of artifacts consulted>
**Verdict:** <APPROVE | REQUEST_CHANGES | COMMENT>

### Must Do (<count>)
- **<file>:<line>** — <description>
  - <agent(s)>: <reasoning>
  - Ref: <artifact citation>

### Learnings (<count>)
- <description> — <why this matters for future work>
  - <agent(s)>
  - Ref: <artifact citation>

### Implementation Coverage
- [x] Story PREFIX-001: All criteria met — `src/auth.ts:45-120`
- [ ] Story PREFIX-002: Missing criterion "..." — no implementing code found
- [~] Story PREFIX-003: Partial — `src/api.ts:30-50` covers auth but not rate limiting

### Provenance Gaps
- <file>:<line> — No documented rationale found for <change>

### Commit Quality
- <commit-sha> — <issue: non-atomic, bundles unrelated changes, missing message, not revertible, etc.>
- or "All commits are atomic and well-structured"
```

### 5.4 Post Review

Determine the review action:
- If any DO findings → `REQUEST_CHANGES`
- If only LEARN findings → `COMMENT`
- If no findings → `APPROVE`

Post via `gh`:
```bash
gh pr review $PR_NUMBER --<action> --body "<review body>"
```

If the review body exceeds GitHub's comment size limit (~65536 chars), split into a review + follow-up comments.

## Step 6: Memory

Save review learnings to vestige:

```
mcp__vestige__smart_ingest:
  content: "Reviewed PR #<number> (<title>) in <repo>. Key findings: <summary>. Agents used: <list>."
  tags: ["review", "<project>"]
  node_type: "event"
```

If the review revealed new architectural patterns or decisions, also save those:

```
mcp__vestige__codebase:
  action: "remember_decision"
  codebase: "<project>"
  decision: "<what was decided>"
  rationale: "<why>"
```
