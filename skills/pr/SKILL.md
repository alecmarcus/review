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
| PR is purely documentation or config | Remove combat agents (black-hat, red-hat, white-hat) |
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
1. Review the diff thoroughly from your domain perspective
2. For every finding, cite the relevant artifact (spec, ADR, standard, or code reference)
3. Flag any provenance gaps — changes that don't trace to a documented decision
4. Categorize each finding as: BLOCKING, SUGGESTION, or NIT
5. If you find no issues in your domain, say so explicitly — don't manufacture findings

## Output Format
Return your findings as a structured list:

### Findings
- **[BLOCKING|SUGGESTION|NIT]** <file>:<line> — <description>
  - Artifact: <spec/ADR/standard reference>
  - Reasoning: <why this matters>

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

1. **Deduplicate** — if multiple agents flagged the same line/issue, merge into one finding and note which agents agreed
2. **Group by severity** — BLOCKING first, then SUGGESTION, then NIT
3. **Group by file** — within each severity, organize by file path
4. **Verify provenance** — every finding should cite an artifact. If an agent finding lacks a citation, either add one or demote to NIT

### 5.2 Check Implementation Coverage

Compare the PR against referenced PRD stories and specs:
- Are all acceptance criteria addressed?
- Are there spec requirements not covered by the diff?
- Are there changes that go beyond the stated scope?

Flag any gaps as BLOCKING findings.

### 5.3 Format Review

Compose the final review in this structure:

```markdown
## Review: PR #<number> — <title>

**Agents:** <list of agents that reviewed>
**Genesis:** <list of artifacts consulted>
**Verdict:** <APPROVE | REQUEST_CHANGES | COMMENT>

### Blocking (<count>)
- **<file>:<line>** — <description>
  - <agent(s)>: <reasoning>
  - Ref: <artifact citation>

### Suggestions (<count>)
- **<file>:<line>** — <description>
  - <agent(s)>: <reasoning>
  - Ref: <artifact citation>

### Nits (<count>)
- **<file>:<line>** — <description>

### Implementation Coverage
- [x] Story PREFIX-001: All criteria met
- [ ] Story PREFIX-002: Missing criterion "..."
- [x] ADR-015: Compliant

### Provenance Gaps
- <file>:<line> — No documented rationale found for <change>
```

### 5.4 Post Review

Determine the review action:
- If any BLOCKING findings → `REQUEST_CHANGES`
- If only SUGGESTION/NIT findings → `COMMENT`
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
