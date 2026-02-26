# review

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin for PR review and comment resolution.

## Skills

| Skill | Description |
|-------|-------------|
| `/review:pr` | Comprehensive code review — gathers genesis context (specs, ADRs, PRDs, loom logs), dispatches review agents in parallel, and produces a provenance-traced review. |
| `/review:resolve` | Resolve PR comments, GitHub issues, or Linear tickets — researches each claim, validates or fixes, replies with reasoning and artifact citations, resolves threads. |

## Install

### From marketplace

```
/plugin marketplace add alecmarcus/claude-plugins
/plugin install review@alec-plugins
```

### From GitHub

```
/plugin install alecmarcus/review
```

### Local

```
claude --plugin-dir /path/to/review
```

## Usage

### PR Review

```
/review:pr              # Review current branch's PR
/review:pr 42           # Review PR #42
/review:pr feat/auth    # Review PR for branch
/review:pr current      # Explicit current branch
```

### Resolve

```
/review:resolve              # Resolve threads on current PR
/review:resolve 42           # Resolve threads on PR #42
/review:resolve issue 15     # Resolve GitHub issue #15
/review:resolve TEAM-123     # Resolve Linear ticket
/review:resolve current      # Explicit current branch
```

## How It Works

### PR Review

1. Parses the target into a PR reference
2. Gathers all genesis context: specs, ADRs, standards, PRD stories, loom logs, commit history
3. Reads the project's CLAUDE.md for the preferred review agent roster
4. Dispatches all review agents in parallel, each with full context
5. Synthesizes findings into a unified review grouped by severity
6. Posts the review to GitHub

### Resolve

1. Parses target(s) into PR threads, GitHub issues, or Linear tickets
2. Fetches unresolved items via GraphQL
3. Researches each item in parallel — traces claims to artifacts
4. Fixes valid issues (parallel where file-isolated), runs tests
5. Replies to each thread with reasoning and artifact citations
6. Resolves threads after verification

## License

MIT
