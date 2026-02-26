# GitHub GraphQL Queries — Resolve

Reference queries for fetching unresolved threads, replying, and resolving via `gh api graphql`.

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
gh api graphql -f query='...' -f owner="$OWNER" -f repo="$REPO" -F number=$PR_NUMBER
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
gh api graphql -f query='mutation AddPullRequestReviewThreadReply($threadId: ID!, $body: String!) { addPullRequestReviewThreadReply(input: { pullRequestReviewThreadId: $threadId, body: $body }) { comment { id body createdAt } } }' -f threadId="$THREAD_ID" -f body="$REPLY_BODY"
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
gh api graphql -f query='mutation ResolveReviewThread($threadId: ID!) { resolveReviewThread(input: { threadId: $threadId }) { thread { id isResolved } } }' -f threadId="$THREAD_ID"
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
gh api graphql -f query='mutation UnresolveReviewThread($threadId: ID!) { unresolveReviewThread(input: { threadId: $threadId }) { thread { id isResolved } } }' -f threadId="$THREAD_ID"
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
gh api graphql -f query='mutation SubmitReview($prId: ID!, $body: String!, $event: PullRequestReviewEvent!) { addPullRequestReview(input: { pullRequestId: $prId, body: $body, event: $event }) { pullRequestReview { id state } } }' -f prId="$PR_NODE_ID" -f body="$REVIEW_BODY" -f event="COMMENT"
```
