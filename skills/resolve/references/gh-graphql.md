# GitHub GraphQL Queries — Resolve

Reference queries for fetching unresolved threads, replying, and resolving via `gh api graphql`.

> **Important:** Do NOT use `gh api graphql -f query='...'` with these queries. The `!` non-null
> markers in GraphQL types (e.g. `String!`, `Int!`) get corrupted by zsh history expansion when
> passed via `-f`. Always use `gh api graphql --input -` with a heredoc instead, escaping GraphQL
> `$variables` as `\$variables` to prevent shell expansion.

## Fetch PR Data (Threads, Comments, Reviews)

Retrieves review threads, PR-level comments, and formal reviews in one request.

```graphql
query PRFetch($owner: String!, $repo: String!, $number: Int!) {
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
      comments(first: 100) {
        nodes {
          id
          url
          body
          author { login }
          createdAt
          isMinimized
          minimizedReason
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
    }
  }
}
```

### Usage

```bash
gh api graphql --input - << EOF
{"query":"query PRFetch(\$owner: String!, \$repo: String!, \$number: Int!) { repository(owner: \$owner, name: \$repo) { pullRequest(number: \$number) { id title body baseRefName headRefName reviewThreads(first: 100) { nodes { id isResolved isOutdated path line startLine diffSide comments(first: 50) { nodes { id body author { login } createdAt path line startLine } } } } comments(first: 100) { nodes { id url body author { login } createdAt isMinimized minimizedReason } } reviews(first: 50) { nodes { id state body author { login } submittedAt } } } } }","variables":{"owner":"$OWNER","repo":"$REPO","number":$PR_NUMBER}}
EOF
```

### Filter Unresolved Threads

```bash
... | jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false and .isOutdated == false)]'
```

### Filter Active Comments

```bash
... | jq '[.data.repository.pullRequest.comments.nodes[] | select(.isMinimized == false)]'
```

### Filter Active Reviews

```bash
... | jq '[.data.repository.pullRequest.reviews.nodes[] | select(.state == "CHANGES_REQUESTED" or .state == "COMMENTED")]'
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

## Minimize (Hide) a Comment

Hides a PR-level comment. Use for comments that are resolved, outdated, or off-topic.

`classifier` values: `RESOLVED`, `OUTDATED`, `OFF_TOPIC`, `SPAM`, `ABUSE`, `DUPLICATE`

The `subjectId` is the node `id` of the comment from the fetch query.

```graphql
mutation MinimizeComment($subjectId: ID!, $classifier: ReportedContentClassifiers!) {
  minimizeComment(input: {
    subjectId: $subjectId
    classifier: $classifier
  }) {
    minimizedComment {
      isMinimized
      minimizedReason
    }
  }
}
```

### Usage

```bash
gh api graphql --input - << EOF
{"query":"mutation MinimizeComment(\$subjectId: ID!, \$classifier: ReportedContentClassifiers!) { minimizeComment(input: { subjectId: \$subjectId, classifier: \$classifier }) { minimizedComment { isMinimized minimizedReason } } }","variables":{"subjectId":"$COMMENT_ID","classifier":"$CLASSIFIER"}}
EOF
```

## Dismiss a PR Review

Dismisses a formal PR review (REQUEST_CHANGES or COMMENTED) as stale.

```graphql
mutation DismissReview($reviewId: ID!, $message: String!) {
  dismissPullRequestReview(input: {
    pullRequestReviewId: $reviewId
    message: $message
  }) {
    pullRequestReview {
      id
      state
    }
  }
}
```

### Usage

```bash
gh api graphql --input - << EOF
{"query":"mutation DismissReview(\$reviewId: ID!, \$message: String!) { dismissPullRequestReview(input: { pullRequestReviewId: \$reviewId, message: \$message }) { pullRequestReview { id state } } }","variables":{"reviewId":"$REVIEW_ID","message":"$DISMISS_MESSAGE"}}
EOF
```

## Request Re-Review

Requests re-review from a specific user. Requires the reviewer's node ID, not their login.
Use the helper query below to look up the ID from a login first.

```graphql
query GetUserId($login: String!) {
  user(login: $login) {
    id
  }
}
```

```bash
USER_ID=$(gh api graphql --input - << EOF
{"query":"query GetUserId(\$login: String!) { user(login: \$login) { id } }","variables":{"login":"$REVIEWER_LOGIN"}}
EOF
jq -r '.data.user.id')
```

```graphql
mutation RequestReviews($pullRequestId: ID!, $userIds: [ID!]!) {
  requestReviews(input: {
    pullRequestId: $pullRequestId
    userIds: $userIds
    union: true
  }) {
    pullRequest {
      id
    }
  }
}
```

### Usage

```bash
gh api graphql --input - << EOF
{"query":"mutation RequestReviews(\$pullRequestId: ID!, \$userIds: [ID!]!) { requestReviews(input: { pullRequestId: \$pullRequestId, userIds: \$userIds, union: true }) { pullRequest { id } } }","variables":{"pullRequestId":"$PR_NODE_ID","userIds":["$USER_ID"]}}
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
