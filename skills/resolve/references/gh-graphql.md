# GitHub GraphQL Queries — Resolve

Reference queries for fetching unresolved threads, replying, and resolving via `gh api graphql`.

> **Important:** Do NOT use `gh api graphql -f query='...'` with these queries. The `!` non-null
> markers in GraphQL types (e.g. `String!`, `Int!`) get corrupted by zsh history expansion when
> passed via `-f`. Always use `gh api graphql --input -` with a heredoc instead, escaping GraphQL
> `$variables` as `\$variables` to prevent shell expansion.

## Fetch Unresolved Review Threads

Retrieves only unresolved, non-outdated threads from a PR. Primary query for the resolve skill.

```graphql
query UnresolvedThreads($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      id
      title
      body
      baseRefName
      headRefName
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
    }
  }
}
```

### Usage

```bash
gh api graphql --input - << EOF
{"query":"query UnresolvedThreads(\$owner: String!, \$repo: String!, \$number: Int!) { repository(owner: \$owner, name: \$repo) { pullRequest(number: \$number) { id title body baseRefName headRefName reviewThreads(first: 100) { nodes { id isResolved isOutdated path line startLine diffSide comments(first: 50) { nodes { id body author { login } createdAt path line startLine } } } } } } }","variables":{"owner":"$OWNER","repo":"$REPO","number":$PR_NUMBER}}
EOF
```

### Filter Unresolved

```bash
... | jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false and .isOutdated == false)]'
```

## Reply to a Review Thread

Adds a comment reply to an existing pull request review thread.

```graphql
mutation AddPullRequestReviewThreadReply($threadId: ID!, $body: String!) {
  addPullRequestReviewThreadReply(input: {
    pullRequestReviewThreadId: $threadId
    body: $body
  }) {
    comment {
      id
      body
      createdAt
    }
  }
}
```

### Usage

```bash
gh api graphql --input - << EOF
{"query":"mutation AddPullRequestReviewThreadReply(\$threadId: ID!, \$body: String!) { addPullRequestReviewThreadReply(input: { pullRequestReviewThreadId: \$threadId, body: \$body }) { comment { id body createdAt } } }","variables":{"threadId":"$THREAD_ID","body":"$REPLY_BODY"}}
EOF
```

## Resolve a Review Thread

Marks a pull request review thread as resolved.

```graphql
mutation ResolveReviewThread($threadId: ID!) {
  resolveReviewThread(input: {
    threadId: $threadId
  }) {
    thread {
      id
      isResolved
    }
  }
}
```

### Usage

```bash
gh api graphql --input - << EOF
{"query":"mutation ResolveReviewThread(\$threadId: ID!) { resolveReviewThread(input: { threadId: \$threadId }) { thread { id isResolved } } }","variables":{"threadId":"$THREAD_ID"}}
EOF
```

## Unresolve a Review Thread

Re-opens a previously resolved thread (use if a fix attempt fails).

```graphql
mutation UnresolveReviewThread($threadId: ID!) {
  unresolveReviewThread(input: {
    threadId: $threadId
  }) {
    thread {
      id
      isResolved
    }
  }
}
```

### Usage

```bash
gh api graphql --input - << EOF
{"query":"mutation UnresolveReviewThread(\$threadId: ID!) { unresolveReviewThread(input: { threadId: \$threadId }) { thread { id isResolved } } }","variables":{"threadId":"$THREAD_ID"}}
EOF
```

## Submit a PR Review

Posts a formal PR review (used when resolve needs to submit a review-level comment).

```graphql
mutation SubmitReview($prId: ID!, $body: String!, $event: PullRequestReviewEvent!) {
  addPullRequestReview(input: {
    pullRequestId: $prId
    body: $body
    event: $event
  }) {
    pullRequestReview {
      id
      state
    }
  }
}
```

Events: `APPROVE`, `REQUEST_CHANGES`, `COMMENT`

### Usage

```bash
gh api graphql --input - << EOF
{"query":"mutation SubmitReview(\$prId: ID!, \$body: String!, \$event: PullRequestReviewEvent!) { addPullRequestReview(input: { pullRequestId: \$prId, body: \$body, event: \$event }) { pullRequestReview { id state } } }","variables":{"prId":"$PR_NODE_ID","body":"$REVIEW_BODY","event":"COMMENT"}}
EOF
```
