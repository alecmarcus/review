# GitHub GraphQL Queries — PR Review

Reference queries for fetching PR data via `gh api graphql`.

## Fetch PR with Review Threads

Retrieves PR metadata and all review threads with comments. Used in step 2 to gather review context.

```graphql
query PRWithThreads($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      id
      title
      body
      state
      baseRefName
      headRefName
      additions
      deletions
      changedFiles
      commits(last: 100) {
        nodes {
          commit {
            oid
            messageHeadline
            messageBody
            author {
              name
              user { login }
            }
          }
        }
      }
      files(first: 100) {
        nodes {
          path
          additions
          deletions
          changeType
        }
      }
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          startLine
          diffSide
          comments(first: 50) {
            nodes {
              id
              body
              author { login }
              createdAt
              path
              line
              startLine
            }
          }
        }
      }
      reviews(first: 50) {
        nodes {
          id
          state
          body
          author { login }
          submittedAt
        }
      }
      closingIssuesReferences(first: 20) {
        nodes {
          number
          title
          body
        }
      }
    }
  }
}
```

### Usage

```bash
gh api graphql -f query='...' -f owner="$OWNER" -f repo="$REPO" -F number=$PR_NUMBER
```

Parse with `jq`:
```bash
# Extract review thread summaries
... | jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false) | {id, path, line, comment: .comments.nodes[0].body}'
```

## Fetch PR Changed Files (Paginated)

For large PRs with >100 changed files:

```graphql
query PRFiles($owner: String!, $repo: String!, $number: Int!, $cursor: String) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      files(first: 100, after: $cursor) {
        pageInfo {
          hasNextPage
          endCursor
        }
        nodes {
          path
          additions
          deletions
          changeType
        }
      }
    }
  }
}
```

### Usage

```bash
# First page
gh api graphql -f query='...' -f owner="$OWNER" -f repo="$REPO" -F number=$PR_NUMBER

# Subsequent pages
gh api graphql -f query='...' -f owner="$OWNER" -f repo="$REPO" -F number=$PR_NUMBER -f cursor="$END_CURSOR"
```
